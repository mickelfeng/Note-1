# I/O模型的基本认识

## 介绍操作系统的I/Om模型之前，先了解一下几个概念：

### 阻塞调用与非阻塞调用

- 阻塞调用是指调用结果返回之前，当前线程会被挂起，调用线程只有在得到结果之后才会返回。
- 非阻塞调用是指在不能在立刻得到结果之前，该调用不会阻塞当前线程。

两者的最大区别在于被调用方在收到请求到返回结果之前的这段时间内，调用方是否一直在等待。

**阻塞** *是指调用方一直在等待而且别的事情什么都不做*；**非阻塞** *是指调用方先去忙别的事情*。

### 同步处理与异步处理

同步处理是指被调用方得到最终结果之后才返回给调用方；异步处理是指被调用方先返回应答，然后再j计算调用结果，计算完最终结果后再通知并返回给调用方。

### 阻塞、非阻塞和同步、异步的区别

- 阻塞、非阻塞的讨论对象是调用者；
- 同步、异步的讨论对象是被调用者。

### recvfrom函数

recvfrom函数（经Socket接收数据），这里把它视为系统调用。

### 一个输入操作通常包含两个不同的阶段：

- 等待数据准备好；
- 从内核向进程复制数据。

对于一个套接字上的输入操作，第一步通常设计等待数据从网络中到达。当所等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用进程缓冲区。

实际应用程序在系统调用完成上面的2步操作时，调用方式的阻塞、非阻塞，操作系统在处理应用程序请求时，处理方式的同步、异步处理的不同，可以分为5种I/O模型。

# I/O模型1：阻塞式I/O模型（blocking I/O）

![](../../image/433.png)

在阻塞式I/O模型中，应用程序在从调用recvfrom开始到它返回有数据报准备好这段时间是阻塞的，recvfrom返回成功后，应用进程开始处理数据报。

**比喻**：一个人在钓鱼，当没有鱼上钩时，就坐在岸边一直等。

**优点**：程序简单，在阻塞等待数据起劲啊进程/线程挂起，基本不会占用CPU资源。

**缺点**：每个链接需要独立的进程/线程单独处理，当并发请求量大时为了维护程序，内存、线程切换开销较大，这种模式在实际生产中很少使用。

# I/O模型2：非阻塞式I/O模型（non_blocking I/O）

![](../../image/1.jpeg)

在非阻塞式I/O模型中，应用程序把一个套接口设置为非阻塞，就是告诉内核，当所请求的I/O操作无法完成时，不要将进程睡眠。而是返回一个错误，应用程序基于I/O操作函数将不断的轮询数据是否已经准备好了，如果没有准备好，继续轮询，直到数据准备好为止。

**比喻**：边钓鱼边玩手机，隔会再看看有没有鱼上钩，有的话就迅速拉杆。

**优点**：不会阻塞在内核的等待数据过程，每次发起的I/O请求可以理解返回，不用阻塞等待，实时性较好。

**缺点**：轮询将会不断地询问内核，浙江占用大量的CPU时间，系统资源利用率较低，所以一般Web服务器不使用这种I/O模型。

# I/O模型3：I/O复用模型（I/O multiplexing）

![](../../image/2.jpeg)

在I/O复用模型中，会用到Select或Poll函数或Epoll函数（Linux2.6以后的内核开始支持），这两个函数也会使用进程阻塞，但是和阻塞I/O有所不同。这两个函数可以同时阻塞多个I/O操作，而且可以同时对多个读操作，多个写操作的I/O函数进行检测，直到有数据可读或可写时，才真正调用I/O操作函数。

**比喻**：放了一堆鱼竿，在岸边一直守着这堆鱼竿，没鱼上钩就玩手机。

**优点**：可以基于一个阻塞对象，同时在多个摩艾殊夫上等待就绪，而不是使用多个线程（每个文件描述符一个线程），这样就可以大大j节省系统资源。

**缺点**：当链接数较少时效率相比多线程+阻塞I/O模型效率较低，可能延迟更大，因为单个连接处理需要2次系统调用，占用时间会有增加。众所周知，Nginx这样的高性能互联网反向代理服务器大获成功的关键就是得益于Epoll。

# I/O模型4：信号驱动式I/O模型（signal-driven I/O）

![](../../image/3.jpeg)

在信号驱动式 I/O 模型中，应用程序使用套接口进行信号驱动 I/O，并安装一个信号处理函数，进程继续运行并不阻塞。当数据准备好时，进程会收到一个 SIGIO 信号，可以在信号处理函数中调用 I/O 操作函数处理数据。

**比喻：**鱼竿上系了个铃铛，当铃铛响，就知道鱼上钩，然后可以专心玩手机。
**优点：**线程并没有在等待数据时被阻塞，可以提高资源的利用率。
**缺点：**信号 I/O 在大量 IO 操作时可能会因为信号队列溢出导致没法通知。

信号驱动 I/O 尽管对于处理 UDP 套接字来说有用，即这种信号通知意味着到达一个数据报，或者返回一个异步错误。

但是，对于 TCP 而言，信号驱动的 I/O 方式近乎无用，因为导致这种通知的条件为数众多，每一个来进行判别会消耗很大资源，与前几种方式相比优势尽失。  

# I/O模型5：异步 I/O 模型（即AIO，全称asynchronous I/O）

![](../../image/4.jpeg)

  由 POSIX 规范定义，应用程序告知内核启动某个操作，并让内核在整个操作（包括将数据从内核拷贝到应用程序的缓冲区）完成后通知应用程序。

这种模型与信号驱动模型的主要区别在于：信号驱动 I/O 是由内核通知应用程序何时启动一个 I/O 操作，而异步 I/O 模型是由内核通知应用程序 I/O 操作何时完成。

**优点：**异步 I/O 能够充分利用 DMA 特性，让 I/O 操作与计算重叠。
**缺点：**要实现真正的异步 I/O，操作系统需要做大量的工作。目前 Windows 下通过 IOCP 实现了真正的异步 I/O。

而在 Linux 系统下，Linux 2.6才引入，目前 AIO 并不完善，因此在 Linux 下实现高并发网络编程时都是以 IO 复用模型模式为主。

# 5种I/O模型总结

![](../../image/5.jpeg)

  从上图中我们可以看出，越往后，阻塞越少，理论上效率也是最优。

这五种 I/O 模型中，前四种属于同步 I/O，因为其中真正的 I/O 操作(recvfrom)将阻塞进程/线程，只有异步 I/O 模型才与 POSIX 定义的异步 I/O 相匹配。  