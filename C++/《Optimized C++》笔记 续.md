

[TOC]
# Chapter 7 Optimize Hot Statements

在语句层面的优化可以认为是一个把指令从执行流中移除的过程。这里的问题在于，除了函数调用之外，C++的语句单独都不会占用太多的机器指令。不过有一些因素会放大这些语句的耗费：

- 循环。profiler可以指出一个函数里边有热循环，但并不会说是哪一个循环。也可以指出一个在循环内调用的函数是热点，但并不会说是在什么地方被调用的。也就是说，profiler并不能直接指向到循环，必须有程序员自己去查找。
- 频繁调用的函数。通常profiler可以直接指向热点函数。
- 程序中的一些习惯用法。可以考虑代价更低的惯例。

对于桌面和集群级的处理器而言，由于指令级的并发和缓存机制，对于语句级的优化效果并没有对内存分配和复制的优化效果显著。所以对于桌面级应用，语句优化也只适用于频繁调用的库函数以及程序中最内层的循环，比方说游戏图形引擎和整天都在运行的可编程语言翻译器。

语句优化的另外一个问题是优化的效果依赖于编译器实现。这也是语句优化效果相比较而言不是那么显著的另一个原因。

## Remove Code from Loops

一个循环有两个部分：一个是循环体，一个是循环控制。通常都是从循环体中移除语句，不过对于循环控制也是有优化机会的，比方说

```c++
char s[]="This string has many space (0x20) chars. ";
for(size_t i=0;i<strlen(s);++i)
{
    if(s[i]==' ')
        s[i]='*';
}
```

显然每次迭代对`i<strlen(s)`的检测会使得这个算法从$O(n)$变成$O(n^2)$。strlen有一个隐藏在库函数内部的循环。

### Cache the Loop End Value

所以这里应该这样

```c++
for(size_t i=0,len=strlen(s);i<len;++i)
{
    ...
}
```

不过，按道理说现代编译器应该会自己把strlen的结果提取并缓存，这是一个很明显的循环不变量。

### Use More Efficient Loop Statements

对于for循环，编译器大产生的代码大体上是这样的

```c++
	initial-expression;
L1:	if(!condition) goto L2;
	controlled-statement;
	continue-expression;
	goto L1;
L2:
```

而对于do while语句，大体上是这样

```c++
L1: controlled-statement
    if(conditon) goto L1;
```

显然比for少了一次跳转，所以从理论上说，程序可以改成这样

```c++
size_t i=0,len=strlen(s);
do{
    if(s[i]==' ')
        s[i]='*';
}while(i<len);
```

不过，还是那句话，对于循环的优化，现代编译器已经做得很好了，所以除非用特定的编译器比较过结果，用for循环通常是没有什么问题的。

### Count Down Instead of Up

缓存结束值的另一种做法是递减计数

```c++
for(int i=(int)strlen(s)-1;i>=0;--i)
{
    ...
}
```

与前面的例子没有本质区别，不过注意这里循环变量是int，这是一个有符号数，size_t是无符号的，在递减计数是要特别注意，因为无符号数总是`>=0`的。

### Remove Invariant Code from Loops

前面的strlen已经是一个循环不变量的例子了，我们再看看其他的

```c++
int i,j,x,a[10];
...
for(i=0;i<10;++i)
{
    j=100;
    a[i]=i+j*x*x;
}
```

显然，这里应该这样

```c++
int i,j,x,a[10];
...
j=100;
int tmp=j*x*x;
for(i=0;i<10;++i)
{
	a[i]=i+tmp;
}
```

不过还是那句话，这个工作编译器通常可以自己做的，但是作为一个程序员，不要走循环中定义无关的变量应该是一个常识。而如果在循环中调用函数，编译器并不能确定其返回值是否一个循环不变量，这个工作就需要程序员来处理了。

### Remove Unneeded Function Calls from loops



## Remove Code from Functions

## Optimize Expressions

## Optimize Control Flow Idioms

## Summary



