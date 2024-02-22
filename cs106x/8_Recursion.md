# 1. Part 1-Recursion(递归)

## 1.1 Recursion（递归） And Loop（循环）

递归（Recursion）和循环（Loop）都是计算机编程中常见的控制结构，用于重复执行一系列操作。它们之间的定义和区别如下：

递归（`Recursion`）：
- 定义：递归是一种通过在函数内部调用自身来解决问题的编程技术。换句话说，函数在执行过程中会重复调用自身，直到满足某个终止条件才会停止递归。
- 特点：**递归是通过将一个大问题划分为一个或多个更小的相同子问题来解决的**。在递归过程中，每次函数调用都会将问题的规模缩小，直到达到基本情况，然后开始回溯并将结果合并，最终解决整个问题。所以注意递归是先不断地调用，之后会从栈顶开始逐步的返回结果并合并。
- 递归更适合于解决那些具有`Self-Similar`性质的问题，如我们在算法导论中见过的一些问题的解决，在n为一般值的情况下，往往可以通过对之前的结果进行某种计算而得到新结果。
- 这就是一个典型的`Recursion`的例子：`result(n) = result(n-1) + result(n-2)`，很明显，这适合于一些较为复杂的的大型问题；
- <u>重复无脑的执行某些代码，并且在递归过程中也不涉及到解决这个问题的子问题，用递归就会显得很愚蠢（比如你在写B+Tree的时候，FindLeafPage，LeafSplit，LeafMerge，本来用Loop就够了，这并不涉及什么子问题的合并，而你非要用递归来写，显然你太蠢了）。</u>

循环（`Loop`）：
- 定义：**循环是一种重复执行一组语句或代码块的控制结构，直到满足特定的终止条件为止。**
- 特点：循环通过在循环体内部执行一系列语句，并在每次迭代后检查终止条件是否满足来实现重复执行。如果终止条件为真（满足），循环会停止执行，否则循环将继续进行。

二者在某些情况下可以相互替代使用：即将程序中的递归操作替换成循环操作。<u>此外，一般情况下，递归解法的运行速度一般会比loop解法的慢。</u>因为C++编译器非常擅长对Loop行为进行优化，但是recursion操作则很难优化。

若使用Recursion意味着我们要进行多次函数调用，那么优先使用Loop。

## 1.2 Recursion and cases

每个递归算法都涉及至少两种情况：

- 基本情况：一个简单的情况，可以直接回答，即一次调用就可以返回结果，不需要继续递归调用。
- 递归情况：问题的一个更复杂的情况，无法直接回答，但可以用相同问题的较小情况来描述，即核心思想依然是将递归情况拆分为多个可以单独解决的基本情况，之后回过头来进行基本情况的合并，就可以最终求解出这个复杂的递归情况的答案。
- 关键思想：在递归代码中，你自己处理任务的一个小部分，然后进行递归调用以处理剩余的部分。<u>即每次递归调用都要可以解决当前问题的一部分子问题。</u>
- 向自己问问题："这个任务如何是自相似的？"
  - "我如何用一个更小或更简单的版本来描述这个算法？"

递归优化一般是在编译器的内部进行的，故在谈论递归时通常不会过多的谈论优化的话题，后面会简要的提一下。

下面以power（a,b）算法为例，说明基本情况（`base case`）与递归情况（`recursive case`）的具体定义：

```c++
int power( int base， int exp) {
    if (exp == o) {
        // base case; base^0 = 1
        return 1;
    }else {
    	// recursive case: x^y = x* x^(y-1)
        return base * power(base, exp - 1);
    }
}
```

## 1.3 Preconditions(先决条件)

前置条件（`precondition`)：指的是在调用代码时，你的代码假定为真的条件，即默认调用者可以满足这个先决条件。通常在函数的头部以注释的形式进行文档化：

```c++
// Returns base^exp.
// Precondition: exp >= 0
int power(int base,int exp) {
```

陈述一个前提条件并不能真正地“解决”问题，但至少可以记录我们的决定并警告客户不要这样做。

如果调用者不听并仍然传递了一个负数幂，怎么办？如果我们想要实际执行前提条件，怎么办？

### 1.3.1 Throwing exceptions

`throw expression`：

- 会生成一个异常，除非有代码来处理（捕获）这个异常，否则程序会崩溃。
- 在Java中，只能抛出继承自Exception的对象；而在C++中，可以抛出任何类型的值（int、string等）。
- 有一个名为std::exception的类可供使用。
  - Stanford C++库的"error.h"也有一个error(string)函数。


c++中我们可以通过`try-catch`搭配来解决函数调用时发生的异常，并进行相应处理。

```c++
// Returns base ^ exp.
// Precondition: exp >= 0
int power(int base，int exp) {
    if (exp < 0) {
    	throw "illegal negative exponent"; 
    }else ...
}
```

### 1.3.2 shorten call stack

通过对power代码中的`recursive case`进行进一步的细化，可以缩短调用栈的深度：

```c++
//Returns base ^ exp.

//Precondition: exp >= 0
int power(int base, int exp) {
    if (exp < 0) {
        throw "illegal negative exponent" ;
    }else if (exp == 0) {
        // base case; any number to 0th power is 1
        return 1;
    }else if (exp % 2 == 0) { // 区分exp奇偶的情况，可以缩短递归次数
    	// recursive case 1:x^y = (x^2)^(y /2)
        return power(base * base, exp / 2);
    }else {
    	// recursive case 2: x^y = x * x^(y-1)
        return base *power(base, exp - 1);
    }
}
```

# 2. Part2-Recursive Data and Optimizations

## 2.1 some tips

进行大量的练习可以对递归的概念理解的更加深刻，本节课主要侧重于讲解在实现Recursive的过程中可以进行的一些优化操作。

注意在Recursive函数中的base case可以是隐式的，即不必非得把它写出来。一般来说当我们去依次读取文件中的每个行的时候，我们更习惯用`getline(input, line)`的方式，而不是用`input.eof()`，因为使用后者会经常出现一些难以理解的bug。

> 对于input.eof()而言，当我们读取了文件中的最后一行数据后，eof()并不会变为true，只有当我们读完最后一行后再次读取时，input.eof()才会变为True；上述"读完最后一行后再次读取"的操作可能会引发bug出现。

对于一些与文件路径、文件扩展名等等相关的题目，在SPL中提供了`filelib.h`头文件，里面有和文件名称、路径、扩展名等等一系列相关的函数。相关变量可以在lecture1中的SPL库集锦处查看。

## 2.2 The optimization for fib tree-Memoization

记忆化（Memoization）：为了提高速度，缓存先前昂贵的函数调用的结果，以避免需要重新计算。

- 通常通过将调用结果存储在集合中来实现。

伪代码模板:

```pseudocode
cache = {}. //empty
function f(args) :
    if I have computed f(args) before:
    	Look up f(args) result in cache.
    else:
    	Actually compute f(args) result.
    	store result in cache.
    Return result.
```

在斐波那契数列计算的递归方法中采用这种思想可以极大地减少函数做不必要运算的次数，缩短运行时间。

1. 视觉上来看它不再是一个二叉调用树的结构，不用2^n次方调用次数。
2. 只要计算过一次fib(n)，后续再次计算fib(n)只需要从cache中取即可，不用再次计算。

缓存的思想还可以用到日常的编程中，想想曾经学过的spark。

## 2.3 static 关键字的作用

在C++中，`static`关键字有以下几种用法：

类中的静态成员变量：静态成员变量在类的所有实例之间共享。它们不与特定对象绑定，内存中只存在一个变量的副本。

类中的静态成员函数：静态成员函数可以在不创建类对象的情况下调用。它们只能访问静态成员变量和其他静态成员函数。

函数中的静态局部变量：带有`static`关键字的局部变量在函数调用之间保持其值。这些变量仅在第一次调用函数时初始化一次，一般在函数开始的部分就声明。

静态全局变量：带有`static`关键字的全局变量具有内部链接，这意味着它们仅在同一个翻译单元（源文件）内可访问。其他翻译单元无法访问这些变量。

# 3. Part3-Fractals

分形（`fractal`）：一个自相似（`self-similar`）的数学集合，通常可以绘制为重复出现的图形模式。

- 相同形状或模式的较小实例发生在模式本身中。
- 当显示在计算机屏幕上时，可以无限放大/缩小分形。

就是用SPL中的图形库进行无限套娃类型的图像绘制。还有用self-similar的思想去画一些特殊的图形。
