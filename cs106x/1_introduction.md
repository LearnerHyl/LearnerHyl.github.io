# 1. introduction

106X是106B的加强版，106L更偏向于实践操作。注意代码风格的简洁性、功能的正确性。106X的官网有一份StyleGuide可以参考一下正确的代码风格是什么。

Qt Creator是本节课用的开发环境，课程网站上有如何配置的教程，课程主页点击"Qt Creator"这个link即可。

课程作业都是以Qt Creator项目的形式打包发下来，所以尽量不要使用别的IDE。

# 2. C++ programs/files

C++的程序在.cpp文件中，cpp文件会与起辅助作用的`.h`或`.hpp`头文件进行交互。

![image-20230607171200923](https://s2.loli.net/2023/06/07/BbskvTWgE3UntxQ.png)

Java文件被编译后，最终会得到一个class文件，这个`class文件`就是程序的二进制版本，一个编译后的程序版本。同时这个class文件也是平台独立（`platform independent`）的二进制文件，这意味着你可以在Windows、Mac、Linux等不同的OS平台上直接运行这个class文件。

<u>但是C++有一些不同。</u>cpp文件被`complie`后，首先生成`.o文件`（object file），这些文件是平台相关的，他们只有在你的OS上才能正常运行。在生成`.o文件`后，可以针对其中的一个或多个`.o文件`，把他们链接(`link`)为一个可执行文件（`executable file`）。<u>这个文件在windows上有.exe扩展名，在别的系统上则没有。因此最后`link`操作生成的可执行文件也是平台相关的。</u>

> complie是我们点击了Build按钮后执行的操作。如上图所示，link的过程中会链接到很多第三方库。

C语言生成可执行文件的流程也是和c++一样的，<u>因为C++的设计者希望c++能向后兼容C语言。</u>

# 3. The main function

无论是Java还是C++，我们都会使用某些method或function去执行我们编写的程序。这两个名词指的是同一类事物，只是称呼方式不同，在Java中称为`method`,在c++中称为`function`。在Java中，我们的main函数位于Public class中，java中把所有的方法与函数都封装到class中。但是C++没有这么做。

![image-20230608114121626](https://s2.loli.net/2023/06/08/gS39fOQzZAjKUlY.png)

# 4. Console I/O

## 4.1 Console output：cout

![image-20230608114658193](https://s2.loli.net/2023/06/08/xCWjLp7itqlKyVQ.png)

`cout << expression`的含义是将指定的值或表达式发送到标准输出控制台(`standard output console`)。

## 4.2 Console input：cin（Discouraged！）

这种输入方法是不被鼓励的，cin不够健壮，因为它不能很好的处理用户输入奇怪的值的情况。

> 在课程的视频中，当value与var的type不匹配的时候，变量被赋予了很奇怪的值，并且程序无法继续运行，卡在了一个很奇怪的点。

> 在c++中，如果我们并没有初始化某个已经定义的变量：在main的scope中，该变量是一个随机的垃圾值；在global scope中，该变量会被赋予0。
>
> 本质上是，在C++中，变量被定义后，会被赋予一个随机的内存地址，这里面的值是什么我们不知道，总是是一个垃圾值。

![image-20230609151628615](https://s2.loli.net/2023/06/09/ehb5a6YCufBjAt8.png)

cin是从控制台读入我们输入的值，把它存在给定的变量中。

`>>`与`<<`的含义是，<u>请将数据放入箭头指向的变量中</u>。如cout<<就是把数据放入cout（标准输出控制台）；cin>>age就是把读取到的数据放入变量age中。

此外，我们不能使用`cin >> (int i)`，即把第一行与第三行合并到一起是行不通的，<u>我们不能尝试在一个变量去存储值的同时，去声明这个变量</u>。但是int i = 1例外。（因为本质上它被编译后，顺序是：int i; i = 1;同样是符合规则的）。

> 这与reference variable规则有关。

此外，在C++中，如果我们不做任何输入，即直接点回车，那么程序会被中止，它不会为变量赋任何值，这就是C++的处理方式。

## 4. 3 why is cin bad?

![image-20230609154956592](https://s2.loli.net/2023/06/09/omGDtvQ3rL2VskO.png)

因此，直接使用cin有跟多的弊端，我们不该向上面的例子中那样直接使用cin。建议使用的是Stanford提供给我们的`simpio.h`库。

# 5.cpp声明语法

## 5.1 #include <>/""

![image-20230608123516079](https://s2.loli.net/2023/06/08/dCeVYPmzZ6xnI8F.png)

`#include`代表我们在程序中需要用到的库。`<>`代表导入C++自带的系统库，`""`代表一个在特定项目中的本地库，这个库是我们自定义的，注意不要用错符号。

当我们声明一个项目本地library时，如果是从头开始编写的该项目，那么在`#include`时可能需要用`/`指定该库文件所在的具体位置。如果所在的当前目录中就存在该库，那么就不需要加`/`，即不需要去别的目录路径中查找库文件；若当前目录中不存在该lib库，那么需要指定该库所在的目录的位置（加`/`去指定位置）。

## 5.2 Namespaces and using

![image-20230609122728056](https://s2.loli.net/2023/06/09/KsENT3IeGjOxmbi.png)

通过`namespace`的设定，我们可以让各个文件中<u>相同名字</u>的函数名，变量名，类名各自独立，不会相互影响。在c++中我们常用的`cout，cin`命名都在命名空间`std`中。

我们在程序开头声明`using namespace name`，代表程序想获得在`name`空间中所有内容的命令，代表着我们可以直接访问，而不用第三种方式去引用他。

```c++
#include<iostream>
using namespace std;
// 这里声明了using namespace std
// cout是std命名空间中的内容，因此可以直接使用
// 若不声明using，则需要使用std::cout。
int main() {
    cout << "Hello world" << endl;
    return 0;
}
```

总结：有了`#include<iostream>`，我们可以使用`std::cout`来访问`cout`；但我们若进一步的声明`using namespace std;`，则我们可以直接用`cout`，而不必再打上前缀`std::`，这样可以让我们少打一些字母。

此外，理论上我们可以using多个namespace，但是不同的命名空间中可能会有相同定义的变量，所以遇到这样的变量记得在前面加上其所属的命名空间。

### 5.2.1 自定义namespace

我们在实际操作中，可以在代码中使用我们自己的`namespace`。如下代码段所示：

```c++
#include <iostream>
using namespace std;

// foo命名空间在c++系统库中是不存在的
namespace foo {
    int x;
}
// 下面可以使用foo中的x
int main() {
    cout << foo::x;
    int x; // 额外声明一个x也是不冲突的，因为他们俩是属于不同的命名空间的
}
```

`namespace`就是c++中的一种机制，允许通过某种方式<u>重复使用</u>函数、变量或类的名称，且不会发生命名冲突。一般来看，`#include`操作与`using`操作是同时使用的。

### 5.2.2 namespace的复杂用法

我们可以在多个文件中定义同名的namespace（也可以是自定义的），但是注意当使用自定义的namespace时，不要在不同的文件中的namespace下定义相同的变量，<u>因为本质上这些文件使用的这个同名自定义namespace是属于同一个namespace的，该namespace中的成员是共用的，若在不同文件中、该namespace下定义了同名变量，那么在编译时就会报重定义的错误。</u>

此外，若文件A使用了两个<u>自定义命名空间f1与f2</u>，且两个空间中都有定义变量X，那么当文件A中使用X时，这个X指的是哪个命名空间的X呢？这也是个细节问题。

> 这与namespace的声明顺序、link文件的顺序、符号的顺序等等的细节有关，此外编译器也会尝试着去消除这些歧义，这涉及到很多杂乱的细节。

# 6.Stanford C++ libraries

## 6.1 console.h（更方便的控制台）

`Console.h`：在项目中引用该库，`#include "console.h"`，会额外显示出一个控制台供我们使用，这比Qt Creator自带的库要方便的多。

## 6.2 simpio.h(更安全的cin)

该库中定义的input函数要比std::cin来的更加安全。

![image-20230609155545600](https://s2.loli.net/2023/06/09/yazPVkNcMD72xA3.png)

如上述的表格中的内容，有专门去获取各种类型值的function，参数是提示语。这些函数具有很强的健壮性，它会不断地提示我们进行输入，直到它从输入中得到正确类型的值，然后返回给我们。

> 这些func遇到错误的值会跳过它，让用户不停地输入，直到遇到类型正确的值。

并且他可以很方便的读取一行数据，这是cin所不具备的。

## 6.3 strlib.h(string的成员函数)

Stanford写了一个基于string类的库，这个库加入了额外的方法，可以让我们更方便的使用一些功能。

![image-20230615231907366](C:\Users\Horizontal\AppData\Roaming\Typora\typora-user-images\image-20230615231907366.png)

注意这些都是全局函数，不是string类内部的方法，因此我们不能像string.xxx()方式去使用，而是要把string对象作为参数传入这些函数，如上图中的例子。这些函数的设计是为了复刻Java中具备的一些string类method的特性。

目前的类型转换函数只支持纯数字与纯字符之间的转换，不支持转换完后是字符与数字的结合体的形式。如下面的例子所示：

```c++
// stringToInteger: "1234" -> 1234 
// realToString:    -123.5 -> "-123.5"
```

还有一些别的方法，可以直接阅读库文档去了解更多。

## 6.4 filelib.h(stream相关的函数)

![image-20230621171718453](https://s2.loli.net/2023/06/21/NOGu2UDPov986Bq.png)

里面有一些功能在C++的系统库中有对应的函数，但是这些函数会随着OS的不同，执行效果会有一些差异，而这个filelib.h库旨在为我们排除不同OS带来的不同之处，并提供了很多更强大的功能。

![image-20230807153601233](https://s2.loli.net/2023/08/07/7b1vHYrsqzNah9G.png)

## 6.5 gwindow.h (图形相关的库)

![image-20231021212101578](https://s2.loli.net/2023/10/21/YDKRrENk7CcXLO8.png)
