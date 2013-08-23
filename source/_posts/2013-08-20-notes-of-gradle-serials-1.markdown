---
published: false
layout: post
title: "Gradle学习笔记-1"
date: 2013-08-20 09:56
comments: true
categories: [技术杂谈, 构建工具, Gradle]
---

## Hello World

在Gradle中，每一个build都由一个或多个project组成，而每一个project都由一个或多个task组成。

这是一个task的Hello World：

{% codeblock build.gradle %}
task hello {
    doLast {
        println 'Hello World'
    }
}
{% endcodeblock %}

<!--more -->

当执行命令：`gradle -q hello`时输出：

{% codeblock gradle -q hello %}
Hello World
{% endcodeblock %}

至于这个参数`-q`是什么意思，我们可以试试不加参数时的输出：

{% codeblock gradle hello %}
:hello
Hello World

BUILD SUCCESSFUL

Total time: 1.879 secs
{% endcodeblock %}

可以看到，`-q`的意思不显示Gradle本身的log。

对于上面的`doLast`块，有一种简写的方式：

{% codeblock build.gradle %}
task hello << {
    println 'Hello World'
}
{% endcodeblock %}

## 使用Groovy

在Gradle脚本中可以使用Groovy的语法，如下：

{% codeblock build.gradle %}
task upper << {
    String someString = 'mY_nAmE'
    println "Original: " + someString
    println "Upper case: " + someString.toUpperCase()
}
{% endcodeblock %}

当执行`gradle -q upper`时，输出如下：

{% codeblock gradle -q upper %}
Original: mY_nAmE
Upper case: MY_NAME
{% endcodeblock %}

