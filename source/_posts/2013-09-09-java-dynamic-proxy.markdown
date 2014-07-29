---
published: false
layout: post
title: "Java动态代理"
date: 2013-09-09 21:53
comments: true
categories: [技术杂谈, Java]
---

前几天在项目里面要用`Spring Batch`配一个bean，但是死活调不通，后来才发现是因为Spring用了动态代理来实现这个类，但我们没给他加上接口，所以不能调用我们定义的方法。真是惭愧，于是回来恶补了一下Java动态代理的相关知识。

