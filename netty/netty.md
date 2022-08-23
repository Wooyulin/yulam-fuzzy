# Netty怎么切换三种I/O模式

## 阻塞与非阻塞

描述的是调用方的行为，调用过程中，调用方线程是否被挂起，挂起的阻塞不能做其他事情

## 同步与异步

描述的是结果的获取，当结果一定要调用方主动获取，则为同步；结果为被调用方回调为异步

## 三种IO模式的支持

- BIO(阻塞)：不支持，连接数高的情况下耗费资源
- NIO(非阻塞)：支持
- AIO(异步)：linux下AIO支持不好，性能提升不明显



# Netty如何支持三种Reactor

| Reactor 单线程模式        | EventLoopGroup eventGroup = new NioEventLoopGroup(1);<br/>ServerBootstrap serverBootstrap = new ServerBootstrap();<br/>serverBootstrap.group(eventGroup); |
| ------------------------- | ------------------------------------------------------------ |
| 非主从 Reactor 多线程模式 | EventLoopGroup eventGroup = new NioEventLoopGroup();<br/>ServerBootstrap serverBootstrap = new ServerBootstrap();<br/>serverBootstrap.group(eventGroup); |
| 主从 Reactor 多线程模式   | EventLoopGroup bossGroup = new NioEventLoopGroup();<br/>EventLoopGroup workerGroup = new NioEventLoopGroup(); <br/>ServerBootstrap serverBootstrap = new ServerBootstrap();<br/>serverBootstrap.group(bossGroup, workerGroup); |



# TCP粘包/半包Netty全搞定

## 粘包

- 发送方每次写入数据< socket缓冲区大小
- 接收方读取缓冲区数据不够及时

## 半包

![image-20220512194434365](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220512194434365.png)

出现粘包半包根本原因是消息无边界

## 解决

### 常用方案对比

![image-20220512194758319](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220512194758319.png)

### Netty的支持

![image-20220512194920421](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220512194920421.png)

### Netty粘包半包源码赏析TODO

# 常用的二次编解码方式

## 一次编解码

一次解码用于解决粘包半包问题

## 二次编解码

用数据的高效转换

### 常见的方式

- Java序列化
- `xml`
- `json`
- `protobuf`



# `keepalive`与idle监测

## idle空闲监测

对方无应答的等待时间控制

## `keepalive`

结合idle 与发起`keepalive`请求

# Netty的锁

## netty锁的关键点

- 减少粒度
- 减少空间占用
- 提高速度
- 因需而变(选择不同的并发类)
- 能不用则不用



# Netty玩转内存

## 减少对象本身大小

能用基本类型则不用包装类

## 对分配内存进行预估

- `hashmap`初始化容量，避免扩容

## 零拷贝

- 使用逻辑组合，避免实际复制
- 包装字节，代替实际复制 `wrappedBuffer(bytes[])`
- 调用JDK的`transferTo`方法实 

## 堆外内存

## 内存池

### 为什么需要内存池

- 创建对象开销大
- 对象高频率创建且可复用
- 支持并发又能保护系统
- 维护、共享有限的资源

#### 如何实现

- apache commons pool
- Netty 轻量级对象池实现 io.netty.util.Recycler



