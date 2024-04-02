# preview

c++中的stream是为了读取与写入文件而设计的，而grid则是一种二维数组结构。

# 1. Stream

## 1.1 Reading/write files

![image-20230617122233566](https://s2.loli.net/2023/06/17/72menokMROctJSb.png)

当我们向读取文件中的内容时，需要#include<fstream>，(即filestream)，fstream是c++的一个系统库。如上图所示，`ifstream`用来处理由文件输入的内容（即读取目标文件内容，如java中的`inputFileStream`），`ofstream`用来处理输出至文件的内容（即将内容写出到文件，如Java中的`outputFilestream`）。

从上图中我们不难得知，c++的stream类是呈一个层次型的继承结构。ifstream继承了istream类，ofstream继承了ostream类，而istream类与ostream类都继承了ios类。此外，`ifstream`与`ofstream`在行为上分别与cin和cout几乎完全一致，所以上面说cin是一个ifstream，cout是一个ofstream。因为ifstream与ofstream的根父类是同一个，所以他们俩有很多member fucntions是相同的。

因此这意味着，一旦我们学会了如何使用cout与cin，我们就可以使用十分相似的语法去执行文件的写入与读取。

## 1.2 reading files-`ifstream`

`ifstream`用来处理由文件输入的内容（即读取文件的内容）。相关成员函数如下：

![image-20230617165703357](https://s2.loli.net/2023/06/17/MKg3ydG5CJLRWXf.png)

```c++
#include<fstream>

string s;
ifstream f;   // ifstream是从文件中读取内容的流对象类型
f.open("course.txt");  // 打开文件course.txt, 现在f代表的就是文件course.txt
f >> s; // 类似于cin，不过这里我们是从文件流f中读取内容到s中，遇到空格就中止
// cin >> s 是从标准控制台输入中读取一个内容并放到s中，遇到空格中止

getline(f, s); // 从文件f中读取一整行数据到对象s中，
f.close(); //读取完毕，可以关闭该ifstream对象
```

此外，如果我们打开了一个并不存在的文件，那么`ifstream`的`failstate`状态为就会被置为true，我们可以在打开后检查这个位的值来确定是否打开成功。`ifstream.is_open()`

下面举出几个常见的使用`ifstream`对象进行文件读取的例子：

### 1.2.1 Line-based I/O example

```c++
// read and print every line of a file
#include <fstream>

ifstream input;
input.open("poem.txt");
string line;
while (getline(input, line)) { // getline()读到内容会return true，读不到就返回false
    cout << line << endl;
}
input.close();
// Java中还需要new一个来初始化该对象,像下面这样
inputFileStream input = new inputFileStream;
```

上面这段代码做了这些事情：打开文件，按行读取文件中的内容并输出，读取完后关闭该文件。`getline()`函数的input与line都是引用语义，该函数会读取input中的内容并将结果存入line。<u>此外，在c++中有一些别的类型与Boolean类型之间相互转换的规则，后续课程中教授会专门讲这块内容。</u>实际上，如果这里把ifstream的引用语义改为值语义，那么就会导致编译错误，因为ifstream对象是无法被复制拷贝的，程序不知道该如何拷贝ifstream对象，因此会导致编译error。

此外，c++与java在声明ifstream上有区别：c++可以直接`ifstream input;`，<u>这就已经意味着input对象被声明+初始化了，它并不是null值，他有自己的内存空间</u>。

相当于Java中的`inputFileStream input = new inputFileStream;`，即在java中还需要额外new一次来为这个对象分配内存空间（即初始化）。

由于fstream对象是无法被拷贝的，所以getline()中用的是引用语义。这也适用于别的场景下，当我们试图用一些复杂的class作为参数时，我们没有写专门的deep copy方法，那么此时使用引用语义就是最好的选择。

下面看一种常见的错误写法：

```c++
// incorrect (why?)
// 教授说这段代码会把最后一行打印两次
// 本质问题在于多了一次无效读取
ifstream input;
input.open("cxk.txt");
// ifstream.fail()是一个成员函数，用于检查ifstream文件输入流对象上一次读取文件的操作是否成功
while (!input.fail()) {
    string line;
    getline(input, line);
    cout << line << endl; 
}
```

 `ifstream.fail()`的作用是检查上一次读取操作是否成功，而在这里，当`getline()`读取完最后一行后，input的内容指针已经到达EOF，由于`input.fail()`的结果还是true，<u>所以会再执行一次getline()，而这次input不会返回任何内容，因为上次读完已经到EOF了，所以line没有接收到任何内容，还是上一次的垃圾内容，故这里的垃圾内容会被输出，</u>一般是上一次读取到的内容（虽然line是临时变量，但是我们没有对其进行显式垃圾回收，故当我们再次声明时，他可能用到的还是上一次的内存地址）。

### 1.2.2 Operator >>

![image-20230618155100030](https://s2.loli.net/2023/06/18/Mhr6tugiHGjDE5p.png)

下面的代码可以将文件中一系列的由空格（换行符）分隔的文本通过ifstream对象input输入到一个string对象或int对象中（类型对应上即可），即可以将文件中的空格（换行符）忽略，只读文本内容：

```c++
#include <ifstream>

ifstream input;
input.open("data.txt"); // 上面PPT中有data.txt文件的内容排版
string word;
input >> word;   // "Marty"
input >> word;   // "is"
int age;
input >> age;    // 12
input >> word;   // "'years'"
input >> word;   // "old!"

// 当文件中已经没有文本可读取时，input >> word将不会读取任何内容，且会返回一个false值
if (input >> word) { // false
    cout << "successful!" << endl;
}
```

## 1.3 reading files-`istringstream`

读取文件时，除了上面提到的`ifstream`，我们在上面的大图中也注意到了还有一个`istringstream`类，<u>该类可以对字符串进行分词（指定分词符号或默认）</u>：如果我们想把文件中的一整行直接读入，然后再将这行中的文本分割为一个个的字母或者wrod，那么`istringstream`显然更合适一些：

```c++
#include <sstream>  // 注意头文件是sstream
// An istringstream lets you tokenize a string

// read specific word tokens from a string
istringstream input("Jenny Smith 8675309");
string first, last;
int phone;
input >> first >> last; // first="Jenny", last="Smith"
input >> phone;    // 8675309

// read all tokens from a string
istringstream input2("To be or not to be");
string word;
while (input2 >> word) {
    cout << word << endl; // To \n be \n or \n not \n .....
}
```

`istringstream`对象也有`ifstream`对象有的成员函数，但是`istringstream`对象会从string对象中读取数据，而非从文件中读取数据。<u>所以我们可以把`ifstream`与`istringstream`结合起来用：使用`ifstream`读取文件中的每一行，之后将行传给`istringstream`对象，使用其对该行中的word进行单独提取。</u>但是注意我们不能直接用`istringstream`去换掉`getline()`中的`ifstream`对象。

学完这节课，我们就知道该如何对大多数类型的数据与文本进行切片或切块了。

## 1.4 `ostringstream`

一个`ostringstream`对象允许你将输出写入到一个`std::string`中，它的工作方式与`cout`很像。当我们向该类型的stream中写入完成后，调用`ostringstream.str()`函数可以将其存储的内容导出并转化为`std::string`对象，如下面的例子所示：

```c++
#include <sstream>

// 我们可以用这种方法构建一个很长的string对象，相比于直接用string增量，这样更有效率
// produce a formatted string of output
int age = 42, iq = 95;
ostringstream output;
output << "cxk is " << age << endl;
output << "and his IQ is" << iq << "!" << endl;
string result = output.str();
```

当你想构建一个string对象，但不想使这个string一边被添加内容一边被更新（即string需要记录中间状态）时，就可以采取这种方式：只进行一次转换，即最后得到的结果的转换。

Java中的`StringBuilder`也可以实现类似于`ostringstream`的功能。

此外，在standford的项目库`"filelib.h"`中，有一些关于stream的自定义方法，详见第一篇笔记。

<u>**注意，在作业中是不允许使用这些项目库的。**</u>

# 2. Collections（容器）：Grid

## 2.1 collections基本定义

`collection`:collection是被用作存储数据的数据结构，在该数据结构中的每一个`object`被称为`elements`。c++有一系列的`standard collection`，它们的集合被称为`standard template Library(STL)。`

Stanford自己也写了一个`standard collection`，我们称之为`SPL`，SPL与C++内置的STL非常像，<u>但是SPL中的collection拥有越界检测功能与内存错误检测功能，还拥有更简洁的接口与错误信息。</u>

课程计划是先教我们SPL，之后再逐渐的过渡到STL，学会SPL后很容易就能上手STL。在家庭作业中建议使用SPL，因为SPL中的collection更容易debug。

## 2.2 Grid(SPL)

它的存储结像一个二维数组，<u>但是有更强大的功能</u>；此外在使用的时候必须在`<>`中指定存储的元素的类型（`<>`是一个template或一个类型参数）。对于读取越界、内存错误问题，Grid可以及时报错，而c++中的普通二维数组则不具备这种功能。

`<>`在Java中被称为`generic`(泛型)，而在c++中我们称之为类型参数或template。

```c++
#include "grid.h" // SPL的库

// constructing a Grid
Grid<int> matrix(3, 4); // 3行四列的二维数组
matrix[0][0] = 75;

// or specify elems in {}
// 当我们提前知道所有元素的时候，可以用这种方式赋值
Grid<int> matrix = {
    {1,2,3,4},
    {5,6,7,8},
    {9,10,11,12}
};
```

此外，当我们声明了Grid对象的类型与维度空间后，他们的默认值是0或与0类似的值。若是double类型，则是0.0；若是bool类型，则是false。

只有在SPL中才会有这么贴心的设计。<u>因为在C++中的大部分语言库都不会将collection中的未赋值元素设置为默认值，一般会是一个垃圾值。</u>

## 2.3 Grid的成员函数

![image-20230621195729240](https://s2.loli.net/2023/06/21/SNAFPaUwthQ237f.png)

Grid未实现自动扩容机制，我们声明它是多大它就是多大，除非通过`resize()`函数来显式的改变其大小，当然可以选择保留原来的内容或舍弃原来的内容，更多的请见官方文档。

此外，最后一个`ostr << g`，这里的ostr值的是负责输出的stream，如`cout,ostream,ostringstream`这些，在C++的STL中是没有这样的直接输出功能的。

## 2.4 Grid as parameter

当Grid类型对象在函数中的参数是值语义时，C++会复制它全部的内容，而Grid对象往往是一块很大的数据内容，需要占用大量内存空间，因此copying非常低效，所以当Grid对象作为参数时，我们总是使用`引用语义`。

此外，当函数不需要修改该Grid对象时，一定要加上`const`的限制，因为若把一个变量以引用语义作为参数传给某函数的话，那么函数就可以实时的修改这个变量的值，而加上`const`则让该变量不可以被修改。

即`const reference`修饰可以在避免低效复制Grid对象的前提下，防止该对象（共享对象）被函数修改。

一般而言，对于Grid对象我们都会使用`reference`参数，若真的需要复制一份副本，我们可以在函数内部进行复制而不是在参数一栏，如：`Grid tmp = param`。在C++中。tmp会直接复制param这个Grid对象的所有内容，而在Java中，tmp与param指向同一个内存地址，这是二者的不同之处。

当我们使用`collections`、`file stream`对象作为函数参数时，我们一般都会使用引用传递。

注意函数声明与函数实现部分的函数头的参数格式要严格的保持一致（如：&,const等等一系列的参数修饰词一定要严格一致，要么都有，要么都没有）。
