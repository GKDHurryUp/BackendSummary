#  网络IO #

## 什么是socket？ ##

是操作系统对TCP/UDP协议的一个封装，为了供上层应该调用，进行网络传输的API

四元组：

​	源IP地址、目的IP地址、源端口、目的端口

五元组：

​	源IP地址、目的IP地址、协议号、源端口、目的端口

七元组：

　源IP地址、目的IP地址、协议号、源端口、目的端口、服务类型、接口索引

协议TCP/UDP也会有影响

应用程序调用内核函数，从网卡上把数据加载进内存（内核空间），后拷贝到用户空间，应用程序便可调用用户空间

## BIO - Blocking I/O ##
描述：
同步阻塞IO，服务器实现为一个链接一个线程，可以通过线程池机制改善

调用内核函数socket得到文件描述符fd，并调用bind(), listen()，在**accept()阻塞，并在recv()阻塞**

```C
//创建socket
int fd = socket(AF_INET, SOCK_STREAM, 0);   
//绑定
bind(fd, ...)
//监听
listen(fd, ...)
//接受客户端连接
int c = accept(fd, ...)
//接收客户端数据
recv(c, ...);
```

编程流程：
	分派一个线程来监听（while(true)）有无客户端Socket进行连接，在这个线程内维护一个线程池，如果有连接进来，在线程池内开启一个新的线程（没有Socket连接的话，会阻塞等待Socket连接），对Socket进行读写，如果没有数据可读，此线程会阻塞在read上

1. 服务器端启动一个ServerSocket
2. 客户端启动Socket对服务器进行通信，默认情况下服务器端需要对每个客户建立一个线程与之通讯
3. 客户端发出请求后，先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝
4. 如果有响应，客户端线程会等待请求结束后，在继续执行

BIO问题
1. 每个请求都需要创立独立的线程，与对应的客户端进行读写业务
2. 并发量较大时，需要创建大量线程来处理连接，系统占用资源大
3. 连接建立之后，如果线程上没有数据可读，线程会阻塞在read操作上，造成线程资源浪费	


## NIO - Non blocking I/O ##
Java的NIO是new IO，系统内核的NIO是non blocking IO
描述：非阻塞，直接返回

系统低层，accept()返回-1，read()返回-1，
问题：每次循环有很大的浪费，都需要调用read

## 多路复用NIO ##
同步非阻塞，服务器实现为一个线程可以处理多个请求，客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询，通过事件驱动的方式来进行I/O处理。



系统低层：select()或者poll()不断传入文件描述符，返回status，表示能否可读

优点：用户态内核态切换变少，read调用变少

### select

**如果列表中的socket都没有数据，挂起进程，直到有一个socket收到数据，唤醒进程**。

使用**数组**存放需要监听的socket，处于效率考虑，默认只能监视1024个socket

```C
int s = socket(AF_INET, SOCK_STREAM, 0);  
bind(s, ...)
listen(s, ...)
 
int fds[] =  存放需要监听的socket
 
while(1){
    int n = select(..., fds, ...)
    for(int i=0; i < fds.count; i++){
        if(FD_ISSET(fds[i], ...)){
        //fds[i]的数据处理
    }
}
```

缺点：

1. 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
2. 仅仅知道有I/O事件发生了，却不知道是哪一个发生（有可能多个），只能无差别轮询，所以为O（n）复杂度
3. select支持的文件描述符数量太小了，默认是1024

### poll

本质上和select没有区别，它将用**户传入的数组拷贝到内核空间**，然后查询每个fd对应的设备状态， **但是它没有最大连接数的限制**，原因是它是基于**链表**来存储的。因此时间复杂度也为O(n)

### epoll

**epoll可以理解为event poll**，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们，epoll实际上是==事件驱动==（==每个事件关联上fd==），==直接在内核开辟空间==，防止重复拷贝
epoll_create(红黑树、VFS虚拟文件系统)
epoll_ctl
epoll_wait

三部分核心：

面向缓冲（块）进行读写，基于Channel和Buffer，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector用于监听多个通道的事件（连接请求，数据到达）

### 操作系统如何知道网络数据对应于哪个socket？

因为一个socket对应着一个端口号，而网络数据包中包含了ip和端口的信息，内核可以通过端口号找到对应的socket。当然，为了提高处理速度，操作系统会维护端口号到socket的索引结构，以快速读取。（就是说网卡中断CPU后，CPU的中断函数从网卡存数据的内存拷贝数据到对应fd的接收缓冲区，具体是哪一个fd，CPU会检查port，放到对应的fd中）

### **什么时候select优于epoll？**

一般认为如果在并发量低，socket都比较活跃的情况下，select效率更高，也就是说活跃socket数目与监控的总的socket数目之比越大，select效率越高，因为select反正都会遍历所有的socket，如果比例大，就没有白白遍历。加之于select本身实现比较简单，导致总体现象比epoll好



### Buffer 缓冲区 ###

介绍：
Buffer就是一个内存块，底层为数组，Buffer对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。数据的读写是通过Buffer，可以双向需要flip切换

Buffer类定义了所有缓冲区都具有的四个属性
```java
private int mark = -1;	
private int position = 0;  // 下一个要被读或者写的元素的索引
private int limit;	// 缓冲区当前的终点，不能超过
private int capacity; // 容量，不可改变
```


Java中的基本数据类型（boolean除外），都有一个Buffer类型与之对应，最常用的是ByteBuffer类

### Channel 通道 ###
介绍：
1. 通道可以同时进行读写，而流只能读或者只能写
2. 通道可以实现异步读写数据
3. 通道可以从缓冲读数据，也可以写数据到缓冲: 
4. Channel在NIO中是一个接口
		public interface Channel extends Closeable{}
5. 常用的 Channel 类有：FileChannel 用于文件的数据读写，DatagramChannel 用于 UDP 的数据读写，ServerSocketChannel 和 SocketChannel 用于 TCP 的数据读写。（ServerSocketChanne 类似 ServerSocket , SocketChannel 类似 Socket）

FileChannel类

	public int read(ByteBuffer dst) //从通道读取数据并放到缓冲区中
	public int write(ByteBuffer src) //把缓冲区的数据写到通道中
	public long transferFrom(ReadableByteChannel src, long position, long count) //从目标通道中复制数据到当前通道，零拷贝
	public long transferTo(long position, long count, WritableByteChannel target) //把数据从当前通道复制给目标通道，零拷贝


细节：
1. ByteBuffer 支持类型化的put 和 get, put 放入的是什么数据类型，get就应该使用相应的数据类型来取出
2. 可以将一个普通Buffer 转成只读Buffer
3. NIO 还提供了 MappedByteBuffer， 可以让文件直接在内存（堆外的内存）中进行修改，操作系统不需要拷贝，而如何同步到文件由NIO 来完成
4. NIO 还支持 通过多个Buffer (即 Buffer 数组) 完成读写操作，即 Scattering 和 Gathering 

### Selector 选择器 ###
介绍：
1. Selector 能够检测多个注册的通道上是否有事件发生，多个Channel以事件的方式可以注册到同一个Selector
2. 只有在 连接/通道 真正有读写事件发生时，才会进行读写，就大大地减少了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程，避免了多线程之间的上下文切换导致的开销

Selector 类是一个抽象类

	public abstract class Selector implements Closeable { 
		public static Selector open();//得到一个选择器对象
		public int select(long timeout);//监控所有注册的通道，当其中有 IO 操作可以进行时，将对应的 SelectionKey 加入到内部集合中并返回，参数用来设置超时时间
		public Set<SelectionKey> selectedKeys();//从内部集合中得到所有的 SelectionKey	
	}

线程监听，循环调用accept()，read(),调用内核recvFrom(noblock,...)



## AIO（了解） ##
异步非阻塞，JDK1.7引入，引入异步通道的概念，采用了Proactor模式，有效的请求才启动线程
基于事件和回调机制，应用操作之后会直接返回，不会阻塞在那里，当处理完成后，有回调函数进行处理

## BIO、NIO、AIO使用场景
1. BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发有局限，程序简单
2. NIO适用于连接数目多且连接时间比较短（轻操作）的架构，如聊天服务器，弹幕服务器，服务器间通讯。
	编程比较复杂，JDK1.4开始支持
3. AIO适用于连接数目多且连接时间比较长（重操作）的架构，如相册服务器
	编程比较复杂，JDK1.7开始支持

【高并发引起的问题】
1、线程不够用, 就算使用了线程池复用线程也无济于事;?

2、阻塞I/O模式下,会有大量的线程被阻塞,一直在等待数据,这个时候的线程被挂起,只能干等,CPU利用率很低,换句话说,系统的吞吐量差
3、如果网络I/O堵塞或者有网络抖动或者网络故障等,线程的阻塞时间可能很长。整个系统也变的不可靠;


## NIO应用实例-群聊系统 ##
1. 编写一个 NIO 群聊系统，实现服务器端和客户端之间的数据简单通讯（非阻塞）
2. 实现多人群聊
3. 服务器端：可以监测用户上线，离线，并实现消息转发功能
4. 客户端：通过channel 可以无阻塞发送消息给其它所有用户，同时可以接受其它用户发送的消息(有服务器转发得到)

【简述下NIO的三种Reactor模式？】
1.Reactor单线程
2.Reactor多线程
	在处理事件时，提升效率
3.Reactor主从

## 零拷贝 ##
- 数据加载到用户空间和内核空间的公共区域，用户只需要根据地址访问。
- 在 Java 程序中，常用的零拷贝有 mmap(内存映射) 和 sendFile。
- 零拷贝是从操作系统角度来看，没有cpu拷贝
- 零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如更少的上下文切换，更少的 CPU 缓存伪共享以及无 CPU 校验和计算

在传统IO中，要经历四次拷贝，四次状态切换
先把硬盘上的数据通过DMA（直接内存拷贝）拷贝到内核Buffer，然后在使用CPU拷贝到用户Buffer，数据在用户态修改完之后，再使用CPU拷贝到Socket Buffer，再由DMA拷贝到protocol engine协议栈

1. mmap优化 --- 三次拷贝，三次状态切换。	
	- 通过内存映射，将文件映射到内核缓冲区，同时，用户空间可以共享内核空间的数据。这样，在进行网络传输时，就可以减少内核空间到用户控件的拷贝次数。

2. sendFile优化  ---三次拷贝，两次状态切换
	- Linux2.1提供sendFile函数，数据不经过用户态，直接从内核缓冲区进入到Socket Buffer，较少了一次上下文切换
3. sendFile进一步优化  ---两次拷贝，两次状态切换
	- 在Linux2.4版本中，做了修改，避免从内核缓冲区拷贝到Socket Buffer的操作，直接拷贝到协议栈，再一次减少了数据拷贝（其实是有一次cpu拷贝，kernel Buffer->socket Buffer，但是拷贝的信息很少，如length，offset，消耗低，可以忽略）