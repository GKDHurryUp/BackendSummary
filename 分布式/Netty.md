## 原生NIO存在的问题 ##
1. NIO 的类库和 API 繁杂，使用麻烦：需要熟练掌握 Selector、ServerSocketChannel、SocketChannel、ByteBuffer 等。
2. 需要具备其他的额外技能：要熟悉 Java 多线程编程，因为 NIO 编程涉及到 Reactor 模式，你必须对多线程和网络编程非常熟悉，才能编写出高质量的 NIO 程序。
3. 开发工作量和难度都非常大：例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常流的处理等等
4. JDK NIO 的 Bug：例如臭名昭著的 Epoll Bug，它会导致 Selector 空轮询，最终导致 CPU 100%。直到 JDK 1.7 版本该问题仍旧存在，没有被根本解决。

## Netty ##

介绍：
1. 由JBOSS提供的一个Java开源框架，现为GitHub上的独立项目
2. Netty是一个异步的、基于事件驱动的网络应用框架，用以快速开发高性能、高可靠的网络IO程序
3. Netty主要针对在TCP协议下
4. Netty本质为一个NIO框架

应用场景：
1. 互联网行业
	作为基础的通信组件被RPC框架使用，如Dubbo使用Dubbo协议进行节点间的通信，使用的就是Netty作为基础通信组件，用于实现各进程节点之间的内部通信
2. 游戏行业	
	Netty提供TCP/UDP和HTTP协议栈，方便定制和开发私有协议栈，帐号登陆服务器	
	地图服务器之间可以使用Netty进行通信
3. 大数据领域
	经典的Hadoop的高性能通信和序列化组件（AVRO实现数据文件共享）的RPC框架，默认使用Netty进行跨节点通信
	它的Netty Service基于Netty框架二次封装实现

优点：
1. 设计优雅
2. 使用方便，详细记录的 Javadoc
3. 没有其他依赖项，JDK 5（Netty 3.x）或 6（Netty 4.x）
4. 高性能、吞吐量更高：延迟更低；减少资源消耗；最小化不必要的内存复制
5. 安全：完整的 SSL/TLS 和 StartTLS 支持

## 线程模型 ##
介绍：
1. 传统阻塞I/O服务模型
2. Reactor模型
	- 单Reactor单线程
	- 单Reactor多线程
	- 主从Reactor多线程
3. Netty线程模式（基于主从Reactor多线程模型改进，主从Reactor多线程模型有多个Reactor）

## Reactor模式 ##
1. 基于 I/O 复用模型：多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理
2. 基于线程池复用线程资源：不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务。

## 单 Reactor 单线程 ##

![](..\imgs\单Reactor单线程.png)

服务器端用一个线程通过多路复用搞定所有的 IO 操作（包括连接，读、写等），编码简单，清晰明了，但是如果客户端连接数量较多，将无法支撑

1. 优点：模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成
2. 缺点：
	1. 性能问题，只有一个线程，无法完全发挥多核 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈
	1. 可靠性问题，线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障
3. 使用场景：客户端的数量有限，业务处理非常快速，比如 Redis在业务处理的时间复杂度 O(1) 的情况

## 单Reactor多线程 ##

![](..\imgs\单Reactor多线程.png)

1. Reactor 对象通过select 监控客户端请求事件, 收到事件后，通过dispatch进行分发
2. 如果建立连接请求, 则右Acceptor 通过accept 处理连接请求, 然后创建一个Handler对象处理完成连接后的各种事件
3. 如果不是连接请求，则由reactor分发调用连接对应的handler 来处理
3. **handler 只负责响应事件，不做具体的业务处理, 通过read 读取数据后，会分发给后面的worker线程池的某个线程处理业务**
2. worker 线程池会分配独立线程完成真正的业务，并将结果返回给handler
3. handler收到响应后，通过send 将结果返回给client

优点：可以充分的利用多核cpu 的处理能力
缺点：多线程数据共享和访问比较复杂， reactor 处理所有的事件的监听和响应，在单线程运行， 在高并发场景容易出现性能瓶颈.

## 主从 Reactor 多线程 ##

![](..\imgs\主从Reactor多线程.png)

1. Reactor主线程 MainReactor 对象通过select 监听连接事件, 收到事件后，通过Acceptor 处理连接事件
2. **当 Acceptor 处理连接事件后，MainReactor 将连接分配给SubReactor** 
3. subreactor 将连接加入到连接队列进行监听,并创建handler进行各种事件处理
4. 当有新事件发生时， subreactor 就会调用对应的handler处理
5. handler 通过read 读取数据，分发给后面的worker 线程处理
6. worker 线程池分配独立的worker 线程进行业务处理，并返回结果
7. handler 收到响应的结果后，再通过send 将结果返回给client
8. **Reactor 主线程可以对应多个Reactor 子线程, 即MainRecator 可以关联多个SubReactor**

优点：
1. 父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。
2. 父线程与子线程的数据交互简单，Reactor 主线程只需要把新连接传给子线程，子线程无需返回数据。

缺点：编程复杂度较高

应用场景：
Nginx 主从 Reactor 多进程模型，Memcached 主从多线程，Netty 主从多线程模型的支持

## Netty模型-简单版 ##
1. BossGroup 线程维护Selector , 只关注Accecpt
2. 当接收到Accept事件，获取到对应的SocketChannel, 封装成 NIOScoketChannel并注册到Worker 线程(事件循环), 并进行维护
3. 当Worker线程监听到selector 中通道发生自己感兴趣的事件后，就进行处理(就由handler)， 注意handler 已经加入到通道

## Netty模型-详细版 ##

![](..\imgs\Netty模型.jpg)

1. Netty抽象出两组线程池 BossGroup 专门负责接收客户端的连接, WorkerGroup 专门负责网络的读
2. BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup
3. NioEventLoopGroup 相当于一个事件循环组, 这个组中含有多个事件循环 ，每一个事件循环是 NioEventLoop
4. NioEventLoop 表示一个不断循环的执行处理任务的线程， 每个NioEventLoop 都有一个selector , 用于监听绑定在其上的socket的网络通讯
5. NioEventLoopGroup 可以有多个线程, 即可以含有多个NioEventLoop
6. 每个Boss NioEventLoop 循环执行的步骤有3步
	1. 轮询accept 事件
	2. 处理accept 事件 , 与client建立连接 , 生成NioScocketChannel , 并将其注册到某个workerNIOEventLoop 上的 selector 
	3. 处理任务队列的任务 ， 即 runAllTasks
7. 每个 Worker NIOEventLoop 循环执行的步骤
	1. 轮询read, write 事件
	2. 处理i/o事件， 即read , write 事件，在对应NioScocketChannel 处理
	3. 处理任务队列的任务 ， 即 runAllTasks
8.  每个Worker NIOEventLoop  处理业务时，会使用pipeline(管道), pipeline 中包含了 channel , 即通过pipeline 可以获取到对应通道, 管道中维护了很多的 处理器

## 异步模型 ##
1. 异步的概念和同步相对。当一个异步过程调用发出后，调用者不能立刻得到结果。实际处理这个调用的组件在完成后，通过状态、通知和回调来通知调用者。
2. Netty 中的 I/O 操作是异步的，包括 Bind、Write、Connect 等操作会简单的返回一个 ChannelFuture。
3. 调用者并不能立刻获得结果，而是通过 Future-Listener 机制，用户可以方便的主动获取或者通过通知机制获得 IO 操作结果
4. Netty 的异步模型是建立在 future 和 callback 的之上的。

Future 说明
1. 表示异步的执行结果, 可以通过它提供的方法来检测执行是否完成，比如检索计算等等.
2. ChannelFuture 是一个接口 ：

		public interface ChannelFuture extends Future<Void>
	我们可以添加监听器，当监听的事件发生时，就会通知到监听器.

Future-Listener 机制

1. 当 Future 对象刚刚创建时，处于非完成状态，调用者可以通过返回的 ChannelFuture 来获取操作执行的状态，注册监听函数来执行完成后的操作。
2. 常见有如下操作
	- 通过 isDone 方法来判断当前操作是否完成；
	- 通过 isSuccess 方法来判断已完成的当前操作是否成功；
	- 通过 getCause 方法来获取已完成的当前操作失败的原因；
	- 通过 isCancelled 方法来判断已完成的当前操作是否被取消；
	- 通过 addListener 方法来注册监听器，当操作已完成(isDone 方法返回完成)，将会通知指定的监听器；如果 Future 对象已完成，则通知指定的监听器

## HTTP服实例 ##
1. Netty 服务器在 8888 端口监听，浏览器发出请求 "http://localhost:8888/" 
2. 服务器可以回复消息给客户端 "Hello! 我是服务器 5 " ,  并对特定请求资源进行过滤.

# 核心组件 #
## Bootstrap、ServerBootstrap ##
Bootstrap是引导，一个Netty应用通常由一个Bootstrap开始，主要**作用是配置整个Netty程序，串联各个组件**。
Netty 中 Bootstrap 类是客户端程序的启动引导类，ServerBootstrap 是服务端启动引导类

常见的方法：
1. public ServerBootstrap **group**(EventLoopGroup parentGroup, EventLoopGroup childGroup)，该方法用于服务器端，用来设置两个 EventLoopGroup
   public B **group**(EventLoopGroup group) ，该方法用于客户端，用来设置一个 EventLoopGroup
3. public B **channel**(Class<? extends C> channelClass)，该方法用来设置一个服务器端的通道实现
4. public <T> B **option**(ChannelOption<T> option, T value)，用来给 ServerChannel 添加配置
5. public <T> ServerBootstrap **childOption**(ChannelOption<T> childOption, T value)，用来给接收到的通道添加配置
6. public ServerBootstrap **childHandler**(ChannelHandler childHandler)，该方法用来设置业务处理类（自定义的 handler）
7. public ChannelFuture **bind**(int inetPort) ，该方法用于服务器端，用来设置占用的端口号
public ChannelFuture **connect**(String inetHost, int inetPort) ，该方法用于客户端，用来连接服务器端

## Future、ChannelFuture ##
Netty 中所有的 IO 操作都是异步的，不能立刻得知消息是否被正确处理。但是可以过一会等它执行完成或者直接注册一个监听，具体的实现就是通过 Future 和 ChannelFutures，他们可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件

常见的方法：
1. Channel channel()，返回当前正在进行 IO 操作的通道
2. ChannelFuture sync()，等待异步操作执行完毕，调用的sync()的目的就是保证ChannelFuture已经完成了。

## Channel ##
Netty 网络通信的组件，能够用于执行网络 I/O 操作。

1. 通过Channel 可获得当前网络连接的通道的状态
2. 通过Channel 可获得 网络连接的配置参数 （例如接收缓冲区大小）
3. Channel 提供异步的网络 I/O 操作(如建立连接，读写，绑定端口)，异步调用意味着任何 I/O 调用都将立即返回，并且不保证在调用结束时所请求的 I/O 操作已完成
4. 调用立即返回一个 ChannelFuture 实例，通过注册监听器到 ChannelFuture 上，可以 I/O 操作成功、失败或取消时回调通知调用方

## Selector ##
1. Netty 基于 Selector 对象实现 I/O 多路复用，通过 Selector 一个线程可以监听多个连接的 Channel 事件。
2. 当向一个 Selector 中注册 Channel 后，Selector 内部的机制就可以自动不断地查询(Select) 这些注册的 Channel 是否有已就绪的 I/O 事件（例如可读，可写，网络连接完成等），这样程序就可以很简单地使用一个线程高效地管理多个 Channel 

## ChannelHandler ##

ChannelHandler 是一个接口，处理 I/O 事件或拦截 I/O 操作，并将其转发到其 ChannelPipeline(业务处理链)中的下一个处理程序。
ChannelHandler 本身并没有提供很多方法，因为这个接口有许多的方法需要实现，方便使用期间，可以继承它的子类

## Pipleine和ChannelPipeline ##

ChannelPipeline 是一个 Handler 的集合，它负责处理和拦截 inbound 或者 outbound 的事件和操作，相当于一个贯穿 Netty 的链。（ChannelPipeline 是 保存 ChannelHandler 的 List，用于处理或拦截 Channel 的入站事件和出站操作）
ChannelPipeline 实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各个的 ChannelHandler 如何相互交互

在 Netty 中每个 Channel 都有且仅有一个 ChannelPipeline 与之对应
- 一个 Channel 包含了一个 ChannelPipeline，而 ChannelPipeline 中又维护了一个由 ChannelHandlerContext 组成的双向链表，并且每个 ChannelHandlerContext 中又关联着一个 ChannelHandler

- 入站事件和出站事件在一个双向链表中，入站事件会从链表 head 往后传递到最后一个入站的 handler，出站事件会从链表 tail 往前传递到最前一个出站的 handler，两种类型的 handler 互不干扰

## ChannelHandlerContext ##

1. 保存 Channel 相关的所有上下文信息，同时关联一个 ChannelHandler 对象
2. ChannelHandlerContext 中 包 含 一 个 具 体 的 事 件 处 理 器 ChannelHandler ， 同 时ChannelHandlerContext 中也绑定了对应的 pipeline 和 Channel 的信息，方便对 ChannelHandler进行调用.

## Unpooled  ##

Netty 提供一个专门用来操作缓冲区(即Netty的数据容器)的工具类，可以获得ByteBuf

	public static ByteBuf copiedBuffer(CharSequence string, Charset charset)

ByteBuf中有两个指针：readerIndex和writerIndex
0-readerIndex表示不可读，
readerIndex-writerIndex表示可读，
writerIndex-capacity表示可写

## Netty应用实例-群聊系统 ##
1. 编写一个 Netty 群聊系统，实现服务器端和客户端之间的数据简单通讯（非阻塞）
2. 实现多人群聊
3. 服务器端：可以监测用户上线，离线，并实现消息转发功能
4. 客户端：通过channel 可以无阻塞发送消息给其它所有用户，同时可以接受其它用户发送的消息(有服务器转发得到)
5. 目的：进一步理解Netty非阻塞网络编程机制

## Netty心跳检测机制案例 ##
1. 编写一个 Netty心跳检测机制案例, 当服务器超过3秒没有读时，就提示读空闲
2. 当服务器超过5秒没有写操作时，就提示写空闲
3. 实现当服务器超过7秒没有读或者写操作时，就提示读写空闲

实现：在pipeline中加入IdleStateHandler
1. IdleStateHandler 是netty 提供的处理空闲状态的处理器
2. long readerIdleTime : 表示多长时间没有读, 就会发送一个心跳检测包检测是否连接
3. long writerIdleTime : 表示多长时间没有写, 就会发送一个心跳检测包检测是否连接
4. long allIdleTime : 表示多长时间没有读写, 就会发送一个心跳检测包检测是否连接
5. 文档说明
	 triggers an {@link IdleStateEvent} when a {@link Channel} has not performed
     * read, write, or both operation for a while.
6. 当 IdleStateEvent 触发后 , 就会传递给管道 的下一个handler去处理。通过调用(触发)下一个handler 的 userEventTiggered , 在该方法中去处理 IdleStateEvent(读空闲，写空闲，读写空闲)

## Netty 通过WebSocket编程实现服务器和客户端长连接
1. Http协议是无状态的, 浏览器和服务器间的请求响应一次，下一次会重新创建连接.
2. 要求：实现基于webSocket的长连接的全双工的交互
3. 改变Http协议多次请求的约束，实现长连接了， 服务器可以发送消息给浏览器
4. 客户端浏览器和服务器端会相互感知，比如服务器关闭了，浏览器会感知，同样浏览器关闭了，服务器会感知


## Netty 本身的编码解码的机制和问题 ##
1. Netty 提供的编码器
	- StringEncoder，对字符串数据进行编码
	- ObjectEncoder，对 Java 对象进行编码
2. Netty 提供的解码器
	- StringDecoder, 对字符串数据进行解码
	- ObjectDecoder，对 Java 对象进行解码
3. ObjectDecoder 和 ObjectEncoder 可以用来实现 POJO 对象或各种业务对象的编码和解码，底层使用的仍是 Java 序列化技术 , 而Java 序列化技术本身效率就不高
	- 无法跨语言
	- 序列化后的体积太大，是二进制编码的 5 倍多。
	- 序列化性能太低

## Protobuf ##
1. Protobuf（Google Protocol Buffers） 是 Google 发布的开源项目，是一种轻便高效的结构化数据存储格式
2. 适合做数据存储或 RPC 目前很多公司由 http+json 转为 tcp+protobuf
3. Protobuf 是以 message 的方式来管理数据的
4. 支持跨平台、跨语言，即[客户端和服务器端可以是不同的语言编写的] （支持目前绝大多数语言，例如 C++、C#、Java、python 等）

## Log4j 整合到Netty ##
1. 在Maven 中添加对Log4j的依赖 在 pom.xml

2. 配置 Log4j , 在 resources/log4j.properties

## TCP 粘包和拆包 ##
1. TCP是面向连接的，面向流的，提供高可靠性服务。收发两端（客户端和服务器端）都要有一一成对的socket，因此，发送端为了将多个发给接收端的包，更有效的发给对方，使用了优化方法（Nagle算法），将多次间隔较小且数据量小的数据，合并成一个大的数据块，然后进行封包。这样做虽然提高了效率，但是接收端就难于分辨出完整的数据包了，因为面向流的通信是无消息保护边界的
2. 由于TCP无消息保护边界, 需要在接收端处理消息边界问题，也就是我们所说的粘包、拆包问题

## TCP 粘包和拆包解决方案 ##
1. 使用自定义协议 + 编解码器
2. 关键就是要解决 **服务器端每次读取数据长度的问题**

## Netty的沾包拆包解决方案

1. 固定长度的拆包器 FixedLengthFrameDecoder，每个应用层数据包的都拆分成都是固定长度的大小
2. 行拆包器 LineBasedFrameDecoder，每个应用层数据包，都以换行符作为分隔符，进行分割拆分
3. 分隔符拆包器 DelimiterBasedFrameDecoder，每个应用层数据包，都通过自定义的分隔符，进行分割拆分
4. 基于数据包长度的拆包器 LengthFieldBasedFrameDecoder，将应用层数据包的长度，作为接收端应用层数据包的拆分依据。按照应用层数据包的大小，拆包。这个拆包器，有一个要求，就是应用层协议中包含数据包的长度

----------


## Netty启动过程源码分析 ##

1. 创建2个 EventLoopGroup 线程池数组。数组默认大小CPU*2，方便chooser选择线程池时提高性能
2. BootStrap 将 boss 设置为 group属性，将 worker 设置为 childer 属性
3. 通过 bind 方法启动，内部重要方法为 initAndRegister 和 dobind 方法
4. initAndRegister 方法会反射创建 NioServerSocketChannel 及其相关的 NIO 的对象， pipeline ， unsafe，同时也为 pipeline 初始了 head 节点和 tail 节点。
5. 在register0 方法成功以后调用在 dobind 方法中调用 doBind0 方法，该方法会 调用 NioServerSocketChannel 的 doBind 方法对 JDK 的 channel 和端口进行绑定，完成 Netty 服务器的所有启动，并开始监听连接事件

## Netty接受请求过程源码剖析 ##
总体流程：接受连接----->创建一个新的NioSocketChannel----------->注册到一个 worker EventLoop 上--------> 注册selecot Read 事件。

1. 服务器轮询 Accept 事件，获取事件后调用 unsafe 的 read 方法，这个 unsafe 是 ServerSocket 的内部类，该方法内部由2部分组成
2. doReadMessages 用于创建 NioSocketChannel 对象，该对象包装 JDK 的 Nio Channel 客户端。该方法会像创建 ServerSocketChanel 类似创建相关的 pipeline ， unsafe，config
3. 随后执行 执行 pipeline.fireChannelRead 方法，并将自己绑定到一个 chooser 选择器选择的 workerGroup 中的一个 EventLoop。并且注册一个0，表示注册成功，但并没有注册读（1）事件

## 将耗时任务添加到异步线程池 ##
在 Netty 中做耗时的，不可预料的操作，比如数据库，网络请求，会严重影响 Netty 对 Socket 的处理速度。

1. Handler 种加入线程池
	第一种方式在 handler 中添加异步，可能更加的自由，比如如果需要访问数据库，那我就异步，如果不需要，就不异步，异步会拖长接口响应时间。因为需要将任务放进 mpscTask 中。如果IO 时间很短，task 很多，可能一个循环下来，都没时间执行整个 task，导致响应时间达不到指标。
2. Context 中添加线程池
	第二种方式是 Netty 标准方式(即加入到队列)，但是，这么做会将整个 handler 都交给业务线程池。不论耗时不耗时，都加入到队列里，不够灵活。