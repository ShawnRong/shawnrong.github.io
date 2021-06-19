---
title: Golang的Error Wrapping
date: "2021-05-20"
tags: ["Golang"]
---

Go的错误处理一直是开发者不断吐槽的一个槽点。Go没有像一般语言那样提供`try catch`的处理方式，而是通过函数返回值的方式直接返回。这样就会让代码显的很啰嗦，要不断的对返回的`err`进行判断处理。
