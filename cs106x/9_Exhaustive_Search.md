# Exhaustive Search（穷举搜索）

# 1. Basic concepts

穷举搜索的搜索空间通常由许多个决策(`decisions`)组成，而每个决策又有好几个选择(`choices)`。

- 比如，我们列举一个五个字符的单词，这五个字符中，每个字符的选择都是一个决策(`decision`)，而每个字符有26个可能的选择(`choice)`。

通常用Recursion的方法去实现穷举搜索。这里的base case与之前Recursion中的定义是有所不同的。

![image-20240110104557331](https://s2.loli.net/2024/02/20/XLphgRB6K7b4M5e.png)

`base case`：在算法已经完成了所有的决策，没有决策可以做的时候，这时就是`base-case`。

## 做题练习

做了`PrintBinary`、`printdecimal`这两个题只需要用简单地recursion做即可。

`Permutations`排序：给出一个单词，需要打印出该单词所有可能的变序组合。
