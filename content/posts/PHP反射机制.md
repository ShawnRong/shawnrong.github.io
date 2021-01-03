---
title: PHP反射机制
date: "2017-11-21"
tags: ["PHP"]
---
# PHP反射机制

## 介绍

>**reflection** is the ability of a computer program to examine, introspect, and modify its own structure and behavior at runtime — wikipedia

反射的关键点就是在运行时分析类或者对象的状态，导出或提取出关于类，方法，属性，参数等信息。

## 代码例子

Reflection/Profile.php

```php
namespace codetest\Reflection;

/**
 * Class Profile
 *
 * @package codetest\Reflection
 */
class Profile
{

    public function getUserName()
    {
        return 'Foo';
    }
}
```



```php
$reflectionClass = new ReflectionClass('codetest\Reflection\Profile');
//当然也可以 $reflectionClass = new ReflectionClass(codetest\Reflection\Profile::class);
var_dump($reflectionClass->getName());
// output: codetest\Reflection\Profile
var_dump($reflectionClass->getDocComment());
// output: /**
// * Class Profile
// *
// * @package codetest\Reflection
// */
```



## 应用场景

PHP的反射API,一般用到`ReflectionClass`和`ReflectionMethod`，

常用`ReflectionClass`API

```php+HTML
ReflectionClass::getMethods     获取方法的数组
ReflectionClass::getName        获取类名
ReflectionClass::hasMethod      检查方法是否已定义
ReflectionClass::hasProperty    检查属性是否已定义
ReflectionClass::isAbstract     检查类是否是抽象类（abstract）
ReflectionClass::isFinal        检查类是否声明为 final
ReflectionClass::isInstantiable 检查类是否可实例化
ReflectionClass::newInstance    从指定的参数创建一个新的类实例
```

常用`ReflectionMethod`API

```php+HTML
ReflectionMethod::invoke        执行
ReflectionMethod::invokeArgs    带参数执行
ReflectionMethod::isAbstract    判断方法是否是抽象方法
ReflectionMethod::isConstructor 判断方法是否是构造方法
ReflectionMethod::isDestructor  判断方法是否是析构方法
ReflectionMethod::isFinal       判断方法是否定义 final
ReflectionMethod::isPrivate     判断方法是否是私有方法
ReflectionMethod::isProtected   判断方法是否是保护方法 (protected)
ReflectionMethod::isPublic      判断方法是否是公开方法
ReflectionMethod::isStatic      判断方法是否是静态方法
ReflectionMethod::setAccessible 设置方法是否访问
```



下面是**Reflection API**的几个应用场景：

1. 获取指定类的父类

   ```php
   $class = new ReflectionClass('Child');
   $class->getParentClass()
   ```


2. 获取类方法的注释文档

   ```php
   $method = new ReflectionMethod('Profile', 'getUserName');
   $method->getDocComment();
   ```

3. 也可以使用**instanceof()**和**is_a()**方法来判断反射类

   ```php
   $class = new ReflectionClass('Profile');
   $obj   = new Profile();
   var_dump($class->isInstance($obj)); // bool(true)
   var_dump(is_a($obj, 'Profile')); // bool(true)
   var_dump($obj instanceof Profile); // bool(true)
   ```

4. 获取方法调用

   ```php
   $stu = new Student();
   $ref = new ReflectionClass(Student::class);
   $method = $ref->getMethod('setName');
   $method->invoke($stu, 'john');
   var_dump($stu->name);
   ```

5. 执行private方法

   ```php
   class Student {
       private    $name;

       private function setName($name)
       {
           $this->name = $name;
       }
   }
   $stu = new Student();
   $ref = new ReflectionClass($stu);
   $method = $ref->getMethod('setName');
   $method->setAccessible(true);
   $method->invoke($stu, 'john');
   //读取私有属性
   $stu = new Student();
   $ref = new ReflectionClass($stu);
   $prop = $ref->getProperty('name');
   $prop->setAccessible(true);
   $val = $prop->getValue($stu);
   var_dump($val);
   ```


## Laravel 中的反射

Laravel中的依赖注入也用到了反射的方法

`make`方法源码

```php
public function build($concrete){
    $reflector = new ReflectionClass($concrete);
    $constructor = $reflector->getConstructor();

    if (is_null($constructor)) {
        return new $concrete;
    }

    $dependencies = $constructor->getParameters();
    $instances = $this->resolveDependencies($dependencies);
    return $reflector->newInstanceArgs($instances);
}
```



### 参考

> [Dependency Injection (DI) Container in PHP](https://medium.com/tech-tajawal/dependency-injection-di-container-in-php-a7e5d309ccc6)
>
> [Introduction to PHP Reflection API](https://medium.com/tech-tajawal/introduction-to-php-reflection-api-4af07cc17db4)