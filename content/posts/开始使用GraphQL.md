---
title: 开始使用GraphQL
date: "2018-05-08"
tags: ["GraphQL"]
---

## GraphQL简介

GraphQL是一种api查询语言，GraphQL并不用绑定具体的数据库或者存储引擎,它是描述请求的一个规范，类似于RESTful, 可以利用已有的代码和技术来进行数据源管理。一个GraphQL查询是一个被发往服务端的字符串，该查询在服务端被解释和执行后返回JSON数据给客户端。

## GraphQL基本语法
GraphQL基本语法可以分为 **Fields**,**Arguments**, **Alias**, **Fragments**,**Operation name**,**Variables**
操作可以分为**Query**和**Mutation**，query就是对数据进行查询，而mutation则是对数据进行操作，如增删改。
GraphQL是强类型的协议，支持的具体的数据类型有`Int`, `Float`, `String`, `Boolean`, `ID`

### Query

```
//下面是一个简单的GraphQL查询，获取id为1的用户的ID，名字,邮箱，所有评论
//其中id,name,email,comments都为fields
//id:1为arguments
//nickname为alias
//...queryComments是fragment
//findUser是operation name 可以省略
query findUser {
	user(id: 1) {
	   id
	   name:nickname #可以给字段设置alias
	   email
	   comments {
	   	content
	   }
	   # ...queryComments
	}
}
fragment queryComments on Comments {
	content
}
```

### Mutation

```
//新建一个用户，然后返回id和名字
//❗️表示字段必输
mutation createUser($id: ID!, $name: String!, $email: String) {
   createUser(id: $id, name: $name, email: $email) {
   	id
   	name
   }
}
//variables
{
   "id": 1,
   "name": "Tom",
   "email": "tom@test.com"
}
```


## 与RESTful api对比

1. **可拓展性**。随着一个API的不断发展，需求的信息可能随时发生变化。如果需要前后端API信息匹配，这样就增加了开发和维护的难度。往往会有冗余信息。
2. **同时对多个API接口的调用**。往往会有一个复杂的数据请求，需要调用用户信息，评论接口等等，RESTful需要进行多次调用，这样的数据获取方式非常不灵活。

当然，GraphQL也有存在的问题，GraphQL每个查询的字段都有自己的一个resolve的方法，如果一次查询操作对数据库跑了大量的query,数据库的查询可能会成为性能瓶颈

## GraphQL应用场景
- 字段冗余 节约带宽
- 参数的类型校验 强类型协议
- 文档与维护， 文档会自动生成
- 请求多个接口

> [graplql.org](http://graphql.org/learn/)

