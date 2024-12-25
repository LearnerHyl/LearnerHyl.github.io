# 1. OS Intro

## 1.1 OS can do

将令人难以置信的底层技术进步应用于迅速发展的应用领域：
- 为应用程序提供一致的抽象，即使在不同的硬件上也是如此。
- 管理多个应用程序之间的资源共享。

主要的构建模块包括：
- 进程
- 线程、并发、调度、协调
- 地址空间
- 保护、隔离、共享、安全
- 通信、协议
- 持久存储、事务、一致性、弹性
- 与所有设备的接口

## 1.2 OS can do

最有可能包括的：
- 内存管理
- I/O管理（输入/输出管理）
- CPU调度
- 通信？（电子邮件是否属于操作系统？）
- 多任务处理/多程序处理

还包括：
- 文件系统
- 多媒体支持
- 用户界面
- 互联网浏览器 （这个目前有争议）

## 1.3 The define of OS

没有一个普遍被接受的定义。

“当你订购一个操作系统时供应商交付的一切”是一个很好的近似定义，但实际上可能差异很大。

“在计算机上始终运行的那个程序”指的是内核（`kernel`）。

- 其他所有内容要么是系统程序（随操作系统一起提供的）要么是应用程序。

我们关注的是OS的作用和OS有哪些重要的组成部分，也许我们永远不会得到一个关于OS的准确定义。

操作系统是一种特殊的软件层，为应用程序提供对硬件资源的访问：

- 提供对复杂硬件设备的便捷抽象
- 保护共享资源的访问
- 提供安全性和身份验证功能
- 支持通信功能

操作系统为上层的应用程序提供一个统一的接口，使得开发者可以更方便地使用计算机的硬件功能，而不需要了解底层复杂的硬件细节。同时，操作系统也负责保护不同应用程序之间的资源共享，确保它们能够安全地访问共享的资源，同时保障系统的稳定性和安全性。此外，操作系统还提供了通信机制，使得应用程序之间可以进行数据交换和信息传递。

何时能将一个事务视为一个系统呢：

- 多个相互关联的部分：系统由多个组成部分组成，这些部分之间可能相互作用和影响。
- 健壮性需要工程思维：确保系统具有高可靠性需要工程师的思维。这包括精确的错误处理，防范对恶意和粗心用户的攻击，以及将计算机视为一个具体的机器，考虑到其所有的限制和可能的故障情况。

在设计和构建系统时，重要的是考虑各个部分之间的相互作用，以及如何处理各种可能的问题和失败情况。一个成功的系统需要有严格的设计和实施，以确保其稳定性、安全性和高性能。

OS可以将底层的硬件细节掩盖，将底层抽象为一个与具体硬件无关的统一接口，方便编写应用程序。

# 2. OS basics/ The roles of OS

## 2.1 Illusionist（幻术师）

"幻术师"是指系统提供对物理资源的清晰、易于使用的抽象：

- 无限的内存，仿佛拥有专用的计算机
- 提供更高级的对象：文件、用户、消息
- 掩盖底层的限制，实现虚拟化

在计算机科学中，系统可以通过创建抽象层来隐藏底层的物理硬件细节，使应用程序和用户能够以更简单、更高级的方式与系统进行交互。这种抽象有助于简化开发和使用过程，让用户感觉好像有无限的资源可供使用，同时提供了一种虚拟化的体验。例如，虚拟内存将主存储器和硬盘空间组合成一个看似无限大的内存空间，以满足应用程序的需求。这些抽象让计算机系统的使用变得更加方便和高效。

![image-20230725195236200](https://s2.loli.net/2023/07/25/hMq5btfdlJiex2Q.png)

看上图，ISA（Instruction Set Architecture）是指令集体系结构。在硬件层面，处理器（CPU）经过OS抽象，向上层提供了线程；内存经过OS抽象，向上层提供了一个统一的地址空间（而不是分散的字节地址）；存储部分经过OS抽象，向上层提供了一个个的文件（而不是一个个的blocks）；网络部分经过抽象，向上层提供了供通信使用的Sockets。

再往上，OS通过结合硬件提供的抽象，OS向上层提供了一个进程（Process）抽象（由操作系统提供的带有受限权限的执行环境）。<u>可以把进程看作一个虚拟机，你的应用程序可以以进程为载体运行。</u>

再向上就到了应用程序的级别了，有系统库，编译好的程序等等。

![image-20230725200352997](https://s2.loli.net/2023/07/25/9NEiqGnLVaKoCH2.png)

应用程序的“机器”是由操作系统提供的进程抽象，即每个应用程序都感觉自己有一个独有的机器。

每个正在运行的程序都在自己独立的进程中运行。

进程提供比原始硬件更友好的接口。

### 2.1.1 What's in a Process

一个进程由以下组成：

- 地址空间：进程拥有自己的地址空间，用于存储程序、数据和堆栈等信息，本质上是一块受保护的内存。
- 一个或多个控制线程：在进程的地址空间内执行的一个或多个控制线程，用于执行程序中的指令。
- 与之相关联的额外系统状态：
  - 打开的文件：进程可能有打开的文件，允许它读取或写入文件内容。
  - 打开的套接字（网络连接）：进程可能会建立网络连接，用于与其他计算机或服务进行通信等。

进程是操作系统对正在运行的程序的抽象，它提供了独立的执行环境，使得每个程序都可以在自己的进程中运行，相互之间不会相互干扰。每个进程拥有自己的资源，例如内存空间和系统状态，这使得操作系统能够高效地管理系统资源，同时确保进程之间的隔离和安全性。

### 2.1.1 The Multi-Process view of OS

对于OS来说，多进程的视图在他们眼中是这样的：

![image-20230727112224526](https://s2.loli.net/2023/07/27/815p4n92LJMs3aB.png)

操作系统将硬件接口转换为应用程序接口，为每个正在运行的程序提供独立的进程，进程之间是相互隔离的。

每个进程都有自己的一组线程、地址空间、文件、套接字。

## 2.2 Referee（裁判员）

裁判员(`Referee`)：主要负责处理资源的保护、隔离和共享等问题。

- 还有资源分配和通信：这是裁判员系统的基本功能。它决定如何在多个任务和用户之间分配资源（如计算能力、存储和网络带宽），以及处理任务间的通信（例如数据传输和消息传递）。 裁判员系统需要在以下几个方面表现出优越性： 

1. 资源保护：确保每个任务或用户的分配不会受到其他任务或用户的影响，从而保证了资源的隔离性。 操作系统将进程相互隔离，操作系统将自身与其他进程隔，即使它们实际上在同一硬件上运行！
2. 隔离性：在程序间、用户间或任务间实现资源的高度隔离，以确保单个任务的失败不会影响其他任务。 
3. 资源共享：鉴于资源有限，系统需要采取策略在任务和用户之间公平地分配资源，实现高效的资源共享。 
4. 灵活性与可扩展性：系统应能够在大范围内适应不同大小和类型的资源需求，并能按需分配和调整资源，以满足多样化的计算需求。 
5. 系统性能：为了提供良好的用户体验，裁判员系统在资源管理和通信方面需要表现出高性能。 

总之，裁判员系统的核心任务是在多任务和多用户的环境中实现资源保护、隔离和共享。为了达到这一目的，它们需要在资源分配、通信和隔离等方面表现出高效的性能。

## 2.3 Glue（粘合剂）

在操作系统中，有一些常见的服务充当胶水-`Glue`，将各种系统组件和功能统一起来。这些基础服务包括：

1. 存储（`Storage`）：操作系统提供统一的存储服务，管理与访问硬盘、固态硬盘以及其他存储设备。存储服务包括文件系统管理、空间分配等功能。
2. 窗口系统（`Windows system`）：操作系统提供窗口系统来管理图形用户界面，包括窗口、菜单、按钮等可视元素。窗口系统使得用户与计算机之间的交互变得更为直观便捷。 
3. 网络（`Networking`）：操作系统提供网络服务，支持计算机与其他设备或网络进行通信。网络服务包括协议管理、网络设备管理、数据传输等功能。 
4. 共享（`Sharing`）：操作系统提供资源共享服务，允许多个用户或进程访问共享文件、打印机等资源。此外，操作系统还实现了进程间通信（IPC）机制，使进程能够相互之间交换信息。 
5. 授权（`Authorization`）：操作系统负责授权管理，确保用户能够安全地访问受限资源。授权服务涉及用户身份验证、访问控制、权限管理等功能。 
6. 外观与体验（`Look and feel`）：操作系统不仅关注功能和性能，还致力于提供统一、美观且易于使用的界面。通过优化界面元素的布局、外观和交互，操作系统使用户能够更快速、便捷地完成任务。

总之，操作系统通过这些通用服务将不同组件和功能紧密集成起来，帮助用户和开发者更为高效地利用计算资源、进行工作与创新。主流的OS都会有上述的功能，有一些特别应用于某些场景的OS可能不会有上述的全部的服务，但是这不意味着他们就不是OS了，他们依然是OS，只是功能覆盖面没有那么大而已。
