---
layout: post
title: mockito-inline最新版是怎么不需要配置javaagent也能工作的
---

在解决厂里PowerMock和jacoco不兼容的问题，然后发现最新版的mockito其实已经支持了大部分powermock的功能（例如 [Mocking final types, enums and final methods](https://javadoc.io/static/org.mockito/mockito-core/4.3.1/org/mockito/Mockito.html#39)、[Mocking static methods](https://javadoc.io/static/org.mockito/mockito-core/4.3.1/org/mockito/Mockito.html#static_mocks)、[Mocking object construction](https://javadoc.io/static/org.mockito/mockito-core/4.3.1/org/mockito/Mockito.html#mocked_construction) ）并且可以兼容jacoco，需要使用 mockito-inline 而不是 mockito-core ，两者的差别见[这里](https://stackoverflow.com/questions/65986197/difference-between-mockito-core-vs-mockito-inline)。

但却发现它文档里提到

> When running on a non-JDK VM prior to Java 9, it is however possible to manually add the Byte Buddy Java agent jar using the -javaagent parameter upon starting the JVM.

这就很不幸，厂里都是jdk8，相信业界也是如此，而现在没有现成的插件在surefire里给这货加argLine，地址比较难取的。

不死心，试了一下，发现居然是能work的，就好奇到底是怎么回事。翻了下代码，发现其实这段文档写的过时了，引入 `org.mockito.internal.creation.bytebuddy.InlineByteBuddyMockMaker` 这个类的第一天，就有了 [ByteBuddyAgent.install()](https://github.com/mockito/mockito/blob/6ccc12149abc98d072de3992da1f18ea58c4c7d9/src/main/java/org/mockito/internal/creation/bytebuddy/InlineDelegateByteBuddyMockMaker.java#L115) 而这个方法[写明了](https://javadoc.io/static/net.bytebuddy/byte-buddy-agent/1.12.8/net/bytebuddy/agent/ByteBuddyAgent.html#install--)，其实大部分jdk8是可以用的，只有jre8不能用。只要把这个方法放到static块里，就能不用手动添加`-javaagent`参数了。

把这个问题给mockito官方提了个[issue](https://github.com/mockito/mockito/issues/2577)，希望能改下文档，避免吓到不知情的小朋友。