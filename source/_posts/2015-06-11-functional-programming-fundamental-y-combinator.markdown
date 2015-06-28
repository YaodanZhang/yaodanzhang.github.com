---
layout: post
title: "函数式编程的基石 - Y Combinator"
date: 2015-06-11 12:39:44 +0800
comments: true
categories: [函数式编程, JavaScript, Lisp, Clojure]
---

本文主要目的是给大家介绍函数式编程中一个非常重要的知识：Y Combinator。包括它是什么，干什么用的，它为什么能够work，它的数学含义以及如何用重构的方式“推倒”他。

<!--more -->

（本文用到的示例代码会用JavaScript和Clojure两种语言来写。）

## 它是什么

Y Combinator也叫做不动点组合子（Fix Point Combinator），是由著名的逻辑学家Haskell Curry发明的（或者说是他发现的）。大家看到这个名字有没有很熟悉的感觉？你猜对了，Haskell语言就是为了纪念他而以他命名的。还有，函数式语言里面著名的柯里化（Currying）也是以他命名的。

我们可以用它来计算一个函数（lambda表达式）的不动点。不知道不动点是什么？翻译成中文来说就是它让我们拥有了在函数式语言里面写递归的能力。

那么问题来了，写个递归很简单嘛，为什么没有它我们就写不了呢？

### Lambda演算

为了让大家更好的理解这个问题，我们还需要学习一点点lambda演算的知识。本来还想先写一篇文章介绍lambda的，后来发现了[这篇文章](http://liujiacai.net/blog/2014/10/12/lambda-calculus-introduction/)。里面的内容很详尽，而且写得比较通俗，不熟悉lambda演算的话可以看看。

当然了，大家以编程语言来理解Y Combinator也可以，但是Church最初提出的lambda演算才是最简洁的，也最容易理解。

看完过后有一点请大家务必牢记在心，并跟我一起默念三遍：“lambda演算中的函数都是匿名函数”，“lambda演算中的函数都是匿名函数”，“lambda演算中的函数都是匿名函数”。

好了，我们步入正题。

## 问题的提出

递归函数，相信大家都会写，我们可以先写一个最简单的递归函数：计算阶乘，然后我们来看看有什么问题：


``` javascript factorial.js
var factorial = function(n) {
  return n == 1 ? 1 : n * factorial(n - 1);
}

factorial(5); // 120
```

``` clojure factorial.clj
(def factorial
  (fn [n] (if (= n 1) 1
             (* n (factorial (- n 1))))))

(factorial 5) ; 120
```

我们刚刚说过了，lambda演算中的函数都是匿名函数，也就是说我们定义的函数名都只是语法糖，真正执行的时候其实会代换成这样（根据lambda演算中的β规约）：


``` javascript factorial.js
(function(n) {
  return n == 1 ? 1 : n * factorial(n - 1);
})(5);
```

``` clojure factorial.clj
((fn [n] (if (= n 1) 1
             (* n (factorial (- n 1)))))
 5)
```

相信到这里大家就能看到问题了，我们用到的函数是没有定义完整的，也就是说在这里，`factorial(n - 1)`解释器是没法处理的。

那么，为什么我们在C++和Java这样的语言里面可以这么写呢？因为他们是有！名！字！的，在编译的时候编译器会对每个函数生成一个符号，在调用的地方会保存目标函数的符号引用，在运行的时候就能确定被调用函数的地址了。

## Y Combinator的数学意义

为了解决这个问题，Curry根据函数的不动点，推导出了Y Combinator，从而在理论上让递归问题有了解决的可能。

我们先来看看什么是一个函数的不动点：


{% codeblock %}
∀F ∃X -> FX = X
{% endcodeblock %}

这句话的意思是对于任意一个函数`F`，存在一个`X`，使得`F(X) = X`。举个栗子，对于抛物线函数`f(x) = x^2`，它的不动点有两个，`0`和`1`，因为`f(0) = 0`，`f(1) = 1`。

在这基础上，Curry定义了这样一个函数：

{% codeblock %}
Y = λf.((λx.f(xx))(λx.f(xx)))
{% endcodeblock %}

他能够达到这样一个目的：对任意的`F`，都能满足`F(YF) = YF`。

## Y Combinator的工程意义

那么这个表达式和我们的递归函数之间有什么关系呢？就这样看确实很难看出他们之间有什么关系，那就先不管他，我们先把递归函数的问题解决了。

我们知道，Lambda演算是与图灵机具有相同计算能力的，那么既然基于图灵机模型发展出来的语言能写递归，我们就应该坚信函数式语言也能写递归。

怎么写？

还是以阶乘为例子，我们可以试着用重构的方式来做。重构之前，我们应该先在心里有个大致的方向：1. 函数要匿名；2. 要尽量通用。

好了，那我们现在就开始重构。

### 1. 先写一个“正常”的递归函数

``` javascript factorial.js
var factorial = function(n) {
  return n == 1 ? 1 : n * factorial(n - 1);
}

factorial(5); // 120
```

``` clojure factorial.clj
(def factorial
  (fn [n] (if (= n 1) 1
             (* n (factorial (- n 1))))))

(factorial 5) ; 120
```

### 2. 把自己作为参数

我们说过了上面这个函数式有问题的，问题就在于函数内部的`factorial`调用是在调用一个还没有定义出来的函数。那么，我们就干脆不让他调用自己了，而是去调用一个已经定义好的函数不就好了吗？那么如何来调用一个定义好了的函数呢？

把自己作为参数传进去嘛。也就是说给`factorial`函数增加一个参数，这个参数是一个函数，用来计算`(n - 1)`的递归。代码如下：

``` javascript factorial.js
var factorial = function(f, n) {
  return n == 1 ? 1 : n * f(f, n - 1);
}

factorial(factorial, 5); // 120
```

``` clojure factorial.clj
(def factorial
  (fn [f n] (if (= n 1) 1
             (* n (f f (- n 1))))))

(factorial factorial 5) ; 120
```

### 3. 柯里化

我们刚刚的修改让`factorial`函数有了两个参数，这是不好的。函数式编程里面，我们应该让所有的函数都只有一个参数，这个过程就是上文提到的柯里化。关于他的好处我们就不展开了，有兴趣的同志可以去看看[这篇文章](https://en.wikipedia.org/wiki/Currying)。代码如下：

``` javascript factorial.js
var factorial = function(f) {
  return function(n) {
    return n == 1 ? 1 : n * f(f)(n - 1);
  }
}

factorial(factorial)(5); // 120
```

``` clojure factorial.clj
(def factorial
  (fn [f] (fn [n] (if (= n 1) 1
                    (* n ((f f) (- n 1)))))))

((factorial factorial) 5) ; 120
```

### 4. 去除重复的自身调用

我们看到，这两段代码可以工作了，但是还是有个问题，就是调用起来太恶心，`factorial(factorial)(n)`或者`((factorial factorial) n)`都把自己传了两遍，我们可以把这段代码抽成一个函数，从而让最终用户的调用更优雅一些。代码如下：

``` javascript factorial.js
var self_invoke = function(f) {
  return f(f);
};

var factorial = self_invoke(function(f) {
  return function(n) {
    return n == 1 ? 1 : n * self_invoke(f)(n - 1);
  };
});

factorial(5); // 120
```

``` clojure factorial.clj
(def self-invoke
  (fn [f] (f f)))

(def factorial
  (self-invoke
   (fn [f]
     (fn [n] (if (= n 1) 1
                (* n ((self-invoke f) (- n 1))))))))

(factorial 5) ; 120
```

### 5. Inline

上面的代码定义了两个函数，其中的`self-invoke`是用的有名字的调用，我们的重构目标是使用匿名函数，所以这一步比较简单，就是把`self-invoke`函数inline。代码如下：

``` javascript factorial.js
var factorial = (function(self_invoke) {
  return self_invoke(function(f) {
    return function(n) {
      return n == 1 ? 1 : n * self_invoke(f)(n - 1);
    };
  });
})(function(f) {
  return f(f);
});

factorial(5); // 120
```

``` clojure factorial.clj
(def factorial
  ((fn [self-invoke]
     (self-invoke
      (fn [f]
        (fn [n]
          (if (= n 1) 1
              (* n ((self-invoke f) (- n 1))))))))
   (fn [f] (f f))))

(factorial 5) ; 120
```

### 6.抽出伪递归

我们看到，上面这段代码中

``` javascript factorial.js
function(n) {
  return n == 1 ? 1 : n * self_invoke(f)(n - 1);
};
```

``` clojure factorial.clj
(fn [n]
  (if (= n 1) 1
      (* n ((self-invoke f) (- n 1)))))
```

这个结构跟我们最开始的递归函数很像，唯一的区别是这里用`self_invoke(f)`代替了递归函数中的函数名调用。那么我们可以试着把这一点再抽出来，写出我们最开始的递归函数的样子：

``` javascript factorial.js
(function(self){
  return function(n) {
    return n == 1 ? 1 : n * self(n - 1);
  };
})(function(m) {
  return self_invoke(f)(m);
}));
```

``` clojure factorial.clj
((fn [self]
  (fn [n]
    (if (= n 1) 1
        (* n (self (- n 1)))))
 (fn [m] ((self-invoke f) m)))
```

这样我们就能很清楚的看到，这一部分代码：

``` javascript factorial.js
function(self){
  return function(n) {
    return n == 1 ? 1 : n * self(n - 1);
  };
}
```

``` clojure factorial.clj
(fn [self]
  (fn [n]
    (if (= n 1) 1
        (* n (self (- n 1))))))
```

简直跟我们直接写出来的递归函数“一摸一样”，唯一的区别是这里增加了一层函数（参数是self）。很明显，如果我们写了一个直接递归的阶乘函数，我们可以很容易的把这个函数改写成上面的样子。更进一步，如果我们写了任意一个递归函数，我们也能很轻易的把它改写成上面的样子。我们把这个函数就叫做`伪递归函数`。

这个`伪递归函数`在我们推导过程中是一个`可变量`，如果能够把它提取出来，那么剩下的就是`不变量`。也就是说，我们可以把这个`不变量`应用到任何一个`伪递归函数`上面，就能得到我们真正可以递归的函数了。

代码如下：

``` javascript factorial.js
var fake_factorial = function(self) {
  return function(n) {
    return n == 1 ? 1 : n * self(n - 1);
  };
}

var y = function(fake_rec) {
  return (function(self_invoke) {
    return self_invoke(function(f) {
      return fake_rec(function(m) {
        return self_invoke(f)(m);
      });
    });
  })(function(f) {
    return f(f);
  });
};

var factorial = y(fake_factorial);

factorial(5); // 120
```

``` clojure factorial.clj
(def fake-factorial
  (fn [self]
    (fn [n]
      (if (= n 1) 1
          (* n (self (- n 1)))))))

(def y
  (fn [fake-rec] ((fn [self-invoke]
                   (self-invoke
                    (fn [f]
                      (fake-rec
                       (fn [m] ((self-invoke f) m))))))
                 (fn [f] (f f)))))

(def factorial
  (y fake-factorial))

(factorial 5) ; 120
```

这里，`y`是不变量，`fake-factorial`是变量。将任意的一个伪递归函数作为实参调用`y`，就能得到真正的递归函数了。这里的`y`也就是我们的Y Combinator。

这里推导出来的`y`还是稍显复杂，我们可以再重构一下，变成这样：

``` javascript factorial.js
var y = function(fake_rec) {
  return (function(f) {
    return f(f);
  })(function(f) {
    return fake_rec(function(x) {
      return f(f)(x);
    });
  });
};
```

``` clojure factorial.clj
(def y
  (fn [fake-rec]
    ((fn [f] (f f)) (fn [f]
                     (fake-rec (fn [x] ((f f) x)))))))
```

## 总结

到这里，`Y Combinator`就已经推导完了。

回到我们最开始的地方，`Y Combinator`的数学定义是对任意的`F`，都能满足`F(YF) = YF`。结合我们的推导，可以看出，这里的`F`其实就是我们的`伪递归函数`，`YF`其实就是我们真实的递归函数。伪递归函数`F`和我们真正的递归函数`f`之间的关系就是`F(f) = f`。

## 致谢

感谢太君的[这篇文章](http://cuipengfei.me/blog/2013/04/09/make-y/)的启发

## Reference

- [康托尔、哥德尔、图灵——永恒的金色对角线](http://mindhacks.cn/2006/10/15/cantor-godel-turing-an-eternal-golden-diagonal/)，刘未鹏。
- [如何一步一步推导出Y Combinator](http://cuipengfei.me/blog/2013/04/09/make-y/)，崔鹏飞。
- [y-combinator.js](https://github.com/zhewuzhou/js-y-combinator/blob/master/y-combinator.js)，周哲武。
