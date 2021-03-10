## Reactor线程模型

> The Reactor design pattern handles service requests that are delivered concurrently to an application by one or more clients

![logging-server-example](../static/netty/logging-server-example.png)

`Reactor`本质是一种设计模式，用来解决分布式环境下高并发问题。上图是论文中的例子，每个客户端都和服务器保持一个Socket链接，当客户端产生事件时又需要对应找到后方的数据据或是打印机进行输出。那么在`Reactor`这种模型之前，可以考虑的实现方式只有`TPR(Thread-per-request)`，显而易见每个请求一个BIO线程的方式不可取。

而网络请求多半可以视为一次读写事件，一个分发者接收到事件后将事件分发给真正的服务提供者，中间的`LoggingServer`就是`Reactor`的核心思想，也就是**事件分发**或者说**I/O多路复用**。

**这是一种将多个连接挂载到一个处理者名下，仅由一个线程维护连接，而处理发生在连接传来的事件被分发后（分发后将是另外的线程进行处理）。**而这种处理方式需要保证一点，服务器不可以因为任何一个请求的阻塞而无法处理其他请求，实际上这点也就是提出了**非阻塞I/O**的需要。



### 角色

那么设计过后提出的`Reactor模式`中包含如下角色：

- `Handle(句柄)`：标识系统资源
- `SynchronousEventDemultiplexer`: 监听事件
- `InitiationDispatcher`：注册事件处理器，分发事件
- `EventHandler`：定义事件处理的接口
- `ConcreteEventHandler`：真正的事件处理者

UML类图如下：

![reactor-uml](../static/netty/reactor-uml.png)

结合`LoggingServer`的例子，可以简单描述一下其工作过程: 

1. 应用向`InitiationDispatcher`注册一个真正的`Handler`，同时提供给`Dispatcher`它感兴趣的事件，以及监控事件的句柄`Handle`。
2. 全部注册完成后调用`handle_envets()`，开始事件循环，其中通过`SynchronousEventDemultiplexer`同步监听句柄中的事件，具体的监听方式取决于系统的实现（论文中给出的是`select()`）
3. `SynchronousEventDemultiplexer`根据监听到的事件通知`Dispatcher`，后者将根据注册信息将事件分发到具体的`handler`中执行

![dispatcher-proces](../static/netty/dispatcher-process.png)

### 优缺点

实际上在讲到依赖底层的NIO时就明白，`Reactor模式`是一种和底层I/O实现联系相当紧密的设计模式，它在I/O的处理过程中添加了多种角色用于解耦，同时定义了`Handler`来模块化提供服务的处理者，**使用一个线程监听事件的方式增加了同时可处理的连接上限**，`Dispatcher`的存在十分类似一层负载均衡，**它通过解放被阻塞的线程以提高CPU的利用效率**。

关于提升CPU效率我是这么理解的：传统高并发场景下的BIO将迫使服务器维护多个阻塞等待资源的线程，而这些线程因为等待资源无法利用CPU，CPU发现线程阻塞后将唤醒其他线程进行CPU计算，但是此时大多数线程都是阻塞状态，将出现内核不断唤醒线程尝试运行，上下文不断切换但却没有实际利用CPU的情况（有巨大频繁的上下文切换代价，且CPU低负载）。那么仅由一个线程阻塞监听连接事件后，CPU就可以被其他线程或者程序利用。

同样的[SEDA: Staged Event-Driven Architecture - An Architecture for Well-Conditioned, Scalable Internet Service](https://people.eecs.berkeley.edu/~brewer/papers/SEDA-sosp.pdf)中验证了系统中线程数量在增长时，系统的吞吐量迅速下降，减少系统中的线程数，是可以提升系统性能的。

但这种设计方式显然提升了系统的复杂性，在非高并发场景下会不如BIO来的有效，同时NIO的非阻塞阶段仅包括资源的等待阶段，读写文件时仍然是阻塞的，因为这个原因多个公用同一线程进行I/O的client可能被彼此阻塞。

### 升级模型

#### Reactor多线程模型

实际上这是考虑到上述模型中的`Dispatcher`是单线程的，也就是多个请求实际仍然会通过一个线程进行IO，会导致如下问题：

- NIO线程同时处理过多的线程则在机器性能上无法满足
- 如果负载过重，将拖累处理速度，吞吐量的下降将导致更多的消息挤压与重发，最终拖垮系统
- 单个NIO的过程中可能出现意外或者阻塞，影响其他所有的客户端，存在单点故障得问题

多线程的设计方式就是分离出了一个线程池处理NIO操作

![Reactor多线程](../static/netty/Reactor多线程.png)

这样的模式下，`Reactor Thread`将仅进行连接请求的处理（`Acceptor`）以及请求的分发，NIO的处理将在线程池中进行。共使用了一个`Reactor Thread` 线程负责连接请求的处理以及请求分发，NIO线程池负责处理网络IO。

#### 主从模型

![main-sub-reactor](../static/netty/main-sub-reactor.png)

对上述的模型仍然有性能的瓶颈，`Reactor Thread`处理连接同时进行转发请求，如果连接协议要求安全认证可能导致`Acceptor`的性能不足，从而对后续链路的处理产生延迟。

Netty中真正使用的是Reactor主从模型，这种模型同样使用在像`Nginx`等技术中。相比之下，主从模型将连接的处理也使用一个线程池进行处理。解耦之后各个角色的功能如下：

- `Acceptor`：类似一个转发者，仅将请求转发给`Main Reactor`
- `Main Reactor`：在TCP握手后，处理`Acceptor`发来的连接事件，创建`NioSocketChannel`对象并注册到`Sub Reactor`中
- `Sub Reactor`：线程池监听`Channel`中的事件，调用处理链，并真正处理网络I/O

如下图：

![main-reactor](../static/netty/main-reactor.png)

这里可以更加明确的体会到`Main Reactor`的意义所在，相当于在底层进行TCP连接之后，需要建立Netty层级的连接时，回到`Main Reactor`的线程池中进行创建，之后被注册到`Sub Reactor`中（提供连接对象以及关心事件）。

所以可以这么总结：**所有客户端连接时，将一次性的在`Main Reactor`线程池中进行连接，所有的事件监听以及网络IO在`Sub Reactor`中**。而Netty中的`Boss Group`和`Work Group`正好也对应`Main`和`Sub`。

模型中的线程可以总结如下：

| Acceptor     | Main Reactor                   | Sub Reactor                                                  | Bussiness Thread Pool |
| ------------ | ------------------------------ | ------------------------------------------------------------ | --------------------- |
| 单线程       | 线程池                         | 线程池                                                       | 线程池                |
| 转发连接请求 | 处理连接，创建NIOSocketChannel | 监听事件，处理IO，多个Channel对象将注册在一个线程上，这里一个线程相当于单线程版的Reactor | 业务逻辑处理          |
|              |                                |                                                              |                       |

> 关于多使用一个线程池处理连接请求对于性能的提升

NIO线程池和业务线程池本身是多线程的，足够充分利用CPU，但一个客户端的连接到业务处理的流程确实按时间有绝对先后顺序的，不可能一边连接一边处理业务一边IO，所以单线程的连接处理会导致服务器处理客户端连接时性能较低，即使后面的链路使用了多线程，仍然可能出现无米下锅的情况，这里的多线程就是确保在整个客户端在服务器的服务周期内都可以充分利用多核CPU的优势来提高并发量。但每一种线程模型中的Reactor都是**串行**处理IO的（多个Channel依次处理），对于大文件的处理不是NIO的强项。