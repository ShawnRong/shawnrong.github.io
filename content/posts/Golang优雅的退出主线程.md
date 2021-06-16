---
title: Golang优雅的退出主线程
date: "2021-05-15"
tags: ["Golang"]
---

当我们想终止一个运行中golang程序, 往往会使用 `ctrl+c` 或者 `kill -9 <pid>` 来杀死程序。当我们正在运行一些原子性操作的代码的时候(比如写文件)， 这样操作的话可能会导致问题产生。 因此需要有一个优雅的处理方式，等原子性的操作代码处理完后，再终止程序。

可以使用 golang的 `os.Signal` 来捕获系统的终止操作

``` go
	sig := make(chan os.Signal)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGKILL, syscall.SIGTERM)
```

[POSIX中定义的信号](https://man7.org/linux/man-pages/man7/signal.7.html)

## 使用2个channel通信的方式

``` go
func main() {
	sig := make(chan os.Signal)
	stopCh := make(chan struct{})
	finishedCh := make(chan struct{})
	signal.Notify(sig, syscall.SIGINT, syscall.SIGKILL)

	go func(stopCh, finishedCh chan struct{}) {
		for {
			select {
			case <-stopCh:
				fmt.Println("stopped")
				finishedCh <- struct{}{}
				return
			default:
				time.Sleep(time.Second * 10)
			}
		}
	}(stopCh, finishedCh)

  //程序被挂起 等待singal
	<-sig
	stopCh <- struct{}{}
  //等待子routine 返回
	<-finishedCh
	fmt.Println("finished")
}

```

- 父goroutine通知子goroutine准备优雅地关闭，也就是stopCh
- 子goroutine通知父goroutine已经关闭完成，也就是finishedCh

## 嵌套 channel

``` go
func main() {
	sig := make(chan os.Signal)
	stopCh := make(chan chan struct{})
	signal.Notify(sig, syscall.SIGINT, syscall.SIGKILL)

	go func(stopChh chan chan struct{}) {
		for {
			select {
			case ch := <-stopCh:
				// 结束后，通过ch通知主goroutine
				fmt.Println("stopped")
				ch <- struct{}{}
				return
			default:
				time.Sleep(time.Second)
			}
		}
	}(stopCh)

	<-sig
	// ch作为一个channel，传递给子goroutine，待其结束后从中返回
	ch := make(chan struct{})
	stopCh <- ch
	<-ch
	fmt.Println("finished")
}
```

## 使用Context

``` go
func main() {
	sig := make(chan os.Signal)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGKILL)
	ctx, cancel := context.WithCancel(context.Background())
	finishedCh := make(chan struct{})

	go func(ctx context.Context, finishedCh chan struct{}) {
		for {
			select {
			case <-ctx.Done():
				// 结束后，通过ch通知主goroutine
				fmt.Println("stopped")
				finishedCh <- struct{}{}
				return
			default:
				time.Sleep(time.Second)
			}
		}
	}(ctx, finishedCh)

	<-sig
	cancel()
	<-finishedCh
	fmt.Println("finished")
}
```

## Context + waitGroup

``` go
func main() {
	sig := make(chan os.Signal)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGKILL)
	ctx, cancel := context.WithCancel(context.Background())
	num := 10

	// 用wg来控制多个子goroutine的生命周期
	wg := sync.WaitGroup{}
	wg.Add(num)

	for i := 0; i < num; i++ {
		go func(ctx context.Context) {
			defer wg.Done()
			for {
				select {
				case <-ctx.Done():
					fmt.Println("stopped")
					return
				default:
					time.Sleep(time.Duration(i) * time.Second)
				}
			}
		}(ctx)
	}

	<-sig
	cancel()
	// 等待所有的子goroutine都优雅退出
	wg.Wait()
	fmt.Println("finished")
}
```


### 参考

> [os.Signal](https://golang.org/pkg/os/signal/#Notify)
