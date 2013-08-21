---
published: false
layout: post
title: "Gradle学习笔记-1"
date: 2013-08-20 09:56
comments: true
categories: [技术杂谈, 构建工具, Gradle]
---

# Hello World

在Gradle中，每一个build都由一个或多个project组成，而每一个project都由一个或多个task组成。

这是一个task的Hello World：

{% codeblock %}
task hello {
	doLast {
		println 'Hello World'
	}
}
{% endcodeblock %}

当执行命令：