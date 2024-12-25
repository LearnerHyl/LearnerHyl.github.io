# Abstractions 3: IPC, Pipes and Sockets A quick programmer’s viewpoint

# 1. Intro&&Recall

## 1.1 Putting it together: web server

以Web Server为例来说明今天的要讲的一些重点：

![image-20230818212413072](https://s2.loli.net/2023/08/18/qiTHe75B9aVmpvf.png)

用户态：即服务器进程所在的空间，下面是内核态，再下面是硬件。服务器创建了一个socket，对应的在内核态有一个缓冲区用于存储来自client的请求。因为收到的数据首先放到内核缓冲区，因此要用low-level接口去读取数据。涉及到内核缓冲区和用户缓冲区的概念，忘了可以看看上一节课的笔记。

1. 服务器进程最终要调用read()来从内核缓冲区读取到达的请求，因为刚开始socket缓冲区是空的，所以read()进程进入等待状态。这时可以让出处理器让别的线程做事。
2. client的请求到达，经过网络接口卡格式化，被存入到内核缓冲区，触发一个interrupt。内核不具备解析请求的能力，在它眼里这些只是一堆字节而已。
3. read()请求监测到内核缓冲区有数据，同样触发一个interrupt：将缓冲区数据复制到用户缓冲区（request buffer）。同时也唤醒服务器相关进程进行处理。
4. 服务器进程调用相关API解析用户缓冲区的数据，搞清楚这个请求要做什么。
5. 读取相关数据，不管高层API是什么，最终都要用low-level API的read()，触发系统调用，进入内核态，需要从磁盘请求数据，I/O阻塞，该线程暂时挂起，让出CPU做别的事情。
6. 内核向Disk 控制器发出读取数据请求，Disk将这个请求发送给磁盘。磁盘缓冲区开始读取需要的数据。
7. 一旦磁盘的数据攒够一定的量，会触发一个interrupt，告诉内核我准备好发送了，内核接收磁盘发送的数据，可能一轮发完，也可能多轮。
8. 当缓冲区有目标磁盘数据时，read()调用被唤醒，涉及到一个interrupt：内核将数据发到用户缓冲区（reply buffer）。
9. 服务器调用相关API对数据进行必要的处理，格式化后，已经可以返回给client。
10. 服务器调用reply操作：底层调用write()操作，系统调用，对socket的内核缓冲区进行写入。
11. write()的工作：内核负责将处理后的数据从reply buffer复制到socket buffer，用来回复请求的内核buffer。
12. 内核将reply包的数据进行进一步格式化，之后用DMA操作回复给client。

今天主要学习Request和Reply这两个涉及到Network Communication的内部流程。

## 1.2 Goals for Today: IPC and Sockets

关键思路：是让进程间通信和全球范围的通信看起来像是文件 I/O。

我们将介绍管道（Pipes）和套接字（Sockets）。

介绍为网络服务器设置的 TCP/IP 连接。

![image-20230818200155055](https://s2.loli.net/2023/08/18/3vAkHCqwM82di1I.png)

## 1.3 Today: Communication Between Processes

 当进程需要相互通信时怎么办？

- 为什么？共享任务、涉及安全问题的合作项目 

进程抽象目的是防止进程间通信！

- 避免一个进程干扰或窃取另一个进程的信息

因此，必须采取特殊措施（且需双方进程同意）

- 可以看做在安全措施上打了一个洞，通过这个洞，两个进程可以合法的进行交流。

这称为“进程间通信”（IPC）

进程间通信（IPC）的概念源于多任务环境，其中不同进程需要协作完成任务或共享资源。然而，进程抽象的本意是限制进程直接访问其他进程的资源和数据，以保证系统安全。由于这一原因，我们需要设计特殊的机制来实现进程间通信。

IPC 是一种突破这种隔离性的技术，通过特定的协议和方法，使得进程之间可以安全、顺畅地交换信息。IPC 的具体方式包括管道、消息队列、共享内存、信号量、套接字等，各种方法有着不同的使用场景和优缺点。

例如，共享内存可以提供高效的数据交换，但可能引发同步问题；信号量则解决了同步问题，但通信能力有限。因此，在实际应用中，我们需要根据情况选择合适的 IPC 技术。

# 2. The ways of processes communication

## 2.1 Bad Method-Persistent Storage Media(Stupid)

生产者（写者）和消费者（读者）可能是不同的进程：

- 可能在时间上是分开的
- 如何实现选择性通信？
  - 简单选择：使用文件！

我们已经展示过父子进程是如何共享文件描述符的：

![image-20230819172621626](https://s2.loli.net/2023/08/19/CoUkRIubd8jlM1V.png)

为什么这可能会浪费资源？

- 如果仅需短暂的通信（非持久性）-涉及到的数据只是中间数据，不需要持久存储，这将非常昂贵

当生产者和消费者是不同的进程时，可以使用文件作为一种简单的选择来实现选择性通信。然而，使用文件的方法可能在一些场景下显得既低效又浪费资源。

首先，我们已经展示了如何通过文件描述符来在父子进程之间实现通信。然而，采用文件进行通信的代价相对较高，尤其是在只需要短暂通信（非持久性）的场景下。文件 I/O 操作通常需要较长的时间，因为它涉及到磁盘访问。此外，每次通信都要打开、关闭文件，并进行读/写操作，可能导致性能降低。

## 2.2 Shared Memory: Better Option? Topic for another day!

![image-20230819172809249](https://s2.loli.net/2023/08/19/JMAmS9jfa5doqOh.png)

共享内存的方式难以控制，但是速度很快。

在讲完进程间通信和如何设置共享内存区域后，就会专门讲解`Shared-Memory Model`。

进程间通信（IPC）机制与共享内存机制还是有些不同的。下面会讲一些常用的方法。

# 3. In-Memory Queue

假设我们请求内核帮忙？

- 考虑一个内存中的队列
- 通过系统调用进行访问（出于安全原因）：

![image-20230819193646551](https://s2.loli.net/2023/08/19/4xiqDgEJQTP26uG.png)

A写入的数据将保存在内存中，直到B读取它

- 与文件使用的接口相同！
- 在内部更高效，因为没有任何数据写入磁盘

一些问题：

如何设置出这样的一个内核缓冲队列？

如果A产生数据的速度快于B消费数据的速度怎么办？

- 如果A比较快，当写满缓冲区后，A可以进入wait状态，直到缓冲区有空闲或者缓冲区为空时再唤醒生产者。

如果B消费数据的速度快于A生产数据的速度怎么办？

- 若B消费速度比较快，当缓冲区没数据后，B可以进入wait状态，直到缓冲区有数据或者缓冲区满了再唤醒B。

wait可以理解为休眠状态，这里涉及到进程的调度机制。下面举一些常见的进程间内存队列通信模式的例子。

## 3.1 POSIX/Unix PIPE

![image-20230819194632969](https://s2.loli.net/2023/08/19/COM1aeBpPtSKITV.png)

内存缓冲区是有限的：

- 如果生产者（A）在缓冲区满时尝试写入，它将会阻塞（休眠直到有可用空间）
- 如果消费者（B）在缓冲区为空时尝试读取，它将会阻塞（休眠直到有数据可读） 

涉及到一个low-level file的API调用：`int pipe(int fileds[2])`;

- 在进程中分配两个新的文件描述符
- 写入到 fileds[1] 时从 fileds[0] 读取
- 以固定大小队列实现

POSIX/Unix 管道（PIPE）就是这种模式的一个典型例子。在这里，使用了一个有限大小的内存缓冲区。当生产者尝试在缓冲区已满时写入数据时，它会被阻塞，直到有空间可用；同样，当消费者尝试在缓冲区为空时读取数据时，它会被阻塞，直到有数据可读。

我们可以使用 `pipe` 函数来实现这个功能，其原型为：`int pipe(int fileds[2])`。这个函数会在进程中分配两个新的文件描述符，分别表示读端和写端。当数据写入到 `fileds[1]` 时，可以从 `fileds[0]` 中读取。这个管道实际上是一个固定大小的队列，用于在进程间传输数据。

<u>管道的特点是简单、高效且易于实现，但它也有一些局限性，例如仅适用于具有父子关系的进程间通信</u>。在更复杂的场景下，我们可能需要考虑其他高级的 IPC 技术，如消息队列、共享内存等。不过，根据实际需求选择合适的 IPC 方法仍然是必要的。

### 3.1.1 Single-Process Pipe Example

```c
#include <unistd.h>
// 很stupid的一个例子，一个进程创建管道，从一端写再从一端读，当然只是为了演示而已
// 工程中我们不会这么做
int main(int argc, char *argv[])
{
    char *msg = "Message in a pipe.\n";
    char buf[BUFSIZE];
    int pipe_fd[2];
    if (pipe(pipe_fd) == -1) {
   	 	fprintf (stderr, "Pipe failed.\n"); return EXIT_FAILURE;
    }
    ssize_t writelen = write(pipe_fd[1], msg, strlen(msg)+1);
    printf("Sent: %s [%ld, %ld]\n", msg, strlen(msg)+1, writelen);
    
    ssize_t readlen = read(pipe_fd[0], buf, BUFSIZE);
    printf("Rcvd: %s [%ld]\n", msg, readlen);
    
    // 先关闭读端口，再关闭写端口
    close(pipe_fd[0]);
    close(pipe_fd[1]);
}
```

### 3.1.2 Pipes Between Processes

利用fork()调用的特性可以实现用pipe在父子进程之间通信：

![image-20230820151818203](https://s2.loli.net/2023/08/20/soJKWPHw4dQ2gCM.png)

```c
// continuing from earlier
pid_t pid = fork();
if (pid < 0) {
    fprintf (stderr, "Fork failed.\n");
    return EXIT_FAILURE;
}
// 父进程负责写入管道
// 因为父进程只用到fd[1]，因此关闭fd[0]
if (pid != 0) {
    ssize_t writelen = write(pipe_fd[1], msg, msglen);
    printf("Parent: %s [%ld, %ld]\n", msg, msglen, writelen);
    close(pipe_fd[0]);
} else { // 子进程从管道中读取。同理子进程只用到fd[0]，因此关闭fd[1]
    ssize_t readlen = read(pipe_fd[0], buf, BUFSIZE);
    printf("Child Rcvd: %s [%ld]\n", msg, readlen);
    close(pipe_fd[1]);
}
```

### 3.1.3 When do we get EOF on a pipe?

进程什么时候可以从pipe得到一个EOF返回值（无论是read()还是write()）:

- 当最后一个写描述符被关闭后，管道实际上是关闭的：读取操作仅返回 EOF。这表示没有更多数据可供读取，因为所有的写端已经关闭。
- 当最后一个读描述符被关闭时，写操作会产生 SIGPIPE 信号：如果进程忽略此信号，那么写操作将因“EPIPE”错误而失败。该错误表明写端试图向没有打开读端的管道中写入数据。

所以确保当不再用某个管道时，记得关闭所有的pipe描述符，但是管道的应用范围较小，只适用于具有父子关系的进程，下面我们将学习一些更普适的IPC方法。

# 4. Protocol（more details on future lecture）

## 4.1 Once we have communication, we need a protocol

一个协议是关于如何进行通信的一种约定，包括以下内容：

- 语法（`Syntax`）：如何规定和构建通信内容，根据一定规则解释字节流的含义。
  - 包括消息的发送和接收的格式和顺序。
- 语义（`Semantics`）：通信的含义，一般采用一个状态机来描述。
  - 包括在发送、接收消息或计时器到期时采取的动作。

协议通常通过状态机来正式描述，

- 通常表示为消息传输图。

实际上，在网络中可能需要一种方法来在数字、字符串等不同表示之间进行转换。

- 这样的转换通常属于远程过程调用（RPC）设施的一部分。
- 虽然目前无需担心这一点，但这显然是协议的一部分。

协议定义了在某个通信系统或网络环境中，信息交换的规则和约定。所有遵循相同协议的设备和应用程序，都能按照这些规则顺利地共享数据，从而实现互操作性。可以使用协议的多层架构（如互联网协议套件）来更好地组织和实现协议规定的功能。这允许构建复杂的通信系统，并简化设备之间的互操作性。

## 4.2 Client-Server Protocols: Cross-Network IPC

客户端是“有时在线”的：

- 当感兴趣时向服务器发送服务请求。比如，笔记本电脑或手机上的网页浏览器。
- 不直接与其他客户端进行通信。
- 需要知道服务器地址。

服务器是“始终在线”的：

- 为许多客户端提供服务。比如，[www.cnn.com](http://www.cnn.com/) 的网页服务器。
- 不主动与客户端联系。
- 需要一个固定的、众所周知的地址。

![image-20230820162109632](https://s2.loli.net/2023/08/20/vV1GPWXKFA6TRs7.png)

# 5. Socket  Abstraction(based on：TCP protocol)

## 5.1 What is a Network Connection?

在可能位于不同机器上的两个进程之间的双向字节流：

- 目前，我们讨论的是“TCP 连接”。

抽象地说，两个端点 A 和 B 之间的连接包括：

- 从 A 发往 B 的数据队列（有界缓冲区）。
- 从 B 发往 A 的数据队列（有界缓冲区）。

TCP（传输控制协议）连接提供了一种在两个端点（例如，客户端和服务器）之间进行可靠、顺序、基于字节流的通信的方法。在这种连接中，数据通过有界缓冲区（或队列）在端点之间发送，确保数据能够按照发送顺序和完整性被接收。

### 5.1.1 Socket and Port

套接字（Socket）和端口（Port）的定义以及它们之间的区别如下：

套接字（Socket）： 套接字是通信链路的一个端点，用于表示在两个设备间建立连接的抽象概念。在网络编程中，套接字用于提供一种进程间通信（IPC）的手段，让不同设备上运行的进程可以相互发送和接收数据。套接字可以通过特定的传输层协议（如TCP或UDP）和网络层协议（如IPv4或IPv6）进行通信。

端口（Port）： 端口是一个整数，它用于表示在网络设备（如计算机或路由器）上运行的特定应用程序或服务的唯一标识。端口号范围从0到65535。<u>在TCP/IP协议中，当数据从一个设备传输到另一个设备时，端口号帮助确定这些数据应该路由到哪个应用程序。</u>通常情况下，一些已知的端口号被分配给特定的网络服务，如HTTP（端口80）或HTTPS（端口443）。

## 5.2 The Socket Abstraction: Endpoint for Communication

**关键思想：全球范围内的通信看起来像文件I/O。**

![image-20230820163151732](https://s2.loli.net/2023/08/20/c6bWkD7GnoRwzTy.png)

`socket`：**通信的某一端端点**。

- 用于暂时存储结果的队列。

`Connection`：通过网络连接的两个套接字，实现网络间进程通信（IPC over network）

- 如何打开（open）？
- 命名空间是什么？
- 它们在时间上是如何连接的？

套接字（Socket）是一种抽象概念，用于表示在两个设备之间建立通信连接的端点。通信过程中，发送和接收数据的方式类似于本地文件I/O操作。套接字之间的连接通过网络实现，使得不同设备上的进程可以相互通信。

1. 如何打开？

   为了建立套接字连接，首先需要创建一个套接字。在大多数编程语言中，这可以通过调用 socket() 函数来完成。然后，根据使用的协议（TCP或UDP）、需要连接的远程服务器地址和端口，来连接（或绑定）此套接字。

2. 命名空间是什么？

   命名空间用于标识套接字以及它们之间的通信。例如，在Internet环境中，套接字命名空间包括IP地址和端口号。这些信息表示套接字在网络中的唯一位置，以便在同一网络上的其他节点可以找到和与之通信。

3. 它们在时间上是如何连接的？

   套接字连接在时间上的建立取决于连接的类型和使用的协议。对于基于TCP的连接，首先需要经历三次握手过程。在此过程中，客户端和服务器通过交换特定的消息来确认双方都已准备好建立连接。成功完成握手后，连接便被建立。对于基于UDP的连接，由于其无连接性，可以直接发送或接收数据包，因此连接在时间上的建立相对简单。

建立连接后，进程可以开始在套接字之间发送和接收数据，就像进行本地文件I/O操作一样。当通信完成后，通常需要关闭套接字，以释放资源。

## 5.3 Sockets: More Details

套接字：网络连接的一个端点的抽象。

- 用作进程间通信的另一种机制。
- 即使在不复制UNIX I/O的情况下，大多数操作系统（如 Linux、Mac OS X、Windows）也提供了此功能。
- 被POSIX标准化。

套接字首次引入于4.2 BSD（伯克利标准发行版）Unix：

- 这个版本带来了一些巨大的好处（以及潜在用户的兴奋）。
- 发布时，跑者们等待着获得磁带上的发行版并带到各个企业。

套接字对于任何类型的网络都提供相同的抽象：

- 本地（在同一台机器内）
- 互联网（TCP/IP，UDP/IP）
- 不再使用的东西（OSI，Appletalk，IPX等）

套接字抽象使得在不同类型的网络中实现通信变得更加简单和直观。通过套接字，开发人员可以使用相同的API和概念处理本地网络、Internet 以及其他协议。这种抽象提供了强大的灵活性，使得许多现代网络应用程序能够在不同的环境中正常运行。

套接字（socket）在某种程度上类似于具有文件描述符的文件。在UNIX和类UNIX系统（如Linux）中，套接字和文件都使用文件描述符（file descriptor，简称fd）进行标识和操作。文件描述符是一个整数，代表了操作系统跟踪打开的文件和套接字的方法。套接字具有的file descriptor特征如下：

- 对应于网络连接（两个队列），send ，receive队列。
- write操作将数据添加到输出队列（目标是另一端的数据队列），
- 而read操作则从输入队列（目标是这一端的数据队列）中移除数据。
- 一些操作可能不起作用，例如lseek。

套接字和文件在许多I/O操作方面具有相似之处，例如可以使用`read()`和`write()`函数进行读写操作。不过，在套接字上有一些特定的操作，如`connect()`, `bind()`, `listen()`, 和`accept()`，它们主要用于建立和管理网络连接。总的来说，套接字确实与带文件描述符的文件类似，但套接字专注于网络通信，而普通文件则用于本地磁盘上的数据存储：

为了让套接字支持实际应用程序，我们可以采用以下方法：

分块消息处理：

- 尽管双向字节流本身并不具有太大的用途，但可以使用消息处理机制（protocol中的语法做的事情）将字节流划分成多个块。
- 这样，应用程序可以在网络通信中发送和接收结构化的消息，而不仅仅是简单的字节流。

远程过程调用（RPC）机制：

- 通过RPC，可以在不同的环境之间进行转换并在网络上实现抽象的函数调用。
- RPC建立在客户端-服务器通信模型基础之上：客户端通过网络向服务器发送请求，服务器收到请求后执行相应的函数或方法，最后将结果返回给客户端。
- 这使得分布式应用可以更好地协同工作，同时隐藏底层通讯细节。
- 使用套接字、消息机制和RPC，应用程序可以实现低延迟、高并发、跨平台的通信，简化开发者处理网络协议和浏览底层细节的负担。

## 5.4 Simple Example: Echo Server（细化与内核交互流程）

![image-20230821153101940](https://s2.loli.net/2023/08/21/KLORFzhJdbgDy1f.png)

Socket本质上是一个文件描述符。在Unix/Linux的设计哲学中，一切皆文件，包括Socket。因此，Socket在内核中确实有对应的打开文件描述条目。

每个Socket在创建后，无论使用的是TCP协议还是UDP协议，都会创建自己的接收缓冲区和发送缓冲区。每个TCP Socket在内核中都有一个发送缓冲区和一个接收缓冲区。这些缓冲区是由内核管理的，用户空间的程序无法直接访问这些缓冲区。

当我们调用`write()`或`send()`函数时，数据并不会立即发送到网络，而是首先被拷贝到内核的发送缓冲区中。同样，当我们调用`read()`或`recv()`函数时，也是从内核的接收缓冲区中读取数据，而不是直接从网络中读取。

因此，虽然我们不能直接访问这些缓冲区，但我们可以通过系统调用（如`write()`, `read()`, `send()`, `recv()`等）来间接地读写这些缓冲区。这样可以确保数据传输的安全性和可靠性，同时也简化了网络编程的复杂性。

```c
// code at client side
void client(int sockfd) {
    int n;
    char sndbuf[MAXIN]; char rcvbuf[MAXOUT];
    while (1) {
        fgets(sndbuf,MAXIN,stdin); /* prompt */
        write(sockfd, sndbuf, strlen(sndbuf)+1); /* send (including null terminator) */
        memset(rcvbuf,0,MAXOUT);                /* clear */
        n=read(sockfd, rcvbuf, MAXOUT);          /* receive */
        write(STDOUT_FILENO, rcvbuf, n); /* echo */
    }
}
// code at server side
void server(int consockfd) {
    char reqbuf[MAXREQ];
    int n;
    while (1) {                   
        memset(reqbuf,0, MAXREQ);
        len = read(consockfd,reqbuf,MAXREQ); /* Recv */
        if (n <= 0) return;
        write(STDOUT_FILENO, reqbuf, n);
        write(consockfd, reqbuf, n); /* echo*/
    }
}
```

![image-20230821155215349](https://s2.loli.net/2023/08/21/SHfA9FZavRMXYwq.png)

当client端内核中的write buffer满了以后，会触发发送机制，当数据到达server的receive buffer后，会唤醒server端的read进程，之后读取数据到用户态缓冲区，进行处理即可。

当server端内核中的write buffer满了以后，会触发发送机制，当数据到达client的receive buffer后，会唤醒client端的read进程，之后读取数据到用户态缓冲区，进行处理即可。

## 5.5 What Assumptions are we Making?

在处理文件和套接字（特别是TCP套接字）等I/O操作时，我们通常会做以下几个假设：

可靠性：

- 写入文件：读取文件时可以获取刚刚写入的内容，不会丢失数据。
- 写入TCP套接字：在对方进行读取操作时会收到刚刚写入的内容，与文件操作类似。
- 类似于管道（pipes）。

按顺序（有序数据流）：

- 当先写入X再写入Y时，读取操作首先得到X，然后得到Y。

何时准备好？

- 当读取文件时，可立即获取文件在当前时间的内容。这就假设了写入操作已经完成。
- 如果数据尚未到达，读取操作可能会阻塞，直到有数据可供读取。
- 同样，这与管道非常类似。

我们通过这些假设来简化处理文件、TCP套接字和管道等I/O操作的方式。这些假设帮助我们确保操作的顺序和可靠性，从而为硬件和操作系统层面的细节提供了高度抽象。

# 6. Socket Implement at C

## 6.1 Socket Creation

文件系统在结构化的命名空间中提供一组永久对象：

- 进程执行打开、读/写/关闭操作
- 文件独立于进程存在
- 轻松指定要打开的文件()

管道（`pipes`）：同一（物理）机器上的进程间单向通信

- 单个队列
- 通过调用`pipe()`临时创建
- 从父进程传递给子进程（所有的文件描述符继承自父进程）

套接字（`sockets`）：在相同或不同机器上的进程间双向通信

- 两个队列（每个方向一个）
- 进程可以在不同的机器上：无共同祖先
- 我们如何命名我们要打开的对象？
- 这些完全独立的程序如何知道其他程序想要与它们“通话”？

## 6.2 Namespaces for Communication over IP

主机名

- [www.eecs.berkeley.edu](http://www.eecs.berkeley.edu/),一个IP地址就是一个命名空间。一般标识了一台机器。

IP地址

- 128.32.244.172（IPv4，32位整数）
- 2607:f140:0:81::f（IPv6，128位整数）

端口号

- 0-1023 是“众所周知”的或“系统”端口
  - 需要超级用户权限来绑定一个
- 1024-49151 是“注册”端口（注册表）
  - 由 IANA 分配给特定服务
- 49152-65535（2^15^ + 2^14^ 到 2^16-1）是“动态”或“私有”的
  - 自动分配为“临时端口”

在 IP 通信中，组合主机名或 IP 地址和端口号可以实现对特定套接字的唯一标识。在通信过程中，系统会将 IP 地址和端口号组合成一个称为“套接字地址”的结构，该结构用于建立连接、监听来自其他进程的连接请求以及发送/接收数据。这些命名空间提供了一种方法，使得独立的进程能够在网络中找到并与彼此通信。

下面是对`hostname`、`IP Address`、`port number`的进一步具体说明：

1. 主机名：主机名是一个易于读取和理解的标识，用于在互联网上找到某个计算机。主机名通过 DNS（域名系统）转换为实际的 IP 地址，计算机才能够实际连接和交互。
2. IP 地址：具有唯一性，用于在 Internet 上标识每个设备。它们分为 IPv4 和 IPv6 两种类型。由于可用 IPv4 地址的不断减少，IPv6 地址（更大的地址空间）被设计用于替换 IPv4 以满足互联网的扩展需求。
3. 端口号：行为类似于房间号码。在一栋具有独特地址的大楼中（一个大楼就是一个IP地址，代表了一台机器），每个房间都有自己的房间号码以保持区别。同样，每个运行在计算机上的进程都需要具有一个唯一的端口号，以便与其他进程区分开来。端口范围从 0 到 65535。如前所述，它们分为三种类型：“众所周知的”、“注册的”和“动态的”。

了解这些命名空间的详细信息将有助于您编写更高效的网络代码和设计网络应用程序，以便它们在 Internet 上正常运行。为了确保通信的可靠性和数据的安全传输，您还需要考虑以下几个方面：

1. 传输协议：TCP（传输控制协议）和 UDP（用户数据报协议）是两种主要的传输层协议，它们负责在两个设备之间建立连接并传输数据。您需要了解每种协议的优点和缺点，以便根据网络应用程序的需求正确选择。
2. 加密和安全性：为了保护网络通信免受黑客攻击和数据泄露，加密和安全措施（例如使用 SSL/TLS 协议）至关重要。尤其是对于涉及传输敏感数据（如生物识别数据、信用卡信息等）的应用程序。

通过扩展您在主机名、IP 地址和端口号方面的知识，以及对传输协议和安全标准的了解，您将能够在网络编程领域具备更扎实的基础。这将使您能够开发可靠的、安全的且高性能的网络应用程序。

## 6.3 Connection Setup over TCP/IP(C/S 两端的socket有些许区别)

![image-20230822185901097](https://s2.loli.net/2023/08/22/Mm2pjBrwFnl61cV.png)

特殊类型的套接字：`Server Socket`

- 具有文件描述符
- 不能读或写

两个操作：

1. `listen()`：开始允许客户端连接
2. `accept()`：为特定客户端创建一个新套接字

服务器套接字用于监听来自客户端的连接请求。它作为中间代理，处理客户端发起的连接请求并在接受连接时创建一个新的套接字。这种设置使服务器能够同时处理多个客户端连接。

服务器套接字通常需要进行以下配置和操作

1. `socket()`：创建服务器套接字。
2. `bind()`：将套接字绑定到特定的 IP 地址和端口号。
3. `listen()`：使服务器套接字开始监听客户端连接。它指定一个队列长度，队列中可以存放等待连接的客户端个数。
4. `accept()`：等待并接受来自客户端的连接。当连接建立时，会为该特定客户端创建一个新的套接字。这个新套接字充当服务器与客户端之间的连接通道，用于读写数据。

需要注意的是，在应用程序的工作流程中，接受连接（accept）通常放在一个循环中，以便不断接受和处理来自多个客户端的连接请求。这两个操作（listen 和 accept）是在服务器端编程中非常关键的，因为它们能够允许服务器与多个客户端建立稳定、可靠的连接通道，从而支持在网络中进行高性能、安全的通信。

在网络通信中，5元组用于识别每个连接（在连接建立后，客户端会得到一个这样的5元组）：

1. 源 IP 地址`Source IP Address`
2. 目标 IP 地址`Destination IP Address`
3. 源端口号`Source Port Number`
4. 目标端口号`Destination Port Number`
5. 协议（在这里始终为 TCP）`Protocol`

**通常，客户端的端口是“随机”分配的**

- 在客户端套接字设置过程中由操作系统完成
- 这不难理解，我们可以打开一个浏览器，打开多个相同的网页，这意味着我们在client（浏览器）上对同一个服务器发起了多个连接请求，每个页面的socket绑定的端口都是不同的。所以我们可以同时打开多个同一个网页，比如打开了8个cs162的官网页面。

服务器端口通常是“众所周知”的

- 80（网络），443（安全网络），25（sendmail）等
- 众所周知的端口范围从 0—1023

一个 5-元组可以在给定协议下唯一标识一个网络连接，这对于确保数据正确传输至预期的目的地及避免网络拥塞至关重要。这些元组对于建立进程之间的通信通道以及确定网络流量来源和目的地非常有用。

在某些情况下，如 NAT（网络地址转换），端口号也起到区分多个来源或目标设备的关键作用。这对于在有限的 IP 地址资源下实现多个设备共享互联网访问至关重要。

为了确保网络应用程序的可靠性和性能，您需要了解这些 5元组，因为它们在网络与程序交互中扮演着关键角色。在某种程度上，它们为客户端和服务器提供了通信的逻辑门户，处理数据在指定协议下传输的方式。

## 6.4 Sockets in concept

![image-20230822190755638](https://s2.loli.net/2023/08/22/yLspcilqZWUwbfC.png)

## 6.5 Client Protocol

```c
char *host_name, *port_name;
// Create a socket
struct addrinfo *server = lookup_host(host_name, port_name);
int sock_fd = socket(server->ai_family, server->ai_socktype,
server->ai_protocol);
// Connect to specified host and port
connect(sock_fd, server->ai_addr, server->ai_addrlen);
// Carry out Client-Server protocol
run_client(sock_fd);
/* Clean up on termination */
close(sock_fd);
```

上述代码示例展示了如何在客户端程序中创建套接字并连接到指定的主机和端口：

1. 首先，定义两个字符串指针变量 `host_name` 和 `port_name`，用于存储目标服务器的主机名和端口号。
2. 创建一个 `addrinfo` 结构的指针，称为 `server`，并通过调用 `lookup_host()` 函数（尚未在这里提供，但可以自行实现或使用类似 getaddrinfo() 函数）连接到指定的主机名和端口号。
3. 调用 `socket()` 函数创建套接字。参数包括从 `server` 的 `addrinfo` 结构中获取的地址族（`server->ai_family`）、套接字类型（`server->ai_socktype`）和协议（`server->ai_protocol`），创建的套接字文件描述符存储在 `sock_fd` 变量中。
4. 使用 `connect()` 函数连接到指定的主机和端口。参数包括套接字文件描述符 `sock_fd`、服务器的地址（`server->ai_addr`）和服务器地址长度（`server->ai_addrlen`）。
5. 调用自定义的 `run_client()` 函数来执行客户端与服务器之间的通信协议。在此，向 `run_client()` 函数传递 `sock_fd` 以使用已建立的连接。
6. 在通信完成时，使用 `close()` 函数关闭套接字以进行清理。

通过这个代码示例，您可以了解如何在客户端程序中创建和使用套接字以连接到远程服务器。在实际应用中，您可能需要处理异常情况，并根据具体的客户端和服务器协议实现 `lookup_host()` 和 `run_client()` 函数。这段代码为您提供了一个基本的客户端套接字设置示例，以便您开始编写网络应用程序并使其与远程服务器进行通信。

## 6.6 Server Protocol (v1)

```c
// Create socket to listen for client connections
char *port_name;
struct addrinfo *server = setup_address(port_name);
int server_socket = socket(server->ai_family,
server->ai_socktype, server->ai_protocol);
// Bind socket to specific port
bind(server_socket, server->ai_addr, server->ai_addrlen);
// Start listening for new client connections
listen(server_socket, MAX_QUEUE);
while (1) {
    // Accept a new client connection, obtaining a new socket
    int conn_socket = accept(server_socket, NULL, NULL);
    serve_client(conn_socket);
    close(conn_socket);
}
close(server_socket);
```

上述代码示例展示了在服务器端程序中创建套接字以监听并接受客户端连接的过程：

1. 定义一个字符串指针变量 `port_name`，用于存储服务器监听的端口号。
2. 创建一个 `addrinfo` 结构的指针，称为 `server`，并通过调用 `setup_address()` 函数（需要您自行实现或使用类似的 getaddrinfo() 函数）设置要绑定的端口。
3. 调用 `socket()` 函数创建服务器套接字。参数包括来自 `server` 的 `addrinfo` 结构的地址族（`server->ai_family`）、套接字类型（`server->ai_socktype`）和协议（`server->ai_protocol`），创建的套接字文件描述符存储在 `server_socket` 变量中。
4. 使用 `bind()` 函数将套接字绑定到特定端口。参数包括服务器套接字文件描述符 `server_socket`、服务器地址（`server->ai_addr`）和服务器地址长度（`server->ai_addrlen`）。
5. 使用 `listen()` 函数开始监听新的客户端连接。参数包括服务器套接字文件描述符 `server_socket` 和客户端连接请求的最大队列长度（由 `MAX_QUEUE` 定义）。
6. 使用一个无限循环（`while (1)`）来持续接受新的客户端连接。为每个新连接调用 `accept()` 函数，创建一个新的套接字（在服务器端），其文件描述符存储在 `conn_socket` 变量中。
7. 对于每个连接的客户端，调用自定义的 `serve_client()` 函数以处理客户端请求。将 `conn_socket` 传递给此函数，以便在建立的通信通道上进行操作。
8. 在为客户端提供服务后，使用 `close()` 函数关闭连接套接字以进行清理。
9. 在所有客户端处理完毕后，使用 `close()` 函数关闭服务器套接字。

这段代码为您提供了一个基本的服务器端套接字设置示例，在此基础上，您可以根据特定的客户端和服务器协议实现 `setup_address()` 和 `serve_client()` 函数。考虑到异常情况和错误处理，这样的实现将更加健壮，为编写网络应用程序提供了一个良好的起点。

## 6.7 Sockets With Protection (each connection has own process)

服务器可以通过以下方法保护自己：

- 在单独的进程中处理每个连接。

![image-20230822195027738](https://s2.loli.net/2023/08/22/4Kbiw786Zh3jLuA.png)

处理每个连接的一种方法是为每个客户端创建一个新的进程，在新进程中处理客户端请求。这样，如果某个客户端的进程出现问题（如意外崩溃或攻击），它不会影响到其他客户端的进程或整个服务器。

如上图所示，当serverr监听到一个连接请求后，他会fork()一个子进程，让该子进程专门来处理与client的连接。

所以，父进程本身用来监听client的连接请求，不负责处理具体的连接；子进程负责专门处理与客户端的连接，不负责监听请求。因此就有了父进程关闭`Conn Socket,`子进程关闭`Listen Socket`的动作。目前只是单进程处理连接，非常类似于Pipe的机制，Pipe也是在父子两端都需要各关闭一个端口，这个端口正好是对方使用的。

```c
// Socket setup code elided…
while (1) {
    // Accept a new client connection, obtaining a new socket
    int conn_socket = accept(server_socket, NULL, NULL);
    pid_t pid = fork();
    if (pid == 0) {
        close(server_socket);
        serve_client(conn_socket);
        close(conn_socket);
        exit(0);
    } else {
        close(conn_socket);
        wait(NULL);
    }
}
close(server_socket);
```

## 6.8 Concurrent Server

到目前为止，在服务器中：

- `listen()` 函数将对请求进行排队
- 缓冲区存在于其他地方
- 但服务器在处理下一个连接之前需要等待每个连接终止

一个并发服务器可以在之前的客户端断开连接之前处理并服务一个新的连接。有多种方法可以实现这种并发服务器，其中两种常见方法是使用多进程（有保护、创建开销大）、多线程（无保护、但轻量）和异步 I/O。这里只讨论多进程的情况。

并发版本的就是在每个连接都有自己的进程的基础上，让parent进程不用等待当前子进程完成，可以不断地去处理client请求，不断地创建新的子进程：

![image-20230822201349793](https://s2.loli.net/2023/08/22/Jxrl9OKmUIMk1Qg.png)

代码里就是把6.7中的父进程执行逻辑中的`wait(NULL)`注释掉即可。

# 7. Some tips about socket

## 7.1 Server Address: Itself

```c
struct addrinfo *setup_address(char *port) {
    struct addrinfo *server;
    struct addrinfo hints;
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE;
    getaddrinfo(NULL, port, &hints, &server);
    return server;
}
```

这个 `setup_address()` 函数设置并返回一个用于服务器端套接字的 `addrinfo` 结构。它接受指定端口上的任何连接。以下是该函数的详细说明：

1. 定义一个 `addrinfo` 结构指针 `server`，用于存储服务器地址信息。
2. 定义一个 `addrinfo` 结构变量 `hints`，用于设置服务器地址的属性。
3. 使用 `memset()` 函数将 `hints` 结构的所有字节初始化为零。
4. 设置 `hints.ai_family` 为 `AF_UNSPEC`，表示服务器套接字可以接受 IPv4 或 IPv6 地址。
5. 设置 `hints.ai_socktype` 为 `SOCK_STREAM`，表示服务器将使用面向连接的传输协议（如 TCP）。
6. 设置 `hints.ai_flags` 为 `AI_PASSIVE`，表示服务器套接字将绑定到通配符地址，以便能够接受来自任何地址的连接。
7. 调用 `getaddrinfo()` 函数，传入 `NULL`（表示通配符地址）、端口号 `port`、设置好的 `hints` 结构以及一个指向 `server` 的指针。该函数将分配并填充一个适用于服务器地址的 `addrinfo` 结构。
8. 返回分配给 `server` 的 `addrinfo` 结构。

通过将这个 `setup_address()` 函数与上述服务器代码示例结合，您现在可以在指定端口上接受来自任何地址的客户端连接，设置一个适用于服务器端程序的套接字。在使用 `setup_address()` 函数后，请确保在不再需要时使用 `freeaddrinfo()` 函数释放 `addrinfo` 结构所占用的内存。

## 7.2 Client: Getting the Server Address

```c
struct addrinfo *lookup_host(char *host_name, char *port) {
    struct addrinfo *server;
    struct addrinfo hints;
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    int rv = getaddrinfo(host_name, port_name,
    &hints, &server);
    if (rv != 0) {
        printf("getaddrinfo failed: %s\n", gai_strerror(rv));
        return NULL;
    }
    return server;
}
```

在这个代码示例中，提供了一个 `lookup_host()` 函数。此函数返回一个用于解析特定主机名和端口的服务器地址的 `addrinfo` 结构。

1. 定义一个 `addrinfo` 结构指针 `server`，用于存储服务器地址信息。
2. 定义一个 `addrinfo` 结构变量 `hints`，并设置服务器地址的属性。
3. 使用 `memset()` 函数将 `hints` 结构的所有字节初始化为零。
4. 将 `hints.ai_family` 设置为 `AF_UNSPEC`，表示套接字可以接受 IPv4 或 IPv6 地址。
5. 将 `hints.ai_socktype` 设置为 `SOCK_STREAM`，表示服务器将使用面向连接的传输协议（如 TCP）。
6. 调用 `getaddrinfo()` 函数，传入主机名 `host_name`、端口号 `port`、设置好的 `hints` 结构以及一个指向 `server` 的指针。该函数将分配并填充一个适用于服务器地址的 `addrinfo` 结构。
7. 如果 `getaddrinfo()` 函数返回值 `rv` 不为0, 则打印错误信息并返回 NULL。
8. 返回分配给 `server` 的 `addrinfo` 结构。

对于客户端，您可以使用此 `lookup_host` 函数来获取服务器的地址，然后使用该地址创建并连接到服务器的套接字。通过这种方式，客户端可以方便地连接到服务器并进行通信。

## 7.3 Concurrent Server without Protection(more lightful)

通过为每个连接创建一个新线程来处理，客户端和服务器通信的效率可以得到提高。主线程在不等待先前生成的线程的前提下发起新的客户端连接。那么，为什么要放弃独立进程带来的保护呢？原因如下：

1. 创建新线程更高效：与创建新进程相比，线程创建和销毁的开销较小。进程需要更多的系统资源，如内存和处理器时间，而线程共享进程的虚拟内存空间和系统资源，因此它们保持启动和运行所需的资源更小。
2. 线程之间的切换更高效：在同一个进程内的线程之间进行上下文切换所需的时间和资源比进程之间切换要少得多。线程之间共享相同的地址空间，因此上下文切换更快。而进程之间的上下文切换涉及更多的开销，如内存管理和进程间通信。

然而，线程模型也存在一些潜在的问题。例如：

1. 线程间资源共享导致竞争条件：由于线程共享内存、文件描述符等资源，它们可能在访问共享资源时发生竞争。因此，需要采取适当的同步机制，如互斥锁或信号量，以确保数据在多线程环境中的正确性和一致性。
2. 缺乏进程级的保护：与独立进程相比，线程级别的保护较弱。如果一个线程在运行过程中错误或崩溃，这可能会影响其他线程或整个进程。独立进程可以提供更好的故障隔离。

考虑到这些利弊，根据具体的需求和场景，选择合适的方式（进程、线程或异步）来管理客户端连接非常关键。通常情况下，在需要处理大量并发连接且希望减少系统资源开销的情况下，线程和异步I/O 是更合适的选择。

![image-20230822204136284](https://s2.loli.net/2023/08/22/M2OG8naI3zFvfeN.png)

与上面多进程的区别就是不用fork()了，用pthread_create()来创建线程，主线程用来监听客户端请求，衍生的线程则用来处理不同的客户端的连接。

但是如果客户端请求太多，创建了太多的线程，那么可能会导致OS崩溃，所以需要某些方法来限制线程的数量。

## 7.4 Thread Pools（限制线程创建数量）

在前面提到的线程模型中，问题在于线程数量没有限制。

- 当网站访问量过大时，无限创建线程会导致吞吐量下降和系统资源耗尽。为了解决这个问题，可以使用线程池来限制并发线程的数量，从而实现更可控的多任务并发。

线程池是一种管理执行多个任务的方法，它包含一个有限数量的线程。这些线程在需要执行任务时被分配，执行完成后则返回到线程池。使用线程池的好处有：

1. 限制并发线程数：线程池能够限制系统资源的消耗，并避免由于线程数量过多而导致的性能下降。
2. 提高资源利用率：线程池可以复用已创建的线程，避免了频繁创建和销毁线程带来的开销。
3. 提高响应速度：线程池中的空闲线程可以立即执行新的任务，无需等待线程创建过程。
4. 统一管理任务执行：线程池可对工作线程进行统一管理和调度，便于实现更高级的功能，如负载均衡、优先级任务等。

为了在服务器中使用线程池，可以选择多种现成的线程池库，如 C++ 的 Boost.Asio, 或 C 的 libuv 等。

![image-20230822204720161](https://s2.loli.net/2023/08/22/6vxFbnwNE58GcIs.png)

# 8. Conclusion

进程间通信（IPC）：

- IPC 是在受保护环境（即进程）之间进行通信的一种设施。

管道（Pipes）：

- 管道是一个单队列抽象。一端只能写数据，另一端只能读数据。
- 它们用于在同一台机器上的多个进程之间进行通信。
- 通过继承来获取文件描述符。（fork()或别的创建方式）
- 在fork()方式中，一旦确定了父子进程的读写角色，就需要在双方的进程中关闭自己不用的那个管道口的文件描述符。

套接字（Sockets）：套接字是两个队列的抽象，一个队列负责每个方向的通信。

- 每端都可以进行读写操作。
- 它们用于在不同机器上的多个进程之间进行通信。
- 通过 socket/bind/connect/listen/accept 获取文件描述符。
- 通过 fork() 继承文件描述符，方便在独立进程中处理每个连接。

两者都支持 read/write 系统调用，就像文件 I/O 操作一样。

通过理解这些基本概念，您可以更好地为您的应用程序选择适当的进程间通信（IPC）方法。管道和套接字可以实现不同场景下的资源共享和数据交换需求。在进程管理方面，线程和进程可以根据应用需求，实现不同程度的并发处理。通过适当选择和组合这些概念，可以满足各种应用程序中的通信、资源管理和性能需求。
