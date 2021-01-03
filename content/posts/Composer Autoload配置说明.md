---
title: Composer Autoload 配置说明
date: "2017-11-18"
tags: ["Composer", "PHP"]
---
# Composer Autoload 配置说明

谈到现代PHP,肯定离不开Composer。对于库的自动加载信息，Composer 生成了一个 `vendor/autoload.php` 文件。你可以简单的引入这个文件，你会得到一个免费的自动加载支持。(注意⚠️：php5.3之后才有namespace)

```php
require 'vendor/autoload.php';
```

**namespace**的应用大大的给开发提供的便利。autoload 是composer.json中的一个配置参数。autoload利用命名空间进行对应规则或标准的路径映射，从而找到最终的类文件。

## 四种Autoload模式

### 1. PSR-0

在 `psr-0` key 下你定义了一个命名空间到实际路径的映射（相对于包的根目录）。注意，这里同样支持 PEAR-style 方式的约定（与命名空间不同，PEAR 类库在类名上采用了下划线分隔）。

请注意，命名空间的申明应该以 `\\` 结束，以确保 autoloader 能够准确响应。

在 install/update 过程中，PSR-0 引用都将被结合为一个单一的键值对数组，存储至 `vendor/composer/autoload_namespaces.php` 文件中。

```json
{
    "autoload": {
        "psr-0": {
            "Monolog\\": "src/",
            "Vendor\\Namespace\\": "src/",
            "Vendor_Namespace_": "src/"
        }
    }
}
```

⚠️下划线 _ 对 psr-0 是有特殊意义的。psr-0 的加载器会将类名中的 _ 解析成目录分隔符。

即 Foo_Bar_Test 类会去加载 Foo/Bar/Test.php 文件。

### 2. PSR-4

将实际路径定义为命名空间。

```php
{
    "autoload": {
        "psr-4": {
            "Monolog\\": "src/",
            "Vendor\\Namespace\\": ""
        }
    }
}
```

### 3. Classmap

`classmap` 引用的所有组合，都会在 install/update 过程中生成，并存储到 `vendor/composer/autoload_classmap.php` 文件中。这个 map 是经过扫描指定目录（同样支持直接精确到文件）中所有的 `.php` 和 `.inc` 文件里内置的类而得到的。当载入需要的类时直接取出路径，速度最快

你可以用 classmap 生成支持支持自定义加载的不遵循 PSR-0/4 规范的类库。要配置它指向需要的目录，以便能够准确搜索到类文件。

```php
{
    "autoload": {
        "classmap": ["src/", "lib/", "Something.php"]
    }
}
```

### 4. Files

如果你想要明确的指定，在每次请求时都要载入某些文件，那么你可以使用 'files' autoloading。通常作为函数库的载入方式（而非类库）

```json
{
    "autoload": {
        "files": ["src/MyLibrary/functions.php"]
    }
}
```

主要用来载入一些没办法懒加载的公共函数, 比如一些工具函数。

## PSR-0和PSR-4区别

1. 在composer中定义的**namespace**，PSR-4必须以\结尾否则会抛出异常，PSR-0则不要求

2. PSR-0里面最后一个\之后的类名中，如果有下划线，则会转换成路径分隔符，如Name_Space_Test会转换成Name\Space\Test.php。在PSR-4中下划线不存在实际意义

3. PSR-0有更深的目录结构，比如定义了**namespace**为` Foo\Bar=>vendor\foo\bar\src`，
   use Foo\Bar\Tool\Request调用**namespace**。
   如果以PSR-0方式加载，实际的目录为`vendor\foo\bar\src\Foo\Bar\Tool\Request.php`
   如果以PSR-4方式加载，实际目录为`vendor\foo\bar\src\Tool\Request.php`

## 其他

当你添加了新的  psr-0/psr-4 的规则，或者在 classmap/files 规则相应的目录下新增了文件时，都需要执行 dump-autoload 来刷新系统的自动载入。

### 参考

> [Composer文档](https://docs.phpcomposer.com/04-schema.html#autoload)
>
> [composer 自动载入 autoload 的使用详解 ](https://my.oschina.net/sallency/blog/893518)