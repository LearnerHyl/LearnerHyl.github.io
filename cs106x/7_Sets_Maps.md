# Chapter5：Set，Lexicon，Map

# 1. Sets

`set`：一组**唯一值**的集合（不允许重复），可以高效地执行以下操作：

- 添加-`add`、删除-`remove`、搜索-`search`（包含-`contains`）
  - 当准备添加一个重复值时，Set会直接选择不操作以略过这个值。
- 我们不认为Set具有任何索引，Set不支持索引；我们只是一般地将事物添加到集合中，不必担心顺序。

![image-20230719145524553](https://s2.loli.net/2023/07/19/mY6Bw2FNCIZcJ4D.png)

上图是对set的概念的描述，主要在于凸显set本身内部的元素是无序的，不存在index与序号的概念。

## 1.1 SPL-C++ Sets

SPL有两种Set实现，他们俩的方法接口完全相同，只是内部实现不同。这两个collection是作为同一个ADT（抽象数据类型）的两个collection，即代表了不同的内部实现。

这类似于Java中的`TreeSet`与`HashSet`，功能与性质上是差不多。

`Set`：使用称为二叉树的链接结构实现。

- 相当快，元素按排序顺序存储
- 值必须具有 `<` 操作符

`HashSet`：使用称为`hash table`的特殊数组实现。

- 最快（比Set要更快），元素按不可预测的顺序存储
- 值必须具有 `hashCode` 函数（对大多数标准类型提供）
  - 变体：`LinkedHashSet`（略慢，但可以记住插入顺序）


如何选择：您需要元素按排序顺序排列吗？

- 如果需要：使用 `Set`。

- 如果不需要：为了性能提升，请使用 `HashSet`。

## 1.2 Set members-SPL

![image-20230719154631841](https://s2.loli.net/2023/07/19/eSzfpDHYA3tsMnh.png)

## 1.3 Set operators-SPL

Set支持operator重载，下图中说明了各个符号具体代表了什么操作类型：

![image-20230719155509376](https://s2.loli.net/2023/07/19/YHOgEkz325QXwud.png)

如果想遍历一个set，只能使用for-each的方式去遍历：

- 集合没有索引；不能使用普通的带索引的 for 循环迭代Set，即不能用Set[i]的方式访问Set。
- Set 按排序顺序迭代；Hashset 按不可预测的顺序迭代。

```c++
// for-each loop
for (type name : collection) {
    statements;
}
```

# 2. Set/HashSet of Structs

## 2.1 Structs介绍

C++中有一个叫做结构体（struct）的实体。

- 类似于轻量级的类；一组数据和行为。
- 默认情况下，成员变量和方法是公共的（`Public`）。

```c++
struct Date {  // declaring a struct type
	int month;
	int day;  // members of each Date structure
};
Date today;   // construct structure instances
today.month = 7;
today.day = 13;

Date xmas {12，25}; // initialize on one line
```

Class是相较于Class而言更为简单地实体（entity），我们一般只用struct来存储几个变量，我们可以将其看作一个只存储数据变量的class，虽然它本身也可以放成员函数。

注意`DataType  a = {1,2,3..}`，这种通用的初始化方法可以用于C++中各种类型变量，这是一种革新。

## 2.2 Set/HashSet of Structs

默认情况下，我们的库无法将您的结构体存储在`set`或`Hashset`中。

- 要放入`Set`中，您的结构体需要使用`< operator`进行排序。即我们需要为Struct单独重写一个`< operator`。(`operator overloading`)
- 要放入`HashSet`中，您需要实现`==`，`!=`，以及一个`hashCode()`函数。
  - 我们稍后会更详细地讨论这些概念。

```c++
// operator overloading; define < operation on dates
// (required in order to put Date in a set)
bool operator <( const Date& d1，const Date& d2) {
	return d1.month < d2.month || (d1.month == d2.month && d1.day < d2.day) ;
}

// converts a Date into an integer key
// (required in order to put Date in a HashSet)
int hashCode( const Date& date) {
	return date.month * 100 + date.day;
}
```

在SPL中，如果我们想`cout << Set<Structs> xms << endl;`同理我们仍然需要重载操作符`<<`。即如果我们想让某个特殊的数据类型可以被打印，则我们必须重载关于该数据类型的 `operator << `的方法。同理也适用于别的操作符，对于特殊/自定义数据类型，我们需要重载目标运算符。

```c++
// operator overloading example
ostream& operator <<(ostream& out, const Date& d) {
	out << d.month << "/" << d.day;
    return out;
}
```

上面是一个重载运算符`<<`的例子，我们设置`ostream &`为返回类型的原因是，这样我们可以连续使用<<运算符，即类似于：`cout << date1 << date2 << date3 << ...... << endl;`的方式。

# 3. lexicon-SPL（后续更详细讲解）

lexicon是SPL库中的一个文件，定义上它是：一组优化了字典和前缀查找的单词的Set。本质上`Lexicon`与`Set<string>`是一样的，但是有一点不同的是，`Lexicon`中的string元素是按照字典顺序排序的。

![image-20230721151922797](https://s2.loli.net/2023/07/21/PF2TDbuyBIijcLN.png)

为什么`Lexicon`可以有这种方法，这是因为Lexicon在内部有一种与Set不同的实现方式,即`prefix tree`(前缀树),我们可以因此很方便的进行string元素的访问。

在作业2的`Boggle`中，我们会用到`Lexicon`这个结构体，后续我们还会进行详细的讲解。

# 4. Maps-SPL

即HashMap，在某些语言中被称为Map，在Python中被称为`Dictionary`。同样Map对象中的key是唯一的。

## 4.1 Concepts

`map`：一种存储键值对的集合，其中每个键值对包含一个称为键（`key`）的第一部分和一个称为值（`value`）的第二部分。

- 有时被称为`Dictionary`、`associative array`或`hash`。
- 用途：添加`(key，value)`对；通过提供`Key`来查找相应的`Value`。注意只能通过`key`来查找`value`，反过来是不成立的。

## 4.2 Map operations -SPL

下面的m默认是一个Map对象，介绍Map对象所具有的一些操作。`#include"map.h"`

`m.put(key, value);` 往Map中添加一个键值对。.

```c++
// 向map中插入pair的方法，两种方法是等价的
m.put("Eric", "650-123-4567"); //  or,
m["Eric"] = "650-123-4567";
// Replaces any previous value for that key.
```

此外，如果在插入前map中已经存在了一个相同的key，那么新的value就会替代掉原来的value。

`m.get( key);`返回与给定键（key）配对的值（value）。

```c++
string phoneNum = m.get( "Yana" ); // "685-2181", or,
string phoneNum = m[ "Yana"];
// Returns a default value (0,0.0,"" , etc.) if the key is not found.
```

当然，如果传入了一个不存在的Key作为参数，那么会返回默认值作为结果。

`m.remove(key );`移除给定的键（key）及其配对的值（value）。移除指定的键值对。

```c++
m.remove( "Marty" );
// Has no effect if the key is not in the map.
```

## 4.3 Map implementations-SPL

在斯坦福C++库中，有两种（Map）类，学习的时候可以类比一下`Set`与`HashSet`：

1. `Map`：使用一种称为二叉搜索树的链式结构来实现。
   - 对于所有操作来说都相当快速；键（keys）按照排序顺序存储。这意味着通过for-each方法访问得到的pairs结果会按照他们的key进行排列。
   - 两种类型的Map实现了完全相同的操作。
   - 键的类型必须是一个可比较类型，支持小于（<）操作符，如果有必要，我们必须重写<操作符。
2. `HashMap`：使用一种特殊的数组结构称为哈希表来实现。
   - 非常快速，但是键的存储顺序是不可预测的。（因为不必实现排序），这意味着通过for-each方法访问会得到一个不可预测的顺序。
   - 键的类型必须有一个哈希码（hashCode）函数（但大多数类型都有这样的函数）。

需要两个类型参数：一个用于键（keys），一个用于值（values）。

```c++
// maps from string keys to integer values
Map<string, int> votes;
```

相关成员函数方法：

![image-20230723210419532](https://s2.loli.net/2023/07/23/u79qgzLPEHa5esF.png)

## 4.4 Looping over a map

在（Map）上，使用 for-each 循环可以处理键（keys）。

- 在 Map 中，键以排序的顺序进行处理；而在 HashMap 中，键的顺序是不可预测的。
- 如果你想要处理值（values），只需通过键（key） k 查找 map[k] 即可。

```c++
for (string name : gpa) {
	cout <<name << "'s GPA is " <<gpa[ name] <<endl;
}
```

Map相关的练习可以看练习题部分，Counting exercise。
