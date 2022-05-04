---
layout: post
title: 为什么说github.com/streadway/amqp的API设计的很烂
---

github.com/streadway/amqp 是一个被写在[RabbitMQ官方文档](https://www.rabbitmq.com/tutorials/tutorial-one-go.html)里的go amqp库，但这个库的API设计的极度不合理，所以不得不开文吐槽一下。

首先这个库是没有java sdk的[自动恢复能力](https://www.rabbitmq.com/api-guide.html#recovery)的，而且这点被明确列在了它家的[Non-goals](https://pkg.go.dev/github.com/streadway/amqp#readme-non-goals)里，意味着永远不会实现，就注定了这个库定位是个low-level sdk，每个操作都要先新建一个Connection，再获取一个Channel。

但具体到最常用的[Consume](https://pkg.go.dev/github.com/streadway/amqp#Channel.Consume)，这画风就不太对了。

这里采用的是方法执行后返回一个go channel的方式来进行消费，但这就有个很大的问题，就是Consume失败导致go channel关闭的话，你是无法知道原因的，到底是被[Cancel](https://pkg.go.dev/github.com/streadway/amqp#Channel.Cancel)了，还是RabbitMQ挂了，还是网络断了，都无从得知，这就让使用起来非常的纠结，因为cancel的话是不应该重试的，但其他几个场景是应该重试，而且策略还应该略有不同，RabbitMQ应该指数退避但网络挂了是不用的。

能找到的唯一获取原因的方法是[NotifyClose](https://pkg.go.dev/github.com/streadway/amqp#Channel.NotifyClose)这个方法，但这个方法也很蛋疼，因为文档写着
> The chan provided will be closed when the Channel is closed and on a graceful close, no error will be sent.

所以这里用起来就非常的蛋疼，要新开一个goroutine来读NotifyClose返回的go channel，想办法把内容传递给外层变量，另一个consume消费的goroutine如果发现consume的go channel关闭了，再去读这个这个外层变量来判断接下来的行为。这里不仅代码变得复杂，而且还有并发问题，毕竟两个goroutine对不同go channel的消费是无序的，如果NotifyClose后发生，那整体判断逻辑就出错了。

这个设计思路缺陷不仅在这里产生巨大混乱，还造成一个功能的缺失，就是服务端生产ConsumerTag的能力，看这个库的[代码](https://github.com/streadway/amqp/blob/v1.0.0/channel.go#L1061-L1063)就能发现，即使传入一个空的ConsumerTag，依然是这个库自己生成一个ConsumerTag，上面的注释写了原因：
> // When we return from ch.call, there may be a delivery already for the  
  // consumer that hasn't been added to the consumer hash yet.  Because of  
  // this, we never rely on the server picking a consumer tag for us.

这也是上面这个不合理的设计思路产生的问题，只是把这个库自己也给坑了，原因就是，这个库一定要返回一个go channel让使用者消费，而在返回前，使用者无法消费任何消息，这个库自己往go channel里放消息就会阻塞，而tcp连接的乱序本质，会产生basic.consume返回的服务端consumer tag比consume读取到的消息要晚这个情况，本来在回调设计里这并不是什么大问题，但在这个库全部逻辑都用go channel，就会阻塞整个tcp连接的消费。

其实这些问题，全部用基于回调的设计模式都可以避免，但很可能是因为这个库早期非常机械的生搬硬套go channel的设计模式，最终造就了这个不可挽回的API设计问题。很不幸的，Azure的AMQP 1.0的sdk[同样有](https://pkg.go.dev/github.com/Azure/go-amqp#example-package)这个设计问题，令人叹息。