---
title: 理解Laravel IOC容器
date: "2017-11-25"
tags: ["PHP", "Composer"]
---
# 理解Laravel IOC容器

IOC容器是Laravel框架一个非常重要的概念

## 依赖注入

理解IOC容器首先要从依赖注入开始。**依赖注入**和**控制反转**是差不多因果关系，通过使用**依赖注入**这种手段实现功能模块对其依赖组件的**控制反转**。

拿一个使用OAuth登录应用场景举例：

```php
interface Login {
    public function login();
}
//微信账号登录
class WechatLogin implements Login {
    public function __construct(){}
    public function login() {}
}
//新浪微博登录
class WeiboLogin implements Login {
    public function __construct(){}
    public function login() {}
}
//QQ登录
class QQLogin implements Login {
    public function __construct(){}
    public function login() {}
}
//站点登录
class SiteLogin {
    protected $oauthClient;
    public function setOauthClient($oauthClient) {
        $this->oauthClient = $oauthClient
    }
    public function appLogin() {
        $this->oauthClient->login();
    }
}

```

一般可以使用两种方法实现注入

1. 通过构造函数注入依赖
2. 通过setter设置方法注入

依赖注入的作用就是使程序更容易维护，降低代码的耦合度。但是，依赖注入也有缺点，就是每次都需要实例化，如果组件中有很多的依赖关系，就需要多个setter或者创建构造方法传递。这就需要用到了**依赖注入容器**

## Laravel的容器

在Laravel框架中，服务容器是整个框架的核心，它提供了整个系统功能及服务的配置，调用。当程序运行时，我们把需要的一些服务放到或者注册到(bind)到容器中，当我们需要的时候直接取出来(make)。

```php
<?php

//容器类装实例或提供实例的回调函数
class Container {

    //用于装提供实例的回调函数，真正的容器还会装实例等其他内容
    //从而实现单例等高级功能
    protected $bindings = [];

    //绑定接口和生成相应实例的回调函数
    public function bind($abstract, $concrete=null, $shared=false) {

        //如果提供的参数不是回调函数，则产生默认的回调函数
        if(!$concrete instanceof Closure) {
            $concrete = $this->getClosure($abstract, $concrete);
        }

        $this->bindings[$abstract] = compact('concrete', 'shared');
    }

    //默认生成实例的回调函数
    protected function getClosure($abstract, $concrete) {

        return function($c) use ($abstract, $concrete) {
            $method = ($abstract == $concrete) ? 'build' : 'make';
            return $c->$method($concrete);
        };

    }

    public function make($abstract) {
        $concrete = $this->getConcrete($abstract);

        if($this->isBuildable($concrete, $abstract)) {
            $object = $this->build($concrete);
        } else {
            $object = $this->make($concrete);
        }

        return $object;
    }

    protected function isBuildable($concrete, $abstract) {
        return $concrete === $abstract || $concrete instanceof Closure;
    }

    //获取绑定的回调函数
    protected function getConcrete($abstract) {
        if(!isset($this->bindings[$abstract])) {
            return $abstract;
        }

        return $this->bindings[$abstract]['concrete'];
    }

    //实例化对象
    public function build($concrete) {

        if($concrete instanceof Closure) {
            return $concrete($this);
        }

        $reflector = new ReflectionClass($concrete);
        if(!$reflector->isInstantiable()) {
            echo $message = "Target [$concrete] is not instantiable";
        }

        $constructor = $reflector->getConstructor();
        if(is_null($constructor)) {
            return new $concrete;
        }

        $dependencies = $constructor->getParameters();
        $instances = $this->getDependencies($dependencies);

        return $reflector->newInstanceArgs($instances);
    }

    //解决通过反射机制实例化对象时的依赖
    protected function getDependencies($parameters) {
        $dependencies = [];
        foreach($parameters as $parameter) {
            $dependency = $parameter->getClass();
            if(is_null($dependency)) {
                $dependencies[] = NULL;
            } else {
                $dependencies[] = $this->resolveClass($parameter);
            }
        }

        return (array)$dependencies;
    }

    protected function resolveClass(ReflectionParameter $parameter) {
        return $this->make($parameter->getClass()->name);
    }

}
```

以上代码定义了一个容器

```php
$app = new Container();
$app->bind("Login", "WechatLogin");//Login 为接口， WechatLogin 是 class WechatLogin
$app->bind("siteLogin", "SiteLogin"); //siteLogin可以当做是Class SiteLogin 的服务别名

//通过字符解析，或得到了Class SiteLogin 的实例
$login = $app->make("siteLogin");

$login->appLogin();
```



### 参考

>- [如何理解 Laravel 的 IoC 容器](https://laravel-china.org/articles/4076/how-to-understand-laravels-ioc-container)
>- [**Laravel IoC Container: Why we need it and How it works**](https://medium.com/@NahidulHasan/laravel-ioc-container-why-we-need-it-and-how-it-works-a603d4cef10f)
>- [Laravel5.6文档](https://laravel.com/docs/5.6/container)