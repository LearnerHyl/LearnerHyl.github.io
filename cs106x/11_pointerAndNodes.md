# Structs and Pointers;Linked Nodes

# Structs

轻量级的class，里面包含了一系列data和具体的行为。成员函数和成员变量默认是public属性。class内部的成员属性默认是private的。

如果我们定义一个class，默认预期的class会很复杂，包含构造函数等一大堆内容；而class只需要简单地有变量，也可以加上几个成员函数，所以class有些必须的内容，对于struct而言可以没有（如构造函数等）。

我们可以在struct内部定义一些方法，如下所示： 

![image-20240221143715629](https://s2.loli.net/2024/02/21/EdXYyaoRUnceGbr.png)

# Pointers

下面这张图能很形象的说明pointers的定义。

![image-20240221150033298](https://s2.loli.net/2024/02/21/EUa5hzp8lkdHiT4.png)

## &和*在指针中的应用

在C/C++中，`&`和`*`操作符有很多用途。下面是一些例子：

`&`操作符用于获取变量的地址：

```c++
int a = 10;
int* p = &a;  // p是一个指向a的指针
```

`*`操作符用于解引用指针，获取指针指向的值：

```c++
int a = 10;
int* p = &a;
int b = *p;  // b的值现在是10
```

指向指针的指针（也就是二级指针）的应用：

```c++
int a = 10;
int* p = &a;
int** pp = &p;  // pp是一个指向指针p的指针
```

在这个例子中，`*pp`将得到`p`（也就是`a`的地址），`**pp`将得到`a`的值（也就是10）。

`&`操作符也用于引用的声明：

```c++
int a = 10;
int& ref = a;  // ref是a的引用，ref和a指向同一块内存
```

`*`操作符还用于定义指针类型的变量：

```c++
int a = 10;
int* p;  // p是一个int类型的指针
p = &a;  // 现在p指向a
```

## Null Pointer

null pointer意味着不指向任何东西，可以用来初始化指针，是指针的默认值或空值。

如果dereference一个null pointer，程序会crash。在C中，人们都写NULL，但是nullptr会更好。

**nullptr意味着指针存储的地址值是0。**一般来说，当我们声明了一个指针却没有给他赋值或让他指向任何内容时，最好赋值为nullptr，这样该指针就不会指向垃圾内容。但注意指针本身仍然需要内存存储，看下图。

![image-20240222134123477](https://s2.loli.net/2024/02/22/78eUJkvBwugPQlq.png)

##  Garbage pointers

如上所说，如果我们声明了一个指针却没有对其进行初始化，那么该指针会指向内存中的一个随机位置，此时该指针就是一个垃圾指针。

- 如果我们尝试解引用该指针指向的地址，程序可能会crash。
- **我们应该总是在指针声明后初始化它。（无值可赋的时候可以给nullptr）**

![image-20240222135402235](https://s2.loli.net/2024/02/22/TOyUMN2PBLih87S.png)

## Pointer to a struct

指针可以指向一个struct或object。为了引用struct中的成员，要写作`ptr->member`。等价于`(*ptr).member`。

![image-20240222140236902](https://s2.loli.net/2024/02/22/sKHrWkV8fLCXR2E.png)

# c++运行时内存分配

下面说的stack和heap本质上都是内存，只是管理的方式有所不同。一个很重要的区别是，**堆和栈在生命周期上是完全无关的**，看完下面内容就会理解是为什么。

在c++中，我们使用`new`关键字来区分我们分配内存的方式：

- 不用`new`关键字，则直接使用栈上的内存。
- 使用`new`关键字，**栈上的对应变量中存储的是指向堆中某块内存的地址**。

## The Stack

运行的程序会将过程中运行函数/变量的信息存到一个栈中。

- 一旦函数return，那么该函数使用的内存空间会被马上释放，即标记为可用。
- 一般来说，为一个程序的信息分配的内存空间都是连续的，本质上是调用栈。

![image-20240222141059338](https://s2.loli.net/2024/02/22/PgzEsXVWywd1hM9.png)

## The heap

堆，这和我们之前讨论的栈是两个独立的内存空间。

我们用`new`关键词分配的内存都在heap上，一般返回一个指向该地址的`Pointer`。**注意，除非我们主动释放堆上的内存，否则在heap上分配的内存会一直在那，**即使程序已经完全退出。

- 如下例，a和c都用了new关键字，栈中存放的是指向实际内容的堆中的内存地址，当函数返回时，a和c变量本身会被释放，如果我们不主动释放他们使用的堆内存，那两块堆内存会一直存在在那，且无法被再次分配，除非有某种方式记录下这两块堆内存的地址。

![image-20240222142301136](https://s2.loli.net/2024/02/22/m3VcHjalGnZtzJh.png)

## new关键字

直接上图，更加清晰，注意new返回的是申请内存的**地址**。用`Delete`关键字回收内存。此外，对于数组和单个指针，Delete的语法是不同的。

![image-20240222150058562](https://s2.loli.net/2024/02/22/oDaf1OHvXGVzCJx.png)

```c++
int* a = new int; // 单个指针
delete a;

int* b = new int[10]; // 申请一串内存
delete[] b; // c++有某种方式可以知道当前数组b的大小，进行删除，[]是为了区分一串内存与一块内存
```

## *和&的区别()

如此一来，*和&的区别就很明显了，\*更加的灵活，指针变量存储的是地址，而这个地址是位于堆中的。&变量本质上引用的仍然是我们声明在栈上的局部变量。因此指针可以进行跨函数使用，因为他被分配的内存空间的生命周期只取决于我们何时释放他，而与具体某个函数本身的生命周期无关。如果我们忘记释放某块内存，那么只有等到整个程序运行结束后，这块未释放的内存才会被释放。

**我们调用delete，清理的是指针指向的那块堆内存，而不是指针本身所在的内存空间，指针本身在栈上，会在当前函数返回之后自动被系统回收，因此delete的对象只能是内存地址，且一般是堆上的内存地址。**

> 因此就出现了smart pointers，这些指针的内存不必我们亲自释放，在运行期间，当没有指针变量指向smart pointer申请的内存时，这块内存会被马上回收。

```c++
int *a = new int(3);
delete(a); //主动释放a存储的内存地址代表的那块内存
// 变量a本身在栈上，会在当前函数结束后被自动回收。
```

我们在c++库中使用的那些vector、set等STL为什么可以在函数之间来回传递，即使返回了也没有释放内存，因为STL内部的实现中，本质上也是用了new在堆上申请了一块内存，由STL内部代我们完成了内存管理工作，我们只需要使用它即可。
