---
layout: post
title: PowerMock到底是怎么和JaCoCo冲突的
---

PowerMock 底层用的是 [Javassist](https://www.javassist.org/) 来修改字节码的，按[这篇文章里 4.2 节](https://mp.weixin.qq.com/s/9eevIy9tQ9v4P_xEKk6_qg#:~:text=%E5%8D%B4%E6%B2%A1%E7%94%9F%E6%95%88-,4.2%20%E5%86%B2%E7%AA%81%E6%A0%B9%E5%9B%A0,-%E9%92%88%E5%AF%B9%E4%B8%8A%E9%9D%A2%E7%BB%93%E6%9E%9C)的描述，因为 [ClassPool](https://www.javassist.org/tutorial/tutorial.html#pool) 始终从文件系统加载 class 文件，导致如果有其它 java agent 修改了内存中的字节码，就会被丢掉。而 jacoco 刚好是这样做的。

所以只能用 [jacoco offline](https://github.com/powermock/powermock/wiki/Code-coverage-with-JaCoCo#offline-instrumentation) 来绕过这个问题，不过注意 pom.xml 的写法要参考[这里](https://github.com/powermock/powermock-examples-maven/blob/master/jacoco-offline/pom.xml)，要点是
1. 显式设置 `jacoco-agent.destfile` 放到target目录里，不然会由于执行时当前目录的问题，被放在项目根目录下面，导致其它环节都找不到
2. jacoco:restore-instrumented-classes 要在 jacoco:report 之前，这样才能正常生成html报告，可以把这两个的phase都设置成test这样就会挨个执行了