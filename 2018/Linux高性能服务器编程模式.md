# Linux高性能服务器编程模式

> 本文时间：2018-11-21，作者：[krircc](https://krircc.github.io/)， 简介：天青色  

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

高性能服务器至少要满足如下几个需求：

- 效率高：既然是高性能，那处理客户端请求的效率当然要很高了
- 高可用：不能随便就挂掉了
- 编程简单：基于此服务器进行业务开发需要足够简单
- 可扩展：可方便的扩展功能
- 可伸缩：可简单的通过部署的方式进行容量的伸缩，也就是服务需要无状态

而满足如上需求的一个基础就是高性能的IO!

讲到高性能IO绕不开Reactor模式，它是大多数IO相关组件如Netty、Redis在使用的IO模式

几乎所有的网络连接都会经过读请求内容——》解码——》计算处理——》编码回复——》回复的过程

## Socket

Socket之间建立链接及通信的过程！实际上就是对TCP/IP连接与通信过程的抽象:

- 服务端Socket会bind到指定的端口上，Listen客户端的"插入"
- 客户端Socket会Connect到服务端
- 当服务端Accept到客户端连接后
- 就可以进行发送与接收消息了
- 通信完成后即可Close

## 阻塞IO(BIO)、非阻塞IO(NBIO)、同步IO、异步IO

- 一个IO操作其实分成了两个步骤：发起IO请求和实际的IO操作
- 阻塞IO和非阻塞IO的区别在于第一步：发起IO请求是否会被阻塞，如果阻塞直到完成那么就是传统的阻塞IO;如果不阻塞，那么就是非阻塞IO
- 同步IO和异步IO的区别就在于第二个步骤是否阻塞，如果实际的IO读写阻塞请求进程，那么就是同步IO，因此阻塞IO、非阻塞IO、IO复用、信号驱动IO都是同步IO;如果不阻塞，而是操作系统帮你做完IO操作再将结果返回给你，那么就是异步IO

BIO优点

- 模型简单
- 编码简单

BIO缺点

- 性能瓶颈低

缺点：主要瓶颈在线程上。每个连接都会建立一个线程。虽然线程消耗比进程小，但是一台机器实际上能建立的有效线程有限，且随着线程数量的增加，CPU切换线程上下文的消耗也随之增加，在高过某个阀值后，继续增加线程，性能不增反降！而同样因为一个连接就新建一个线程，所以编码模型很简单！

就性能瓶颈这一点，就确定了BIO并不适合进行高性能服务器的开发！

NBIO:

- Acceptor注册Selector，监听accept事件
- 当客户端连接后，触发accept事件
- 服务器构建对应的Channel，并在其上注册Selector，监听读写事件
- 当发生读写事件后，进行相应的读写处理

优点

- 性能瓶颈高

缺点

- 模型复杂
- 编码复杂
- 需处理半包问题

NBIO的优缺点和BIO就完全相反了!性能高，不用一个连接就建一个线程，可以一个线程处理所有的连接！相应的，编码就复杂很多，从上面的代码就可以明显体会到了。还有一个问题，由于是非阻塞的，应用无法知道什么时候消息读完了，就存在了半包问题！需要自行进行处理!例如，以换行符作为判断依据，或者定长消息发生，或者自定义协议！

NBIO虽然性能高，但是编码复杂，且需要处理半包问题！为了方便的进行NIO开发，就有了Reactor模型!

## Proactor和Reactor

Proactor和Reactor是两种经典的多路复用I/O模型，主要用于在高并发、高吞吐量的环境中进行I/O处理。

I/O多路复用机制都依赖于一个事件分发器，事件分离器把接收到的客户事件分发到不同的事件处理器中，如下

![event](../../../static/imgs/blog/2018/linux-server-programming/event.png)

## select，poll，epoll

在操作系统级别select，poll，epoll是3个常用的I/O多路复用机制，简单了解一下将有助于我们理解Proactor和Reactor。

### select

select的原理如下：

![select](../../../static/imgs/blog/2018/linux-server-programming/select.png)

用户程序发起读操作后，将阻塞查询读数据是否可用，直到内核准备好数据后，用户程序才会真正的读取数据。

poll与select的原理相似，用户程序都要阻塞查询事件是否就绪，但poll没有最大文件描述符的限制。

### epoll

epoll是select和poll的改进，原理图如下：

![epoll](../../../static/imgs/blog/2018/linux-server-programming/epoll.png)

epoll使用“事件”的方式通知用户程序数据就绪，并且使用内存拷贝的方式使用户程序直接读取内核准备好的数据，不用再读取数据

## Proactor

Proactor是一个异步I/O的多路复用模型，原理图如下：

![proactor](../../../static/imgs/blog/2018/linux-server-programming/proactor.png)

- 用户发起IO操作到事件分离器
- 事件分离器通知操作系统进行IO操作
- 操作系统将数据存放到数据缓存区
- 操作系统通知分发器IO完成
- 分离器将事件分发至相应的事件处理器
- 事件处理器直接读取数据缓存区内的数据进行处理

## Reactor

Reactor是一个同步的I/O多路复用模型，它没有Proactor模式那么复杂，原理图如下：

![reactor](../../../static/imgs/blog/2018/linux-server-programming/reactor.png)

- 用户发起IO操作到事件分离器
- 事件分离器调用相应的处理器处理事件
- 事件处理完成，事件分离器获得控制权，继续相应处理

## Proactor和Reactor的比较

- Reactor模型简单，Proactor复杂
- Reactor是同步处理方式，Proactor是异步处理方式
- Proactor的IO事件依赖操作系统，操作系统须支持异步IO
- 同步与异步是相对于服务端与IO事件来说的，Proactor通过操作系统异步来完成IO操作，当IO完成后通知事件分离器，而Reactor需要自己完成IO操作

## Reactor多线程模型

前面已经简单介绍了Proactor和Reactor模型，在实际中Proactor由于需要操作系统的支持，实现的案例不多，有兴趣的可以看一下Boost Asio的实现，我们主要说一下Reactor模型，Netty也是使用Reactor实现的。

但单线程的Reactor模型每一个用户事件都在一个线程中执行：

- 性能有极限，不能处理成百上千的事件
- 当负荷达到一定程度时，性能将会下降
- 单某一个事件处理器发送故障，不能继续处理其他事件

## 多线程Reactor

使用线程池的技术来处理I/O操作，原理图如下：

![muti-thread](../../../static/imgs/blog/2018/linux-server-programming/muti-thread.png)

- Acceptor专门用来监听接收客户端的请求
- I/O读写操作由线程池进行负责
- 每个线程可以同时处理几个链路请求，但一个链路请求只能在一个线程中进行处理

## 主从多线程Reactor

在多线程Reactor中只有一个Acceptor，如果出现登录、认证等耗性能的操作，这时就会有单点性能问题，因此产生了主从Reactor多线程模型，原理如下：

![master-worker](../../../static/imgs/blog/2018/linux-server-programming/master-worker.png)

- Acceptor不再是一个单独的NIO线程，而是一个独立的NIO线程池
- Acceptor处理完后，将事件注册到IO线程池的某个线程上
- IO线程继续完成后续的IO操作
- Acceptor仅仅完成登录、握手和安全认证等操作，IO操作和业务处理依然在后面的从线程中完成

## Reactor模式结构

在解决了什么是Reactor模式后，我们来看看Reactor模式是由什么模块构成。图是一种比较简洁形象的表现方式，因而先上一张图来表达各个模块的名称和他们之间的关系：

![Reactor_Structures](../../../static/imgs/blog/2018/linux-server-programming/Reactor_Structures.png)

- Handle：即操作系统中的句柄，是对资源在操作系统层面上的一种抽象，它可以是打开的文件、一个连接(Socket)、Timer等。由于Reactor模式一般使用在网络编程中，因而这里一般指Socket Handle，即一个网络连接（Connection，在Java NIO中的Channel）。这个Channel注册到Synchronous Event Demultiplexer中，以监听Handle中发生的事件，对ServerSocketChannnel可以是CONNECT事件，对SocketChannel可以是READ、WRITE、CLOSE事件等。
  
- Synchronous Event Demultiplexer：阻塞等待一系列的Handle中的事件到来，如果阻塞等待返回，即表示在返回的Handle中可以不阻塞的执行返回的事件类型。这个模块一般使用操作系统的select来实现。在Java NIO中用Selector来封装，当Selector.select()返回时，可以调用Selector的selectedKeys()方法获取`Set<SelectionKey>`，一个SelectionKey表达一个有事件发生的Channel以及该Channel上的事件类型。上图的“Synchronous Event Demultiplexer ---notifies--> Handle”的流程如果是对的，那内部实现应该是select()方法在事件到来后会先设置Handle的状态，然后返回。不了解内部实现机制，因而保留原图。

- Initiation Dispatcher：用于管理Event Handler，即EventHandler的容器，用以注册、移除EventHandler等；另外，它还作为Reactor模式的入口调用Synchronous Event Demultiplexer的select方法以阻塞等待事件返回，当阻塞等待返回时，根据事件发生的Handle将其分发给对应的Event Handler处理，即回调EventHandler中的handle_event()方法。

- Event Handler：定义事件处理方法：handle_event()，以供InitiationDispatcher回调使用。

- Concrete Event Handler：事件EventHandler接口，实现特定事件处理逻辑。

### 优点

1）响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；

2）编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；

3）可扩展性，可以方便的通过增加Reactor实例个数来充分利用CPU资源；

4）可复用性，reactor框架本身与具体事件处理逻辑无关，具有很高的复用性；

### 缺点

1）相比传统的简单模型，Reactor增加了一定的复杂性，因而有一定的门槛，并且不易于调试。

2）Reactor模式需要底层的Synchronous Event Demultiplexer支持，比如Java中的Selector支持，操作系统的select系统调用支持，如果要自己实现Synchronous Event Demultiplexer可能不会有那么高效。

3） Reactor模式在IO读写数据时还是在同一个线程中实现的，即使使用多个Reactor机制的情况下，那些共享一个Reactor的Channel如果出现一个长时间的数据读写，会影响这个Reactor中其他Channel的相应时间，比如在大文件传输时，IO操作就会影响其他Client的相应时间，因而对这种操作，使用传统的Thread-Per-Connection或许是一个更好的选择，或则此时使用Proactor模式。

## Reactor中的组件

- Reactor:Reactor是IO事件的派发者。
- Acceptor:Acceptor接受client连接，建立对应client的Handler，并向Reactor注册此Handler。
- Handler:和一个client通讯的实体，按这样的过程实现业务的处理。一般在基本的Handler基础上还会有更进一步的层次划分， 用来抽象诸如decode，process和encoder这些过程。比如对Web Server而言，decode通常是HTTP请求的解析， process的过程会进一步涉及到Listener和Servlet的调用。业务逻辑的处理在Reactor模式里被分散的IO事件所打破， 所以Handler需要有适当的机制在所需的信息还不全（读到一半）的时候保存上下文，并在下一次IO事件到来的时候（另一半可读了）能继续中断的处理。为了简化设计，Handler通常被设计成状态机，按GoF的state pattern来实现。

## Rust异步网络编程

Rust的高性能异步网络编程模式目前是基于[`mio`](https://github.com/carllerche/mio)和[`futures`](https://github.com/rust-lang-nursery/futures-rs)这两个库构建的生态。

[Tokio](https://github.com/tokio-rs/tokio)则连接这2个库构建了一个异步非阻塞事件驱动编程平台。

## 什么是 `mio`,`futures`,`tokio`

### 1- Mio

Mio是Rust的轻量级快速低级IO库，专注于非阻塞API,事件通知以及用于构建高性能IO应用程序的其他有用实用程序.

#### 特征

* 快速 - 相当于OS设施级别的最小开销（epoll，kqueue等..）
* 非阻塞TCP，UDP。
* 由epoll，kqueue和IOCP支持的I/O事件通知队列。
* 运行时零分配
* 平台特定扩展。

#### 平台支持

* Linux
* OS X
* Windows
* FreeBSD
* NetBSD
* Solaris
* Android
* iOS
* Fuchsia (experimental)

## 2- futures

Rust中的零成本异步编程库,Futures可在没有标准库的情况下工作，例如在裸机环境中。

提供了许多用于编写异步代码的核心抽象：

* `Future`是由异步计算产生的单一最终值。一些编程语言（例如JavaScript）将此概念称为“promise”。
* `Streams`表示异步生成的一系列值。
* `Sinks`支持异步写入数据。
* `Executors`负责运行异步任务。

还包含异步I/O和跨任务通信的抽象。

所有这些是任务系统的基础，它是轻量级线程(**协程**)的一种形式。使用`Future`，`Streams`和`Sinks`构建大型异步计算，然后将其生成作为独立完成的任务运行，但不阻塞运行它们的线程。

## 3- Tokio

Tokio : Rust编程语言的异步运行时,提供异步事件驱动平台，构建快速，可靠和轻量级网络应用。利用Rust的所有权和并发模型确保线程安全

* 基于多线程，工作窃取的任务调度程序。
* 一个反应器操基于作系统的事件队列（epoll的，kqueue的，IOCP等）的支持。
* 异步TCP和UDP套接字。

这些组件提供构建异步应用程序所需的运行时组件。

## 快速

Tokio构建于Rust之上，提供极快的性能，使其成为高性能服务器应用程序的理想选择。

1:零成本抽象

与完全手工编写的等效系统相比，Tokio的运行时模型不会增加任何开销。

使用Tokio构建的并发应用程序是开箱即用的。Tokio提供了针对异步网络工作负载调整的多线程，工作窃取任务调度程序。

2:非阻塞I/O

Tokio由操作系统提供的非阻塞，事件I/O堆栈提供支持。

## 可靠

虽然Tokio无法阻止所有错误，但它的目的是最小化它们。Tokio在运送关键任务应用程序时带来了安心。

1- 所有权和类型系统

Tokio利用Rust的类型系统来提供难以滥用的API。

2- Backpressure

Backpressure开箱即用，无需使用任何复杂的API。

3- 取消

Rust的所有权模型允许Tokio自动检测何时不再需要计算。Tokio将自动取消它而无需用户调用cancel函数。

## 轻量级

Tokio可以很好地扩展，而不会增加应用程序的开销，使其能够在资源受限的环境中茁壮成长。

1- 没有垃圾收集器

因为Tokio使用Rust，所以不包括垃圾收集器或其他语言运行时。

2- 模块化

Tokio是一个小组件的集合。用户可以选择最适合手头应用的部件，而无需支付未使用功能的成本。