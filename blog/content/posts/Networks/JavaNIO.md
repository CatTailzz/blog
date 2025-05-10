---
title: JavaNIO
subtitle: 
date: 2024-09-19T02:04:18+08:00
slug: 97dfdef
draft: true
tags:
  - 计算机网络
categories:
  - 计算机网络
password: 
message:
---
# NIO 三大组件

- Channel，负责数据读写的双通道
- Buffer，读写数据缓冲区
- Selector，事件驱动模型，用一个线程管理多个channel，监听就绪事件

## Buffer

### 基础结构

buffer 有几个重要属性，并且在不同模式下会切换

- capacity，即缓冲区的容量
- position，写模式下表示下一个写入位置，读模式下表示下一个读的位置
- limit，写模式下表示写入的限制位置，读模式下表示读取的限制位置

通过 flip() 来切换模式，另外这个 buffer 并不是循环的，想复用要么使用 clear() 清空，要么用具体实现类的 compact() 方法把剩余元素移动到开头，也可以 rewind() 单纯重置 position 为 0

关于挪动还有一点要说就是，挪动之前的数据位置，再挪动后不会置为 0 或空，而是保留原值，不过 limit 和 position 都变了，后续这个位置要么就是读不到，要么就是被覆写，倒也没问题。

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202409192117728.png)

关于这里的 mark，是提供给用户的一个标记位，调用 mark() 方法会给 mark 赋值为当前的position，再调用 reset() 则会把 position 重新置为 mark

### 空间分配

Buffer 在空间分配时指定一个容量，这个容量不可变。在分配方式上，分为两种

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202409192136652.png)

从名字能直观看出来，HeapByteBuffer 是 Java 堆内存分配，DirectByteBuffer 是直接内存分配。下面说说二者区别

- Heap 读写要慢点，因为多一次拷贝，这个是 IO 零拷贝相关的知识
- Heap 分配受 JVM 管理，可能被 GC，有好有坏，Direct 就需要自己管理，防止内存泄漏

# 网络编程

## 阻塞和非阻塞

- 阻塞模式下，调用一些 accept() 或者 read() 方法会导致线程停止运行，在单线程模式下更离谱，server 调用 accept() 卡住了，会导致这个线程里的 socket 也无法调用 read()，需要多线程支持。但即使上了多线程也有问题，一个线程一个连接，长久下去必然 OOM，还会上下文切换影响效率，上了线程池也是治标不治本。
- 非阻塞模式下，不会导致线程停止运行，而是快速返回 null 或者 0，但要想正常工作，必然要在代码层面做 while true 的处理，不断轮询，本质还是同步的，且浪费 CPU
- 多路复用，使用 Selector 实现对 Channel 可读写事件的监听，注意这里是网络IO下，而不是文件IO。只需要一个线程配合一个 Selector，就可以实现对多个 Channel 的监听。

## Selector

- 适用于网络IO，且注册的 channel 必须非阻塞
- 需要绑定 Channel 事件，具体有四种类型
	- connect - 客户端连接成功时触发
	- accept - 服务端成功接受连接时触发
	- read - 数据可读时触发
	- write - 数据可写时触发

select() 方法会阻塞，直到有某个 channel 就绪了，那么**何时不会阻塞**呢？

- 事件发生时
	- 发起连接请求，触发了 accept 事件，且这个事件没被处理
	- 发送数据，客户端正常、异常关闭，都会触发 read 事件
	- channel 可写时触发 write 事件
	- linux 下 nio bug 发生时
- 调用 selector.wakeup()
- 调用 selector.close()
- 当前线程被 interrupt

## 零拷贝

- 第一阶段

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202409211931416.png)

- 第二阶段

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202409211943180.png)

- 第三阶段

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202409211944831.png)

- 第四阶段

![image.png](https://obsidian-img-1300316500.cos.ap-shanghai.myqcloud.com/cattail/obsidian/pic/202409211944091.png)

## AIO

异步的一种常见处理是，主线程尝试读写操作，并把获取到数据后的操作封装成一个回调函数，接着不管是否有数据准备好，都直接返回进行其他操作。而当数据真的准备好时，会有另一个线程来执行回调函数做后续操作。异步的关键就是，有另一个线程来做处理，不阻塞主线程。