<!-- TOC -->
- [一、JavaIO](#一JavaIO)
- [二、装饰者模式](#二装饰者模式)
- [三、编码与解码](#三编码与解码)
- [四、上下文切换](#四上下文切换)
- [五、NIO](#五NIO)
<!-- TOC -->


# 一、JavaIO

# 二、装饰者模式

# 三、编码与解码

# 四、上下文切换

# 五、NIO

首先，我们需要弄清楚几个概念：同步和异步，阻塞和非阻塞。

## 同步和异步

### 1. 同步 

进程触发 IO 操作的时候，必须亲自处理；
比如你必须亲自去银行取钱。

### 2. 异步

进程触发 IO 操作的时候，可以不亲自处理，它把操作委托给 OS 处理，委托的时候需要告知数据的地址和大小，然后自己去做别的事情，当 IO 操作结束后会得到通知；
比如你把银行卡给我，让我帮你去银行取钱，你需要告诉我银行卡密码和取多少钱，我取完了之后把钱给你。

### 3. 总结

**自己干就是同步，别人干就是异步。**

## 阻塞和非阻塞

### 1. 阻塞

进程触发 IO 操作的时候，如果此时此时没办法读或者写，那么进程就一直等待，直到读写结束；
比如你去银行 ATM 取钱，前面有人在排队，那么就要一直等待，直到你取完钱；

### 2. 非阻塞

进程触发 IO 操作的时候，如果此时此时没办法读或者写，那么就先去做别的，等到有通知后，再继续读写；
比如你去银行柜台取钱，人比较多，那就先领一个号，等着叫到号再去对应的窗口办理业务；这里稍微有些不太恰当的是，我们等待的过程中，还得听着叫号。

### 3. 总结

**我要等着不能做其他事就是阻塞，我不用等可以做其他事就是异步。**

## BIO

同步阻塞；一个请求过来，应用程序开了一个线程，等 IO 准备好，IO 操作也是自己干；
采用 BIO 模型的服务端，由一个独立的 Acceptor 线程负责进行监听；在 while(true) 循环中调用 accept() 方法，等待客户端的请求；
一旦接收到请求，就可以建立套接字开始进行读写操作，这时候不再接收其他的请求，直到读写完成；

为了让 BIO 能够同时处理多个请求，那么就需要使用多线程处理；当服务端接收到请求，就为客户端创建一个线程进行处理，处理完成后再做线程销毁；

![BIO 同步阻塞](https://github.com/CodeDaShu/JavaNotes/blob/master/img/NIO/BIO.jpg)

不过因为一个请求就要启动一个线程，所以开销是比较大的，启动和销毁线程开销很大，而且每个线程都要占用内存，所以可以引入线程池，可以在一定程度上减少线程创建和销毁的开销；这也被叫做 **伪异步 IO**。

![伪异步 IO](https://github.com/CodeDaShu/JavaNotes/blob/master/img/NIO/BIO-pseudo-asynchronous.jpg)

线程池维护着 N 个线程和一个消息队列；当有请求接入时，服务端将 Socket 作为参数传递到一个线程任务中进行处理；通过对线程池最大线程数和消息队列大小进行控制，所以就算访问量高于服务端的承载能力，也不会因为服务端的资源耗尽而导致宕机；

这个模型在 请求量不高的时候，效率还是不错的，而且也不需要考虑限流的问题（控制线程池的最大线程数量）。

### IO

同步非阻塞；不用等待 IO 准备，准备好了会通知，不过 IO 操作还是要自己干；NIO 是一种多路复用机制，利用单线程轮询事件，Channel 来决定做什么，避免连接数多的时候，频繁进行线程切换导致性能问题（Select 阶段阻塞）。

听到这里，很多人可能已经懵了...什么是多路复用？Channel 又是啥？Select 阶段到底是什么阶段？这里我用白话解释一下。
NIO 是面向缓冲区的，可以将数据读取到一个缓冲区，稍后进行处理。NIO 有几个核心概念：

![NIO](https://github.com/CodeDaShu/JavaNotes/blob/master/img/NIO/NIO.jpg)

### 1. Channel 和 Buffer 

Channel 可以理解成一个双向流，或者理解成一个通道，Buffer 就是缓存区，或者你就把它看做是一块内存空间，数据可以从 Channel 流进 Buffer ，也可以从 Buffer 流进 Channel 。
Channel 有很多种实现，比如：FileChannel 是从文件中读写数据，SocketChannel 通过 TCP 读写网络中的数据等等。
Buffer 也有多重类型，比如：ByteBuffer、CharBuffer、IntBuffer等等，光看他们的名字就知道他们代表了不同的数据类型。

### 2. Selector

我们可以把 Selector 看做是一个管理员，可以管理多个 Channel ， Selector 能够知道到哪个 Channel 已经做好了读写的准备。这样一个线程只要操作这个管理员就可以了，相当于一个线程可以管理多个 Channel；一旦监听到有准备好的 Channel，就可以进行相应的处理。
不过 Java 原生的 NIO 不好用，直到 Netty 的出现。

## AIO

异步非阻塞；因为事情不是自己做，其实也没有阻塞一说（都是非阻塞）；
AIO 是在 NIO 的基础上，引入异步通道的概念；NIO 是采用轮询的方式，不停地询问数据是否准备好了，准备好了就处理；AIO 是向操作系统注册 IO 监听，操作系统完成 IO 操作了之后，主动通知，触发响应的函数（自己不做，让操作系统来做）。

目前看，AIO 应该的还不是很广泛。
