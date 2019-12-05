# Optimized C++

[TOC]



## Chapter 1 An Overview of Optimization

### Use better compilers,and turn on the optimizer

Intel的C++编译器生成的代码性能最好，GCC性能略低，但是对标准支持最好，MSVC则介于两者之间。

### Use Better Algorithms

比方说排序查找算法的选择

### Use Better Libaries

- 标准库的性能不一定高，所以可以使用一些其他的项目，比方说Boost和Google Code，
- 函数调用的代价是比较高的，所以可以选择比方说能够提供`get_buffer()`而不仅仅是`get_char()`函数来获取一个字符的API。
- 也可以实现一些项目特定的库，通过放松一些安全性和鲁棒性要求来得到更高的速度。

### Reduce Memory Allocation and Copying

- 减少对内存管理的调用是一个非常有效的优化手段。大多数C++语言特性只是很少的指令，而调用内存管理则是以上千条来衡量的。
- 调用一次缓冲复制函数会消耗几千个指令周期，减少复制也是一个很明显的优化途径。

### Remove Computation

- 除了内存分配和函数调用外，单条C++语句的代价是微不足道的，然而在循环中突然就不一样了，大多数情况下，执行热点总是在循环中。
- 移除一两条不在循环中的内存访问语句对优化而言并没有什么实际意义。
- 现代C++编译器本身对优化代码也做了很突出的工作，比方说循环不变量的处理，`i++`和`++i`的使用。

### Use Better Data Structures

- 这是与算法紧密相关的问题。
- 这是与内存管理紧密相关的问题。
- 这是与缓存效率紧密相关的问题。

### Increase Concurrency

这个要看应用场合

### Optimize Memory Management

## Chapter 2 Computer Behavior Affecting Optimization

### Lies C++ Believes About Computers

- C++程序只是看起来好像是按顺序执行。实际上编译器和计算机本身会改变指令执行顺序。
- C++11开始，标准引入了对线程和同步的支持。
- 一些内存地址可能是硬件继承器而不是普通内存，在多线程的情况下，这些地址的内容可能在两条顺序执行的语句之间发生了变化，这就是volatile的意义。
- C++11提供了`std::atomic<>`，用来确保在一个时期内对一块内存空间的访问的原子性。但这不是volatile。

### The Truth About Computers

- 计算机的存储器的速度远远低于指令执行速度。
- 存储器不是按字节访问的，不是一个简单的线性阵列，容量也是有限的。
- 计算机快，不是执行每一条指令有多快，而是可以交叉执行多条指令

#### Memory Is Slow

这是众所周知的事情。

#### Memory Is Not Accessed in Bytes

- 没有对齐的数据访问的时间两倍于对齐的数据。
- C++编译器对齐结构的结果是可能会产生空洞，造成空间浪费。
- 所以需要注意结构中数据的大小和出现顺序。

#### Some Memory Accesses Are Slower tha Others

- 由于cache的存在，使得指令执行时间不那么确定。
- 显然，访问越多的数据对其访问的速度就越快。
- 同样，正在访问的数据周围的数据其访问速度也要比远处的数据快。
- 因此，包含循环的语句块速度要快一些，而函数调用和if语句分支会慢很多。
- 同样，vector和数组的访问速度要比那些使用指针来链接节点的数据结构快。

#### Memory Words Have a Big End and a Little End

大端小端是一个跨节点和跨平台时必须考虑的问题。

#### Memory Has Finite Capacity

- 由于cache的存在，一个函数的效率在不同情况下会很不一样，比方说单元测试和总体测试。
- 一个大程序可能会在cache中造成页面抖动，就如以前远古时代操作系统的页面抖动一样。

#### Instruction Execution Is Slow

- 内存访问决定了计算的代价。
- 对于桌面级处理器，由于使用了流水线，交叉执行的指令如果发生流水线堵塞，则会使得指令执行速度大幅下降。

#### Making Decisions Is Hard for Computers

对于分支或者跳转，如果预测失败，也是会造成流水线堵塞的。

#### There Are Multiple Streams of Program Execution

- 切换线程上下文的代价也是相当大的。
- 切换进程上下文的代价更大。
- cache和主存的数据一致性会影响多线程的执行效率。

#### Calling into the Operating System Is Expensive

系统调用的代价远远高于普通函数调用。

### C++ Tells Lies Too

#### All Statements Are Equally Expensive

即便是一个赋值语句，比方说

```c++
int i,j; 
...
i=j;
```
与
```c++
BigInstance i=OtherObject;
```

代价也是不同的。

#### Statements Are Not Executed in Order

由于指令的乱序执行，内存的延迟写入，在并发情况下，开发者必须显式考虑同步以保证一致性。

### Summary

- Access to memory dominates other cost in a processor.
- An unaligned access takes twice as long as if all the bytes were in the same word.
- Heavily used memory locations can be accessed more quickly than less heavily used locations
- Memory in adjacent locations can be accessed more quickly than memory in distant locations.
- Due to caching,a function running in the context of a whole program may run slower than the same function running in a test harness.
- It’s much slower to access data shared between threads of execution than unshared data.
- Computation is faster than decision.
- Every program competes with other programs for the computer’s resources.
- If a program must run at startup,or at times of peak load,performance must be measured under load.
- Every assignment,function argument initialization,and function return invokes a constructor,a function that can hide arbitrary amounts of code.
- Some statements hide large amounts of computation.The form of the statement does not tell how expensive it is.
- Synchronizing code reduces the amount of concurrency obtainable when concourrent threads share data.

## Chapter 3 Measure Performance

测量和实验室优化的基础。主要的工具有

- profiler，编译器提供的功能，生成一个报告表，统计每个函数在程序运行时的调用次数，还有执行的时间。最主要可以用来评估程序的热点。
- 软件计时器，程序员可以自己写一个，当然profile也有相关的信息，至少GCC的profile是有的。
- 笔记本，当然做个电子表格也是可以的，记录相关的信息。

### The Optimizing Mindset

#### Performance Must Be Measured

- 做出可测试的预测，并写下来
- 记录代码的变化
- 使用可用的最好的工具进行测量
- 对实验结果进行详细的说明

#### Optimizers Are Big Game Hunters

1%的提升并不值得花费精力和时间，但20%就值得了。

#### The 90/10 Rule

一个程序90%的运行时间花费在其10%的代码上。所以，优化的第一步就是找出这10%，并从这10%开始。当然，这只是个比方，意思就是找出热点，并从热点开始。

#### Amdahl’s Law

用来描述优化部分代码后对整体性能的提升
$$
S_T=\frac{1}{(1-P)+\frac{P}{S_p}}
$$


其中$S_T$是优化之后程序整体运行时间的提升比例，P是优化前所优化的那部分占原始执行时间的比例，$S_P$是优化后对P的提升比例。

比方说，一个程序运行时间100秒，通过profiling，发现程序有80秒在调用一个函数`f`，假定重新编码使得`f`快了30%，于是有$P=0.8，S_P=1.3/1$，那么程序的整体运行时间提升了多少？
$$
S_T=\frac{1}{(1-0.8)+\frac{0.8}{1.3}}=\frac{1}{0.2+0.62}=1.22
$$
还是同样这个程序，假如发现一个函数`g`，运行了10秒，通过改进代码，使得其效率提高了100倍，于是有$P=0.1,S_P=100/1$，整体效率就是
$$
S_T=\frac{1}{(1-P)+\frac{0.1}{100}}=\frac{1}{0.9+0.001}=1.11
$$
意义很明显了，一个是30%的改进使得整体提升了22%，一个是10000%的改进，整体提升了11%。当然，另一方面说，g的优化应该是比较容易的，也就意味着并不需要花费太多的编码时间，11%的提升也还可以接受。

### Perform Experiments

首先，代码必须是正确的的，这是优化的基础。然后考虑为什么热点会是热点？是不是在某些地方浪费了时间，有没有可改进的余地？是不是使用了昂贵的资源？

如果期望中的提升并没有出现，或者过于优秀而不像是真的，那么通常都需要重新测试，或者重新考虑之前的假定，或者检查bug，尤其是对于超出预计的提升，很有可能是改错了。

#### Keep a Lab Notebook

如果每一次试验都有文档记录，重复就很容易了。每一次运行都包括事先的假定，代码的修改，输入的数据，以及结果的记录。

#### Measure Baseline Performance and Set Goals

从用户体验的角度来说，通常有这么一些内容是需要测量的

- 启动时间。用户敲入回车或者双击鼠标之后到进入程序主界面的时间
- 关闭时间。用户发出终止命令或者点击关闭按钮到程序完全退出的时间。
- 响应时间。一条命令执行的平均时间或者最坏时间。大体上可以如下划分
    - 小于0.1秒：用户直接控制
    - 0.1秒到1秒：用户通过命令控制，这个时间是命令执行的时间
    - 1到10秒：计算机控制，用户发出命令，计算机控制执行
    - 10秒以上：喝杯咖啡去
- 吞吐量。单位时间内执行的操作。

#### You Can Improve What You Measure

优化一个单独的函数，子系统，任务或者测试用例并不等同于对整个程序的优化。比方说，优化一个对数据库的查询语句，并不意味着对数据库本身的优化，也不意味着对这个数据库的删除或者更新等等操作的优化。

### Profile Program Execution

Profiler是一个程序，用来统计其它程序耗费的时间，用报告来说明一条语句或者一个函数执行的频度以及累计的时间。

在Windows和Linux环境下使用profiler的方法大体如下：

1. 编译时使用特定的参数，比方说GCC的`-pg`。
2. 编译链接程序。
3. 运行程序，会在磁盘上生成一个profile文件。
4. profiler程序会以这个表格为输入，生成一个文本或者图形格式的报告，比方说适用于GCC的profiler是gprof。

总之，就是加上参数编译，运行程序，程序终止后就可以找到一个profile表格文件，然后用profiler程序处理这个文件，生成一个报告。

### Timing Long-Running Code

#### “A Little Learning” About Measuring Time

#### Measuring Time with Computers

#### Overcoming Measurement Obstacles

#### Create a Stopwatch Class

#### Time Hot Functions in a Test Harness

### Estimate Code Cost to Find Hot Code

#### Estimate the Cost of Individual C++ Statements

#### Estimate the Cost of Loops

### Other Ways to Find Hot Spots

### Summary