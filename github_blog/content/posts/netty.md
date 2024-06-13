---
title: "Netty"
date: 2023-11-09T14:02:05+08:00
draft: truey一级
---





### 简单的说下nio是什么

一个线程（对应一个selector） ，可以监听多个channel上发生的事件

与传统的一个线程处理一个channel 以及利用一个线程池处理多个channel相比。每此selector.select()方法调用结束后，可以获取到注册到该select上所有channel上的事件。并且进行处理。

而传统的单线程是，一个线程只能监听一个channel发生的事件，且一个read()方法阻塞了，其他的read（）事件就不能处理





nio适合连接数特别多，但流量低的场景



### Netty里的几种 EventLoopGroup

第一种.  NioEventLoopGroup接收channel上accpet事件

第二种.  NioEventLoopGroup接收channel上read write事件

第三种 . DefaultEventLoopGroup处理业务  



为什么还会有第三种？

因为在nio中，即使将accpet事件和 读写事件区别开来用不同的NioEventLoopGroup来管理，但如果一个线程接受到多个channel上的读写的事件，那他在其中一个读写事件处理的耗时长，就会影响到其他channel上读写事件的处理.所以将这种非读写的业务事件统一交给业务的DefaultEventLoopGroup来处理。提高整体的吞吐量



### slice

【零拷贝】的体现之一，对原始 ByteBuf 进行切片成多个 ByteBuf，切片后的 ByteBuf 并没有发生内存复制，还是使用原始 ByteBuf 的内存，切片后的 ByteBuf 维护独立的 read，write 指针



### 自定义编解码器

需要配置自定义协议，一般需要实现ByteToMessageCodec 类

在pipelin中 按照 LengthFieldBasedFrameDecoder->自定义的ByteToMessageCodec  类配置  的顺序，先处理半包黏包，后面是自定义协议解码器

半包黏包处理器的作用就是把消息处理完整后，才发给后面的handler，不然不会发送给后面的handler。

```
new LengthFieldBasedFrameDecoder(1024,12,4,0,0),
new MessageCodec());
```





### @Shareable

对黏包半包来说，每个channel 有自己的黏包半包handler处理器，不能共用.

attach方法是selectkey上的，selectkey是属于channel的，每一个channel对应一个黏包半包handler就行



如果不保存状态，handeler可以被共享



### 问题

netty中，为什么clinet的channel 没有调用syn方法，server端口也能接收数据

DefaultEventLoopGroup 和NioEventLoopGroup 区别？
