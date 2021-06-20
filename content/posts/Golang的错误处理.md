---
title: Golang 的错误处理
date: "2021-05-20"
tags: ["Golang"]
---

Go 的错误处理一直是初学 Go 开发者不断吐槽的一个点。Go 没有像一般语言那样提供`try catch`的处理方式，而是通过函数返回值的方式直接返回。
需要不断的进行判断
``` go
if err != nil {
    return err
}
```
Rob Pike 也做了[说明](https://blog.golang.org/errors-are-values)

> Values can be programmed, and since errors are values, errors can be programmed.

Rob Pike 在这篇文章里也展示了如果优雅的 handle error。

## 标准的Error interface

Go j的标准包提供了一个 error interface

``` go
type error interface {
	Error() string
}
```

创建 error

``` go
// Example 1
func demo() (bool, error) {
	return false, errors.New("A new error")
}

// Example 2
fmt.Errorf("A new error")
```

实现一个自定义的 error， 只要实现 error interface 的 `Error` 方法即可

``` go
// errorString is a trivial implementation of error.
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

举个实际的例子

``` go
func main() {
	// 假设 readFile 存在于第三方或公用的库，我们没有权限修改、或者修改它的影响面很大
	_, err := readFile("test")

	// 错误中包含业务逻辑:
	// 1. 文件不存在时，认为是 正常
	// 2. 其余报错时，认为是 异常
	if err != nil {
		if strings.Index(err.Error(), "no such file or directory") >= 0 {
			log.Println("file not exist")
			os.Exit(0)
		}
		log.Println("open file error")
		os.Exit(1)
	}
}

func readFile(fileName string) ([]byte, error) {
	b, err := ioutil.ReadFile(fileName)
	if err != nil {
    // 为了排查问题 往往在返回错误的同时打印出来
    // 这样会导致一个错误多处打印
		log.Printf("read file %s error %v", fileName, err)

		return nil, fmt.Errorf("read file %s error %v", fileName, err)
	}
	return b, nil
}
```

这里存在3个明显的问题：

- 破坏性 - fmt.Errorf 破坏了原有的error，将它从一个**具体对象**转化为**扁平**的 string，再填充到了新的error中。所以，通过fmt.Errorf处理后的error，都只传递了一个string的信息
- 实现僵化 - “no such file or directory” 这个错误信息用的是**硬编码**，对第三方readFile的内容有强依赖，不灵活
- 排查问题效率低 - 可以通过日志组件了解到error在main函数哪行发生，但无法知道错误从readFile中的哪行返回过来的

这篇[文章](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)也说明了现有的错误处理方式的不足

## pkg/errors

根据Go2的 proposal,未来Go的Error handle方式参考了流行的 [github.com/pkg/errors](github.com/pkg/errors)

这个包主要有2个方法

``` go
// Wrap annotates cause with a message.
func Wrap(cause error, message string) error

// Cause unwraps an annotated error.
func Cause(err error) error
```

``` go
func ReadFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, errors.Wrap(err, "open failed")
	}
	defer f.Close()

	buf, err := ioutil.ReadAll(f)
	if err != nil {
		return nil, errors.Wrap(err, "read failed")
	}
	return buf, nil
}
func ReadConfig() ([]byte, error) {
  home := os.Getenv("HOME")
  config, err := ReadFile(filepath.Join(home, ".settings.xml"))
  return config, errors.Wrap(err, "could not read config")
}
 
func main() {
  _, err := ReadConfig()
  if err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}
```
如果配置文件不存在，会返回

```
could not read config: open failed: open .settings.xml: no such file or directory
```

也可以使用`%+v`打印额外的信息
``` go
func main() {
  _, err := ReadConfig()
  if err != nil {
	  fmt.Printf("%+v\n", err)
    os.Exit(1)
  }
}
```

```
readfile.go:27: could not read config
readfile.go:14: open failed
open /Users/dfc/.settings.xml: no such file or directory
```

Unwrapping
``` go
// IsTemporary returns true if err is temporary.
func IsTemporary(err error) bool {
	te, ok := errors.Cause(err).(temporary)
	return ok && te.Temporary()
}
```

## golang.org/x/xerrors

在Go 1.13中 errors 加入了 golang.org/x/xerrors 的部分功能, 但是它现在并不支持调用栈信息的输出

主要的调用方法都大同小异。

## 实际应用

### 全局定义的error实现 - MyError
``` go
// 全局的 错误号 类型，用于API调用之间传递
type MyErrorCode int

// 全局的 错误号 的具体定义
const (
	ErrorBookNotFoundCode MyErrorCode = iota + 1
	ErrorBookHasBeenBorrowedCode
)

// 内部的错误map，用来对应 错误号和错误信息
var errCodeMap = map[MyErrorCode]string{
	ErrorBookNotFoundCode:        "Book was not found",
	ErrorBookHasBeenBorrowedCode: "Book has been borrowed",
}

// Sentinel Error： 即全局定义的Static错误变量
// 注意，这里的全局error是没有保存堆栈信息的，所以需要在初始调用处使用 errors.Wrap
var (
	ErrorBookNotFound        = NewMyError(ErrorBookNotFoundCode)
	ErrorBookHasBeenBorrowed = NewMyError(ErrorBookHasBeenBorrowedCode)
)

func NewMyError(code MyErrorCode) *MyError {
	return &MyError{
		Code:    code,
		Message: errCodeMap[code],
	}
}

// error的具体实现
type MyError struct {
	// 对外使用 - 错误码
	Code MyErrorCode
	// 对外使用 - 错误信息
	Message string
}

func (e *MyError) Error() string {
	return e.Message
}
```

### 具体场景

```go
func main() {
	books := []string{
		"Hamlet",
		"Jane Eyre",
		"War and Peace",
	}

	for _, bookName := range books {
		fmt.Printf("%s start\n===\n", bookName)

		err := borrowOne(bookName)
		if err != nil {
			fmt.Printf("%+v\n", err)
		}

		fmt.Printf("===\n%s end\n\n", bookName)
	}
}

func borrowOne(bookName string) error {
	// Step1: 找书
	err := searchBook(bookName)

	// Step2: 处理
	// 特殊业务场景：如果发现书被借走了，下次再来就行了，不需要作为错误处理
	if err != nil {
		// 提取error这个interface底层的错误码，一般在API的返回前才提取
		// As - 获取错误的具体实现
		var myError = new(MyError)
		if errors.As(err, &myError) {
			fmt.Printf("error code is %d, message is %s\n", myError.Code, myError.Message)
		}

		// 特殊逻辑: 对应场景2，指定错误(ErrorBookHasBeenBorrowed)时，打印即可，不返回错误
		// Is - 判断错误是否为指定类型
		if errors.Is(err, ErrorBookHasBeenBorrowed) {
			fmt.Printf("book %s has been borrowed, I will come back later!\n", bookName)
			err = nil
		}
	}

	return err
}

func searchBook(bookName string) error {
	// 下面两个 error 都是不带堆栈信息的，所以初次调用得用Wrap方法
	// 如果已有堆栈信息，应调用WithMessage方法

	// 3 发现图书馆不存在这本书 - 认为是错误，需要打印详细的错误信息
	if len(bookName) > 10 {
		return errors.Wrapf(ErrorBookNotFound, "bookName is %s", bookName)
	} else if len(bookName) > 8 {
		// 2 发现书被借走了 - 打印一下被接走的提示即可，不认为是错误
		return errors.Wrapf(ErrorBookHasBeenBorrowed, "bookName is %s", bookName)
	}
	// 1 找到书 - 不需要任何处理
	return nil
}
```

1. MyError 作为全局 error 的底层实现，保存具体的错误码和错误信息；
2. MyError向上返回错误时，第一次先用Wrap初始化堆栈，后续用WithMessage增加堆栈信息；
3. 从error中解析具体错误时，用errors.As提取出MyError，其中的错误码和错误信息可以传入到具体的API接口中；
4. 要判断error是否为指定的错误时，用errors.Is + Sentinel Error的方法，处理一些特定情况下的逻辑；


> Tips
> 1. 不要一直用errors.Wrap来反复包装错误，堆栈信息会爆炸，具体情况可自行测试了解
> 2. 利用go generate可以大量简化初始化Sentinel Error这块重复的工作
> 3. github.com/pkg/errors 和标准库的error完全兼容，可以先替换、后续改造历史遗留的代码
> 4. 一定要注意打印error的堆栈需要用%+v，而原来的%v依旧为普通字符串方法；同时也要注意日志采集工具是否支持多行匹配

## 参考
[Why does Go not have exceptions?](https://golang.org/doc/faq#exceptions)

[Errors are values](https://blog.golang.org/errors-are-values)

[Proposal: Go 2 Error Inspection](https://go.googlesource.com/proposal/+/master/design/29934-error-values.md)

[Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

[Go Error的工程化探索](http://junes.tech/2021/05/05/go-tip/go-tip-2/)
