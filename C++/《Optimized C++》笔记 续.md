

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

对于这样的代码

```c++
UsefulTool subsystem;
InputHandler input_getter;
...
while(input_getter.more_work_available())
{
    subsystem.initialize();
    subsystem.process_work(input_getter.get_work());
}
```

是否可以把`subsystem.initialize();`移到循环外，需要看情况而定。但是，这种地方总是需要关注的。

### Remove Hidden Function Calls from Loops

隐含的函数调用包括这些

- 声明类实例(调用构造函数)
- 初始化类实例(调用构造函数)
- 类实例赋值(赋值操作符)
- 包括类实例的算术表达式(调用成员操作符)
- 退出作用范围(调用类实例的析构函数)
- 函数参数(每一个实际参数表达式都会复制构造到对应的形式参数)
- 函数返回的类实例(调用复制构造函数，可能两次)
- 向标准容器插入元素(元素是移动或者复制构造的)
- vector这样的容器如果插入数据时引发重新分配，其中的元素也需要移动或者复制构造到新的空间

构造函数形式参数导致的隐藏函数调用可以通过传递引用或者指针来移除。

复制函数返回值导致的隐藏函数调用可以通过使用输出参数，也就是在参数列表中用于返回结果的参数，这类参数都是引用或者指针类型。

如果赋值或者声明是个循环不变量，那么可以将其移除循环外。如果一个变量每次迭代都需要改变状态，可以寻找代价较小的状态改变方式，比方说

```C++
for(...)
{
    std::string s("<p>");
    ...
    s+="</p>";
}
```

可以改成这样

```c++
std::string s;
for(...)
{
    s.clear();
    s+="<p>";
    ...
    s+="</p>";
}
```

这里不但减少了s的构造和析构函数的调用，而且s的空间还可以复用，可能可以减少一些对内存管理的调用。

### Remove Expensive,Slow-Changing Calls from Loops

有些函数不是不变量，但最好是。比方说日志应用中获取当前时间的调用。可以考虑获取一次时间记录多条日志的方式，不过这个方式要具体而定。

### Push Loops Down into Functions to Reduce Call Overhead

函数调用的代价总是比循环迭代的代价要大。

### Do Some Actions Less Frequently

有这么一个问题，如果一个程序的主循环每秒处理1000个事务，那么要以什么样的频率检查中止循环的命令？事实上，这取决于两件事，一个是对于中止请求的响应时间要求，一个是检查中止条件的代价。

如果需要在一秒钟内响应中止请求结束程序，并且在检测到中止命令后需要$500 \pm 100$毫秒的时间来中止程序，那么就需要每隔400毫秒检查一次(1000-(500+100))，检测次数再多就是浪费。

对于检查中止条件的代价，如果是Windows消息循环，那么对WM_CLOSE这样的消息的检测代价需要派发这个事件，没有额外的代价，如果是通过信号句柄设置一个bool标志，检测这个标志的代价也是微不足道的。

但如果是在一个嵌入式设备中，需要在循环中检测按键消息，而键被按下的消息会有个50毫秒的抖动持续时间，如果循环的每次迭代都去检测一次是否按了退出键，那么处理每个事务的时间就需要加上这50ms的代价，于是本来每秒1000次的处理频度就会变成了$\frac{1000}{51} \approx 20$每秒。但是如果改成每隔400ms处理一次退出按键的消息，为了能在这400ms的间隔里完成一次50ms的按键消息，那么按键消息就需要每隔350ms检查一次，或者每1000毫秒检查2.5次，因此事务处理频度就是$1000-(2.5*50) = 875$次每秒。看看代码就能更好的理解这里到底说了啥

```c++
void main_loop(Event evt)
{
    static unsigned counter=1;
    if((count%350)==0)
    {
        if(poll_for_exit())
            exit_program();
    }
    ++counter;
    switch(evt)
    {...}
}
```

这里假定每1ms进入main_loop一次，`poll_for_exit()`执行需要50ms，`exit_program()`需要大概400-600ms的执行时间。

当然上面只是示意说明，并不代表真是情况，要对这样的情况做优化，需要进行实测，然后确定一个可行的方案。

### Whar About Everything Else?

没什么别的了，很多说法比方说`++i`比`i++`效率高，还有什么循环展开以减少迭代次数，实际上，对于循环的优化现代编译器已经做得很好了，甚至基本上可以认为比人做得更好，所以不需要再去考虑这些犄角旮旯的东西。

## Remove Code from Functions

与循环一样，函数也包括两个部分，函数体，以及函数头，同样，也是可以分别优化的。

函数体执行的代价依函数规模而定，而函数调用的代价相比一次函数执行可能是微小的，但是如果函数需要多次调用，那么这个代价也还是会很显著的。

### Cost of Function Calls

### Declare Brief Functions Inline

### Define Functions Before First Use

### Eliminate Unused Polymorphism

### Discard Unused Interfaces

### Select Implementation at Compile Time with Templates

### Eliminate Uses of the PIMPL Idiom

### Eliminate Calls int DLLS

### Use Static Member Functions Instead of Member Functions

### Move Virtual Destructor to Base Class

## Optimize Expressions

### Simplify Expressions

### Group Constants Together

### Use Less-Expensive Operator

### Use Integer Arithmetic Instead of Floating Arithmetic

### Double May Be Faster than Float

### Replace Iterative Computations with Closed Forms

## Optimize Control Flow Idioms

### Use Switch Instead of if-elseif-else

### Use Virtual Functions Instead of switch of if

### Use No-Cost Exception Handling

## Summary



