---
title: Laravel 生命周期
date: "2017-11-08"
tags: ["Laravel", "PHP"]
---
# Laravel 生命周期

## 生命周期概述

### 入口

`publuc/index.php`是一个Laravel应用程序的入口，是整个框架的起点。`index.php`代码不多，主要的阶段就是：

1. 加载Composer项目依赖
2. 创建一个app实例容器
3. 接收并且处理http请求

## 生命周期详解

### 加载项目依赖

Laravel使用Composer进行包的管理，所有组件的加载工作，仅需要一行代码

```php
require __DIR__.'/../vendor/autoload.php';
```

### 创建App实例

接下来便是创建应用实例(`Illuminate\Foundation\Application`)，也叫服务容器

```php
$app = require_once __DIR__.'/../bootstrap/app.php';
```

整个初始化的过程包括：注册项目基础的ServiceProvider,注册SerciveProvider的Alias,注册目录路径等。

`bootstrap/app.php`中也完成了内核绑定。

Laravel会依据http请求的运行环境不同，将请求发送至相应的内核**HTTP内核** 或 **Console内核**。无论哪个内核，它们作用都是处理http请求。

最终，HTTP内核用`handle`method,单纯的接收一个`Request`以及返回一个`Response`。

#### HTTP内核

HTTP内核继承了`Illuminate\Foundation\Http\Kernal`类，它定义了在执行请求之前运行的 `bootstrappers` 数组。包含完成环境检测，配置加载，异常处理，Facades注册，ServiceProvider注册，启动服务这6个引导程序。

HTTP内核定义了所有被请求应用程序处理之前必须经过的HTTP中间件列表。 这些中间件可以处理 HTTP session 的读写, 可以判断服务器当前是否处于维护模式, 验证 CSRF token ( 为了保护服务器不受 [CSRF 攻击](https://laravel-china.org/docs/laravel/5.2/routing#csrf-protection) ) 等等功能.

#### ServiceProvider

最重要的引导操作之一就是加载应用程序的ServiceProvider。应用程序的所有ServiceProvider都在`config/app.php`配置文件的`providers`数组中配置。所有的provider都会调用`register`方法，由`boot`方法负责调用所有的被注册provider。

ServiceProvider负责引导所有框架的各种组件，如数据库、队列、验证和路由组件。也就是说，框架提供的每个功能都它们来引导并配置。因此也可以说，ServiceProvider是整个 Laravel 引导过程中最重要的方面。

### 接收并处理请求

处理请求包含两个阶段：

- 创建请求实例
- 处理请求

#### 创建请求实例

	请求实例`Illuminate\Http\Request`的`capture()	`方法，内部通过Symfony实例创建一个Laravel请求实例

```php
    /**
     * Create a new Illuminate HTTP request from server variables.
     *
     * @class Illuminate\Http\Request
     * @return static
     */
    public static function capture()
    {
        static::enableHttpMethodParameterOverride();
        return static::createFromBase(SymfonyRequest::createFromGlobals());
    }

    /**
     * Create an Illuminate request from a Symfony instance.
     *
     * @see https://github.com/symfony/symfony/blob/master/src/Symfony/Component/HttpFoundation/Request.php
     * @param  \Symfony\Component\HttpFoundation\Request  $request
     * @return \Illuminate\Http\Request
     */
    public static function createFromBase(SymfonyRequest $request)
    {
        if ($request instanceof static) {
            return $request;
        }

        $content = $request->content;

        $request = (new static)->duplicate(
            $request->query->all(), $request->request->all(), $request->attributes->all(),
            $request->cookies->all(), $request->files->all(), $request->server->all()
        );

        $request->content = $content;

        $request->request = $request->getInputSource();

        return $request;
    }
```

#### 处理请求

请求处理发生在HTTP内核的`handle()`方法内

```php
/**
     * Handle an incoming HTTP request.
     *
     * @class Illuminate\Foundation\Http\Kernel
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function handle($request)
    {
        try {
            $request->enableHttpMethodParameterOverride();

            $response = $this->sendRequestThroughRouter($request);
        } catch (Exception $e) {
            $this->reportException($e);

            $response = $this->renderException($request, $e);
        } catch (Throwable $e) {
            $this->reportException($e = new FatalThrowableError($e));

            $response = $this->renderException($request, $e);
        }

        $this->app['events']->dispatch(
            new Events\RequestHandled($request, $response)
        );

        return $response;
    }

```

`handle()`方法接收一个HTTP请求，返回一个HTTP响应。

### 发送响应

发送响应由`Illuminate\Http\Response`父类`Symfony\Component\HttpFoundation\Response`中的`send()`方法完成。

```php
   /**
     * Sends HTTP headers and content.
     *
     * @see https://github.com/symfony/symfony/blob/master/src/Symfony/Component/HttpFoundation/Response.php
     * @return $this
     */
    public function send()
    {
        $this->sendHeaders();// 发送响应头部信息
        $this->sendContent();// 发送报文主题

        if (function_exists('fastcgi_finish_request')) {
            fastcgi_finish_request();
        } elseif (!\in_array(PHP_SAPI, array('cli', 'phpdbg'), true)) {
            static::closeOutputBuffers(0, true);
        }
        return $this;
    }
```

### 终止程序

程序终止，完成`terminateMiddleware`的调用

### 参考

>- [深度挖掘 Laravel 生命周期](https://laravel-china.org/articles/10421/depth-mining-of-laravel-life-cycle)
>- [Request Life Cycle of Laravel](http://blog.mallow-tech.com/2016/06/request-life-cycle-of-laravel/)
>- [Laravel请求生命周期](https://www.dyike.com/2017/04/22/laravel-request-life-cycle/)
>- [Laravel 的请求周期](https://laravel-china.org/docs/laravel/5.6/lifecycle/1358)

