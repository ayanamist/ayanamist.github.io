---
layout: post
title: Go里如何组合多个error
---

起因是这个：proposal: errors: add With(err, other error) error [#52607](https://github.com/golang/go/issues/52607)

这其实在其他有异常的语言里是很常见也很容易实现的一个场景，例如Java的Throwable就可以用[构造器](https://docs.oracle.com/javase/8/docs/api/java/lang/Throwable.html#Throwable-java.lang.String-java.lang.Throwable-)进行嵌套，这样可以一层一层的`getCause`然后`instanceof`判断类型。

但Go里的`fmt.Errorf`是无类型的，所以如果要错误嵌套，就必须在各个Error对象上自己实现嵌套的能力，这显然非常的麻烦，而且有些错误是库里的，并不好做这个改动。

但我们回顾下errors包的使用，主要判断方式就是`Is`和`As`两个，直接用`Unwrap`的场景并不多，而且errors包的[`Is`](https://cs.opensource.google/go/go/+/refs/tags/go1.13:src/errors/wrap.go;l=31)和[`As`](https://cs.opensource.google/go/go/+/refs/tags/go1.13:src/errors/wrap.go;l=66)方法在实现上也会判断被比较对象是不是实现了这两个方法并返回true，所以自己简单写一个，把多个error给包起来，同时也实现`Is`和`As`方法进行逐个比较就完事，50行代码还是很简单的

代码贴在这里：https://gist.github.com/ayanamist/f988de9c073ca7d07a29820621bdaeb3