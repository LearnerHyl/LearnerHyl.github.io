# 1.C++ Strings

## 1.1 [],header file,at

```c++
#include <string>
using namespace std;

int main(){
    string s = "hello";
    //std::string a = "world";
}
```

上表面这段代码说明使用string需要的头文件，以及它是属于std命名空间的。

![image-20230615222155638](https://s2.loli.net/2023/06/15/t7fZJnEGQTVWjai.png)

c++有两种不同类型的strings，string中存储的characters是char类型的，可以用index方式访问，此外这些characters也有自己的ASCII编码，可以转化为int类型打印出来：

```c++
#include<string>
using namespace std;

string s = "Hi cxk";
char c1 = s[3];              //'c'
char c2 = s.at(1);           // 'i'
cout << (int) s[0] << endl;  //72 ASCII->integer
```

在Java中就不能使用`[]`的方式来获取某个位置的字符，只能用某种method来获取。

## 1.2 Operators

下面用一段代码示例说明String类可能涉及到的一些操作符。

```c++
// string的相加可以用 + 或 +=
string s1 = "Mar";
s1 += "ch";                     // March

// string的比较用关系运算符，本质是基于每个character的ASCII码的大小的
// 逐个字符比较，直到有一个（不）符合标准就会停止
string s2 = "Cynthia";          // ==,!=,<,<=,>,>=
if (s1 > s2 && s2 != "Joe") {}  // 这是True

// string是可以被修改的
s1.append(" stepp");            // "March Stepp"
s1.erase(3, 2);                 // "Mar Stepp"
s1[6] = 'o'; 					// "Mar Stopp"
```

c++中的string类可以直接使用操作符来比较字符串内容是否相等，而Java不行，因为C++具有一种Java不具备的语言特性：`operator overloading`（操作符重载）。

```c++
cout << string << int << double;
```

注意，直接使用+操作符的只适用于：`string + string，string+char`。string与别的类型相加是不行的，如对于String+int，我们就必须把int类型转化为string或char才可以继续进行。因此，一旦stringA+stringB中任何一个是C++-String，那么结果就会是C++-string，这就是我们所说的`type enhanced`(类型增强)。

但是如上代码所示，我们在输出内容的时候是可以混杂不同类型的，因为`<<`操作符可以输出不同类型内容。

```c++
// string的相加可以用 + 或 +=
string s1 = "Mar";
s1 += "ch";                     // March

s1.append(" stepp");            // "March Stepp"
```

上面是两种不同的方式得到新字符串的方法，他们在底层细节上存在一些差异：在大多数语言中，对string进行更改的操作往往会返回一个新的string；对于直接用+=的方式而言，他会创建一个新的string对象（使用的是另一块新的内存空间），可是原来的s1就会被遗弃(如果我们不继续使用它)，他会无用的占用内存空间。

<u>因此，上面两种方法的差异在于，第一种+=方式会新申请一块内存创建一个新的string对象，即你现在看到的S1与之前的S1用的不是一块内存空间。但是第二种方式则可以直接修改原来的那个string对象，只是对它进行了扩容操作，而不用整个的再重新申请一次。</u>

> 这个细节注意把握一下，第二种的方法更利于管理内存。但是一般而言，如果涉及到的字符串不是特别大，两种方法效率上是没什么差距的，第一种写起来更方便一些。

## 1.3 Member Functions

![image-20230615225700769](https://s2.loli.net/2023/06/15/kDIUG9Mm6CSyXPE.png)

上图是string中的一些成员函数，唯一要注意的是，当调用`s.find(str)`时，如果没能找到目标str，那么就会返回一个常量：`string::npos`。即使我们已经`#include<string>`，但是还是要用`string::npos`的这种完整的方式访问该静态成员变量。

> `::`操作符有区别。在namespace中，如`std::xxx`，意味着我们使用了命名空间中std中的xxx成员<u>。但是上面的`string::npos`则是访问string类中的静态成员npos。</u>Java中访问静态成员的方法是：`class.member`,而在c++中访问静态成员的方法是：`class::member`。

此外，Stanford自己也定义了一个名为strlib.h的string成员函数库，里面加入了很多额外的功能，具体介绍请见第一堂课中的Stanford Libraries内容介绍。

## 1.4 string user input

![image-20230616145150711](https://s2.loli.net/2023/06/16/CMe9Q7GVFpquPtd.png)

上面是如何进行string读取的方法，可以使用`cin`来进行读取，但是每次只能读取一个单词，遇到空格就会终止。或者在Stanford库中的`getLine()`函数，可以一次性读取一整行数据，不会发生终止。此外，在c++的标准库中也有一个`getline()`函数，但是这个函数需要输入两个参数：第一个是你想从哪个对象中读入参数值（cin），再将该参数值存储至哪个string对象中（name）。

所以我们一般更倾向于使用Standford库中提供的getLine()函数，更加的方便快捷。

## 1.5 C vs. C++strings(两种类型的string)

![image-20230616150827827](https://s2.loli.net/2023/06/16/iDud2bXUZlOKnm9.png)

c++的设计者希望c++可以与c语言兼容，因此c++就有两种类型的strings：一种是c语言中的strings（`char arrays`）、另外一种是c++中的strings（`string objects`）。<u>简单一点来说，当代码中出现单独的"XXXX"这种内容且没有直接赋给一个c++ string类型时，那么他就是一个c-string，如下面的代码段：</u>

```c++
auto s = "hi there" + "cxk"; // 这种写法是错误的，编译能通过，但是运行会报错，具体分析见第2节
string s1 = "hi there"; // 这样就会被转化为c++ string
```

上面学习的string就是c++的string objects，形如我们写的像"hi there"这样的就是一个C string，C++ string中的任何成员方法都不能被用于C string中。因为C语言是一种非常轻量级的语言，它无法支持太多的成员函数，但是c++可以听更多的函数与特性。

我们应该尽量避免C string类型的出现，因为c++ string足够应付我们遇到的大多数场景。两种string的转换实现：

```c++
string s1 = "cxk"; // c++ string
string s2 = string("cxk"); //将C string转化为c++ string
s1.c_str(); // 将c++ string转化为c string
```

# 2. C string bugs(哪些场景可能会碰到)

本质上是因为c++中pointer特性的问题：本质上下面的问题语句执行的都是内存地址值的相加，因此可能会导致string拿到的是一个垃圾数据，或者发生内存越界。

注意我们说明的c/c++ string的内存地址值的是string的第一个字符的地址。

这些bug最恐怖的地方在于，他们大部分是能够通过编译的，但是在你运行的时候会莫名其妙的报错，然后你还不知道具体错在什么地方。

## 2.1 bug1: c++ string = c-string + c-string

尽管大多数场景下我们都不大可能遇到c string的使用，但是总有遇到的时候，我们有必要理解c string与c++ string一起使用时会出现什么bugs。

```c++
string s = "hi" + "there"; // C-string + C-string
```

这句话是不对的，因为C-string不能直接使用c++ string的成员函数与运算符，但是这样的写法在c++中能编译通过，但是运行时程序会出现中断或错误。本质上，这个问题的出现是因为c++中的一个特性：pointer（指针）。

<u>c++在运行这句代码时所经历的过程是这样的：程序会去取得"there"的内存地址，然后将该内存地址与"hi"所在的内存地址相加，之后将这个相加后的地址的和的值返回，无论相加后的内存地址是什么，都会在相加完成后被返回。所以这句代码其实并没有执行字符串相加，它本质上是做了地址的相加。</u>

因此这可能会导致s拿到的是一个垃圾数据或发生内存越界，因此程序会崩溃或打印垃圾内容，这很蠢，但是没办法这就是c++语言特性的一部分。

## 2.2 bug2: c++ string = c-string + char/int

```c++
string s = "hi" + '?'; // C-string + char
string s = "hi" + 41;  // C-string + int
```

这两种写法同样是不对的，错误类似于2.1中：

`c-string+char`同样会直接将char的内存地址加到cstring上，并把相加后的内存地址返回给s；

`c-string+int`有些许不同，如第二个，这里会直接把int个bytes偏移加到第一个c-string的内存地址上，即在"hi"的首内存地址值上+41，将相加后的内存地址返回给S。

<u>为了进一步理解上面的内存地址值相加的原理，这里以一个例子来详细说明：</u>

```c++
string s = "abcdefghijklmnopqrst" + 10; // C-string + int
```

程序返回给S的字符串是从前面的c-string的第10个字符开始的。即s的地址实际上是前面那个c-string的第10个byte的位置，因为每个char是1个byte，因此s这个字符串的首地址，就是C-string的第10个字符的地址。

所以，不要对C-string类型进行+操作，这会导致程序垃圾的出现。

如果我们对c++ string类型直接进行加法，会有不同的结果出现：

```c++
string s = "hi";
s += 41;      // "hi)"
```

如果一个c++string直接+一个int类型，它确实会加上一个字符（该字符的ASCII码是该int值），而不会产生"hi41"。

## 2.3 bug3: C-string to int

```c++
int n = (int) "42";  // n = 0x7ffdcb08
```

如果我们尝试对一个C-string进行强制类型转换，如上面的语句，这一般是不能通过编译的，<u>但是一旦编译通过了，那么程序就会获取"42"的内存地址，然后将该内存地址值转化为int值，并存储在n中。</u>

## 2.4 C-string bugs fixed

针对上面提出的3个bug，我们有方法可以解决这些问题，如下代码段：

```c++
// bug1: c++ string = c-string + c-string
string s = string("hi") + "there";  // 提前将一个转化为c++-string，就会发生type-enhanced

// bug2: c++ string = c-string + char
string s = "hi";
s += '?'; // "hi?"  这里发生了自动类型转换，同样是type-enhanced

// bug3: c++ string = c-string + int
s += integerToString(41);      // "hi?41"
int n = stringToInteger("42");  // n = 42   必须用Stanford的strlib库调用这些转换函数才可以
```

我们可以用不同的方法解决这些bug，但是最好的方式是尽量避免使用c-string，因为c++-string已经足够满足我们需要的大部分场景。
