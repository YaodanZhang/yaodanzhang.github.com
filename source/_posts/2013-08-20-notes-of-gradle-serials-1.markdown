---
published: true
layout: post
title: "Gradle学习笔记-1：基础知识"
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

## Task之间的依赖

{% codeblock build.gradle %}
task hello << {
    println 'Hello world!'
}

task intro(dependsOn: hello) << {
    println "I'm Gradle"
}
{% endcodeblock %}

执行结果如下：

{% codeblock gradle -q intro %}
Hello World
I'm Gradle
{% endcodeblock %}

## 动态Task

在Gradle里面，Task是可以动态定义的，如下：

{% codeblock build.gradle %}
4.times { counter ->
    task "task$counter"(dependsOn: "depending$counter") << {
        println "I'm task number $counter"
    }
}

4.times { counter ->
    task "depending$counter" << {
        println "I'm depending number $counter"
    }
}
{% endcodeblock %}

执行结果如下：

{% codeblock gradle -q task1 %}
I'm depending number 1
I'm task number 1
{% endcodeblock %}

## 多次定义Task的依赖

{% codeblock build.gradle %}
4.times { counter ->
    task "task$counter"(dependsOn: "depending$counter") << {
        println "I'm task number $counter"
    }
}

task0.dependsOn task1, task2

4.times { counter ->
    task "depending$counter" << {
        println "I'm depending number $counter"
    }
}
{% endcodeblock %}

执行结果如下：

{% codeblock gradle -q task0 %}
I'm depending number 0
I'm depending number 1
I'm task number 1
I'm depending number 2
I'm task number 2
I'm task number 0
{% endcodeblock %}

这里可以看到，Task执行的顺序是根据定义的顺序执行的，如果还有doFirst，则先执行doFirst：

{% codeblock build.gradle %}
task hello << {
    println 'Hello Earth'
}
hello.doFirst {
    println 'Hello Venus'
}
hello.doLast {
    println 'Hello Mars'
}
hello << {
    println 'Hello Jupiter'
}
{% endcodeblock %}

执行结果如下：

{% codeblock gradle -q hello %}
Hello Venus
Hello Earth
Hello Mars
Hello Jupiter
{% endcodeblock %}

## 默认task

gradle中可以定义默认的task，定义后直接使用`gradle`命令就可以执行这些task，代码如下：

{% codeblock build.gradle %}
defaultTasks 'clean', 'run'
task clean << {
    println 'Default Cleaning!'
}
task run << {
    println 'Default Running!'
}
task other << {
    println "I'm not a default task!"
}
{% endcodeblock %}

执行结果如下：

{% codeblock gradle -q %}
Default Cleaning!
Default Running!
{% endcodeblock %}

在这里，执行`gradle`等价于执行`gradle clean run`。