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

总之，就是加上参数编译，运行程序，程序终止后就可以找到一个profile表格文件，然后用profiler程序处理这个文件，生成一个报告。生成的报告格式也多种多样，但不管怎样，都可以很直观的看到热点，于是可以根据这份报告逐个检查热点，直到都不那么热或者找不到优化的余地为止。通常来说，用debug版本来做profile要比release版本更合适，因为debug版本会覆盖全部代码，而release版由于编译时对代码做了一些处理，会有所不同，比方说debug版本包括所有的函数调用，包括inline函数，而release版本通常inline函数已经展开了。

profiler也有一些不足的地方：

- profiler不会告诉你某个地方可以使用效率更高的算法。调整一个糟糕的算法是很费时间的。
- 对于同一个输入，如果用来执行不同的任务，profiler得到的结果可能不是那么准确。比方说数据库select和insert的代码是不同的，当执行select时insert可能根本就不会进行，因此，混合了select和insert的代码，在profiler报告中可能就看不出来insert其实是个热点。所以，一次只执行一个优化。
- 在I/O密集或者多线程的情况下，profiler的报告可能会有所误导，因为，profiler会减去系统调用或者等待事件的时间。不过，一些profiler会同时列出函数调用次数和执行时间，这也会是一个线索。

### Timing Long-Running Code

如果一个程序只做一件事，profiler可以很容易的看出热点，但是如果一个程序做了很多不同的事，那么热点可能就不那么明显。这个时候就需要手工计时了。

通过不断的计时缩小范围，找到候选的热点代码段，然后进行单独的实验。大多数情况下，程序员需要自己写个计时程序。

#### “A Little Learning” About Measuring Time

关于误差分析学生时代做过实验的都应该知道。

#### Measuring Time with Computers

由于各种软硬件方面的原因，对于计时产生的误差心里要有数。

#### Overcoming Measurement Obstacles

需要进行大量的试验以平均化各种外部因素的影响，但是不要忘记精确度和准确度是两回事。

#### Create a Stopwatch Class

秒表类模板

```c++
template<typename T>
class BasicStopwatch:T
{
    	using BaseTimer=T;
	public:
    	explicit BasicStopwatch(bool start);
    	explicit BasicStopwatch(char const* activity="Stopwatch",bool start=true);
    	BasicStopwatch(std::ostream& log,char const* activity="Stopwatch",bool start=true);
    
    	~BasicStopwatch();
    
    	unsigned lapGet() const;
    	bool isStarted() const;
    	unsigned show(char const* event="show");    
    	unsigned start(char const* event_name="start");
    	unsigned stop(char const* event_name="stop");
    
    private:
    	char const* m_activity;
    	unsigned m_lap;
    	std::ostream& m_log;    	
};
```

下面是BaseTime的几个不同实现

1. 使用chrono库，移植性比较好，需要c++11

```c++
#include <chrono>
class TimerBase
{
    public:
    	TimerBase():m_start(system_clock::time_point::min()){}
    	
    	void clear()
        {
            m_start=system_clock::time_point::min();
        }
    
    	bool isStarted() const
        {
            return (m_start.time_since_epoch()!=system_clock::duraton(0));
        }
    
    	void start()
        {
            m_start=system_clock::now();
        }
    
    	unsigned long getMs()
        {
            if(isStarted())
            {
                system_clock::duraton diff;
                diff=system_clock::now()-m_start;
                return (unsigned)(duration_cast<milliseconds>(diff).count());
            }
            return 0;
        }
    private:
    	system_clock::time_point m_start;
};
```
2. 使用`clock()`，适用于linux和Windows
```c++
class TimerBaseClock
{
    public:
    	TimerBaseClock() { m_start=-1;}
    
    	void clear() {m_start=-1;}
    	bool isStarted() const {return m_start!=-1;}
    
    	void start() {m_start=clock();}
    	unsigned long getMs()
        {
            clock_t now;
            if(isStarted())
            {
                now=clock();
                clock_t dt=now-m_start;
                return (unsigned long)(dt*1000/CLOCK_PER_SEC);
            }
            return 0;
        }
    private:
    	clock_t m_start;    	
};
```

#### Time Hot Functions in a Test Harness

如前面所说，要得到更精确的结果需要多次测量

```c++
using counter_t=unsigned;
counter_t const iterations=10000;
{
    Stopwatch sw("function_to_be timed()");
    for(counter_t i=0;i<iterations;++i)
    {
        result=function_to_be_timed();
    }
}
```

注意最外层的`{}`，这样使得sw可以使用RAII。

### Estimate Code Cost to Find Hot Code

profiler可以指出最常调用的函数，或者整体运行时间中占比最高的部分，但不一定能指出问题在哪条语句上。手工计时也不一定能明确指出是哪里的问题。这时候就需要估算其中每一条语句的代价了。

#### Estimate the Cost of Individual C++ Statements

内存访问的时间代价远超过其它指令执行的代价。理论上，一条指令执行的时间包括从内存中读取指令的时间，加上从内存中读取输入数据的时间，加上向内存中写入结果的时间，而指令解码及执行的时间基本上可以忽略不计。不过，在流水线的情况下，读取指令的时间大体上也可以忽略，所以，要评估一条C++语句的时间代价，基本上可以通过对这条语句读取和写入内存的次数进行计数得到。比方说对`a=b+c`，包括了对b和c的读取，以及对a的写入，也就是两次读取，一次写入。再比如对`r=*p+a[i]`，包括对`i`的一次读取，对`a[i]`的一次读取，对`p`的一次读取，对`*p`所指的一次读取，以及对`r`的一次写入，也就是总共5次对内存的访问。

#### Estimate the Cost of Loops

对于循环的分析，可以参考算法课或者数据结构课上关于时间复杂度的分析，那里更为全面一些。不过要注意的是一些隐式的循环，比方说事件驱动的事件循环，这是隐藏在框架后面的，只要有足够的事件，也总是可以找到热点事件的。

### Other Ways to Find Hot Spots

这里所谓另外的方法指的是程序员的本能，但是这没有什么意义，要说明问题或者说服别人，还是需要数据，比方说profiler的结果，计时的结果，以及算法分析的结果，但是算法分析的过程及结果对普通人来说可能不好接受。

### Summary

- Perfomance must be measured.
- Make testable predictions,and write the predictions down.
- Make a record of code changes.
- If each experiment run is documented,it can quickly be repeated.
- A program spends 90% of its run time in 10% of its code.
- A measurement must be both true and precise to be accurate.
- Resolution is not accuracy.
- On Windows,the function `clock()` provides a reliable 1-millisecond clock tick.For Windows8 and later,the function `GetSystemTimePreciseAsFileTime()` provides a submicrosecond tick.
- Accepting only large changes in performance frees the developer from worrying about methodology.
- To estimate how expensive a C++ statement is,count the number of memory reads and writes performed by the statement.

## Chapter 4 Optimize String Use:A Case Study

### Why Strings Are a Problem
- string是动态分配的

- strings是值

- string会有很多的复制操作

### First Attempt at Optimizing Strings

```c++
std::string remove_ctrl(std::string s)
{
    std::string result;
    for(int i=0;i<s.length();++i)
    {
        if(s[i]>=0x20)
            result=result+s[i];
    }
    return result;
}
```

这段代码在性能上有很大的问题。主要问题字符串的连接是很昂贵的。在这里，每次连接操作(`+`)都会调用内存管理来构建一个临时对象，然后将这个临时对象赋值给result，这也意味着一次复制操作。而且在进行连接操作前，会把result复制到临时对象里去，因此，如果s的长度为n，那么就会做$n^2$次复制。换句话说，这个算法实际上是O($n^2$)的。

#### Use Mutating String Operations to Eliminate Temporaries

```c++
std::string remove_ctrl_mutating(std::string s)
{
    std::string result;
    for(int i=0;i<s.length();++i)
    {
        if(s[i]>=0x20)
            result+=s[i];
    }
    return result;
}
```

这样的改进去除了对临时对象的所有操作：构造，复制和释放。并且也去除了赋值相关分配和复制操作。然而，由于result的长度是个缺省值，因此，还是有可能要时不时的增加其内置存储的长度。

#### Reduce Reallocation by Reserving Storage

```c++
std::string remove_ctrl_reserve(std::string s)
{
    std::string result;
    result.reserve(s.length());
    for(int i=0;i<s.length();++i)
    {
        if(s[i]>=0x20)
            result+=s[i];
    }
    return result;
}
```

`reserve()`不但可以去除对string内置存储区的重新分配，并且提高了函数对cache的访问效率，这是出于局部性原理。

然而，这里还有一个问题，就是s作为值传入，那么就会涉及到临时对象的创建和复制。我们也看到，s是不会被更改的，所以可以作为一个const的引用传入。

#### Eliminate Copying of String Arguments

```c++
std::string remove_ctrl_ref_args(std::string const& s)
{
    std::string result;
    result.reserve(s.length());
    for(int i=0;i<s.length();++i)
    {
        if(s[i]>=0x20)
            result+=s[i];
    }
    return result;
}
```

然而，这样更改实际测试效果并不一定有效。这与引用的实现有关，引用通常是指针实现的，在循环中对s的每次使用，都需要去解引用，这样也是会影响执行效率的。

#### Eliminate Pointer Dereference Using Iterators

```c++
std::string remove_ctrl_ref_args_it(std::string const& s)
{
    std::string result;
    result.reserve(s.length());
    for(auto it=s.begin(),end=s.end();it!=end;++it)
    {
        if(*it>=0x20)
            result+=*it;
    }
    return result;
}
```

通常来说，迭代器要比下标快，并且不需要每次都对s进行解引用。另外就是end循环开始就获取了，就不用每次迭代再取一次，当然这个工作编译器应该能够实现，毕竟这是一个很明显的循环不变量。

最后，还有一个问题，就是对结果的复制。

#### Eliminate Copying of Returned String Values

```c++
void remove_ctrl_ref_result_it(std::string& result,std::string const& s)
{
    result.clear();
    result.reserve(s.length());
    for(auto it=s.begin(),end=s.end();it!=end;++it)
    {
        if(*it>=0x20)
            result+=*it;
    }
}
```

在很多时候，这个版本不需要任何的分配操作。这里的问题在于其接口可能会被误用，比方说

```c++
std::string foo("this is a string");
remove_ctrl_ref_result_it(foo,foo);
```

其结果会是一个空串。

然而，事情并没有结束。

#### Use Character Arrays Instead of Strings

```c++
void remove_ctrl_cstring(char * destp,char const* srcp,size_t size)
{
    for(size_t i=0;i<size;++i)
    {
        if(srcp[i]>=0x20)
            *destp++=srcp[i];
    }
    *destp=0;
}
```

显然，这是效率最高的版本了。这里只是要考虑对于destp是动态分配一个空间然后释放，还是静态的声明一个数组。其实差别不大，对整体效率而言。

#### Summary of First Optimization Attempt

|Function| Debug(μs) |  Δ(%) | Release(μs) |  Δ(%)| Release vs. debug(%)|
| :---------: | :---: | :--: | :-----: | :--: | :---------------: |
| remove_ctrl() | 967  |      |     24.8    |      |3802          |
|remove_ctrl_mutating()|104|834|1.72|1341|5923|
|remove_ctrl_reserve()|102|142|1.47|17|6853|
|remove_ctrl_ref_args_it()|215|9|1.04|21|20559|
|remove_ctrl_ref_result_it()|215|0|1.02|2|21012|
|remove_ctrl_cstrings()|1|9698|0.15|601|559|

上面的结果看大体趋势就行，不需要纠结具体的值。

以上的优化立足于一个简单的原则：移除内存分配和相关的复制操作。

### Second Attempt at Optimizing Strings

#### Use a Better Algorithm

在前面的算法里，每次只复制一个字符，这很有可能造成不断地重新分配内存。

```c++
std::string remove_ctrl_block(std::string s)
{
    std::string result;
    for(size_t b=0,i=b,e=s.length();b<e;b=i+1)
    {
        for(i=b;i<e;++i)
        {
            if(s[i]<0x20)
                break;
        }
        result=result+s.substr(b,i-b);
    }
}
```

这里有两个改进

1. 不是复制一个字符，而是把符合条件的字符一起复制到结果中去。
2. 源字符串的长度先保存起来了，不用每次迭代都去计算一次。当然，这个工作正常的编译器会处理的

但是，还是有改进的余地

1. 如前面所示，result的长度可以事先确定，
2. 可以使用`+=`而不是`+`，或者使用append函数，这样可以不用创建临时对象。

```c++
std::string remove_ctrl_block_append(std::string s)
{
    std::string result;
    result.reserve(s.length());
    for(size_t b=0;i=b,e=s.length();b<e;b=i+1)
    {
        for(i=b;i<e;++i)
        {
            if(s[i]<0x20) 
                break;
        }
        result.append(s,b,i-b);
    }
}
```

当然，这里的主要改进是每次复制一块数据，而不是一个字节的数据，前面提到过的改进还是可以结合使用的，比方说用迭代器而不是使用下标运算。

另外，虽然这里有两层for循环，但时间复杂度仍然是O(n)。

最后，还有一个方法，就是直接从源字符串中删除控制符

```c++
std::string remove_ctrl_erase(std::string s)
{
    for(size_t i=0;i<s.length();)
    {
        if(s[i]<0x20)
            s.erase(i,1);
        else
            ++i;
    }
    return s;
}
```

这种方法很好，但是这种方法的语义可能不是想要的，或者说与大多数string操作的语义不一致。

#### Use a Better Complier

#### Use a Better String Library

c++标准要求迭代器是随机访问的，并且禁止写时复制(COW)。

#### Use a Better Allocator

### Eliminater String Conversion

字符串转换可能意味着字符拷贝和动态分配内存，这也就意味着有优化的余地。

#### Conversion from C String to std::string

一个常见的浪费计算机时钟的毫无必要的转换是从一个null结尾的字符串转换成std::string，比如

```c++
std::string MyClass::Name() const
{
    return "MyClass";
}
```

这里需要把一个字符串常量转化成一个std::string，因此需要分配空间，复制字符。

但事实上这里并不需要做这个转换，可以留到使用的地方再由编译器自动转换，比方说赋值给一个string或者作为一个函数参数

```c++
char const* MyClass::Name() const
{
    return "MyClass";
}
```

由于”MyClass”是个字面量，并不会因为Name结束而不知所踪。

在一个大系统当中，通常是分层的，可能会出现一个string转换成const char*，然后这个const char*又转换成string的情况，所以还不如直接使用一个字面量。

#### Conversion Between Character Encodings

建议使用UTF-8，兼容C风格的字符串。

### Summary

- Strings are expensive to use because they are dynamically allocated,they behave as values in expressions,and their implementation requires a lot of copying.
- Treating strings as objects instead of values reduces the frequency of allocation and copying.
- Reserving space in a string reduces the overhead of allocation.
- Passing a const reference to a string into a function is almost the same as passing the value,but can be more efficient.
- Passing a result string out of a function as a reference reuses the actual argument’s storage,which is potentially mor efficient than allocating new storage.
- An optimization that only sometimes removes allocation overhead is still an optimization.
- Sometimes a different algorithm is easier to optimize or inherently more efficient.
- The standard library class implementations are general-purpose and simple.They are not necessarily high-performance,or optimal for any particular use.

## Chapter 5 Optimize Algorithms

### Time Cost of Algorithms

- O(1)，常量时间
- O($log_2n$)
- O(n)，线性时间
- O($nlog_2n$)
- O($n^2$)，O($n^3$)等等
- O($2^n$)

详细的就不讨论了，找本算法书说的更加全面。

#### Amotized Time Cost

比方说，向一个堆中插入数据需要O($log_2n$)的时间，因此用一次插入一个元素的方法构建一个堆需要O($nlog_2n$)的时间。然而，最高效的堆构建算法是O(n)，也就是说通过这个方法均摊到每一个元素插入只需要O(1)的时间，高效算法并不是每次插入一个元素，而是通过分治法处理一系列的子堆得到。这里的意思是，通过使用高效的构建算法，向堆中插入元素的时间变成O(1)而不是O($log_2n$)，当然这只是对构建堆的时候的分析，并不是在普通情况下向堆插入元素的时间。

在比如说，向一个std::string插入字符的均摊时间是个常数，但是其中包含了对内存管理的调用，如果string比较短，那么有可能每次插入都会调用内存管理，只有当string比较长的时候，这个均摊时间才会变得比较低，当然这是string重新分配缓冲算法的原因。

#### Other Costs

除了时间代价，还有空间代价，比方说递归算法，必然需要一个栈。通常来说，时间和空间的矛盾在算法效率评估上是一直存在的。

### Toolkit to Optimize Searching and Sorting

- 用平均时间更好的算法代替平均时间糟糕的算法
- 考虑数据结构，选用更适合特定数据结构的算法
- 调整算法以获取更好的常量因子。

### Efficient Search Algorithms

#### Time Cost of Searching Algorithms

- 线性查找，O(n)。

    这个没什么好说的，记住一点就是对于基本有序的线性表，查找效率并不是很低。另外，如果把每次查找到的结果插入到表头，也会提高查找效率，这一点通常算法教科书是没有提到的，虽然局部性原理很多课程都会提到。

- 二分查找，O($log_2n$)。

    二分查找需要输入数据有序。实际上，对输入数据先进行排序，对将来的操作都是有益的。

- 插值查找，最好情况O($log log n$)。

    跟二分查找一样，需要对输入数据分区，但是分区的标准立足于对输入数据的更多的了解，从而达到更好的分区效果。比方说，要在一个有序的数值线性表中查找一个数x，二分查找的话通常就是$mid=\frac{high+low}{2}$，而插值查找则可以是$mid=low+\frac{(x-A[low])*(high-low)}{A[high]-A[low]}$。如果表很大或者检查表中每一条数据的代价比较高，比方说磁盘上的数据，那么使用插值插值提升是很大的。

- 哈希表是有可能达到O(1)的算法，哈希表最坏情况是O(n)，但是hash表会浪费一些空间。

#### All Searches Are Equal When n Is Small

这里很小指的是只有几个数据的表，表越大，差异自然也就越大

| 表大小 | 线性 | 二分 | 哈希 |
| :----: | :--: | :--: | :--: |
|   1    |  1   |  1   |  1   |
|   2    |  1   |  2   |  1   |
|   4    |  2   |  3   |  1   |
|        |  4   |  4   |  1   |
|   16   |  8   |  5   |  1   |
|   26   |  13  |  6   |  1   |
|   32   |      |  6   |  1   |

### Efficient Sort Algorithms

- 说排序算法时间最好的是O($nlog_2n$)是不对的。首先这是通过比较输入数据的值来排序的算法的最好的时间复杂度，对于像基数排序这样，不比较关键字的值进行排序的算法来说，则是不对的。其次，如果排序关键字具备某种属性，比方说一个从1到n的整数集合，flashsort可以达到O(n)。
- 快速排序最坏情况是O($n^2$)，并且最坏情况是不可避免的。
- 有些排序，包括插入排序，在输入数据基本有序的时候可以得到线性的时间，而快速排序在这种情况下如果选择轴的方法不当，可能会变成O($n^2$)的。

#### Time Cost of Sorting Algorithms

| 排序算法  | 最好情况  | 平均情况  | 最坏情况  | 所需空间 | 备注                                 |
| :-------: | :-------: | :-------: | :-------: | :------: | :----------------------------------- |
| 插入排序  |     n     |   $n^2$   |   $n^2$   |    1     | 最好情况是输入数据有序或者基本有序   |
| 快速排序  | $nlog_2n$ | $nlog_2n$ |   $n^2$   | $log_2n$ | 最坏情况是输入数据有序并且轴选择不当 |
| 归并排序  | $nlog_2n$ | $nlog_2n$ | $nlog_2n$ |    1     |                                      |
|  树排序   | $nlog_2n$ | $nlog_2n$ | $nlog_2n$ |    n     |                                      |
|  堆排序   | $nlog_2n$ | $nlog_2n$ | $nlog_2n$ |    1     |                                      |
|  timsort  |     n     | $nlog_2n$ | $nlog_2n$ |    n     | 最好情况是输入数据有序               |
| introsort | $nlog_2n$ | $nlog_2n$ | $nlog_2n$ |    1     |                                      |

#### Replaces Sorts Having Poor Worst-Case Performance

如果对输入数据一无所知，那么归并排序，树排序和堆排序通常可以提供一个比较正常时间。如果一定要使用快速排序，那么就必须好好考虑轴的选取，但是总有运气不好的时候。

#### Exploit Known Properties of the Input Data

如果已知输入数据是有序或者基本有序的，那么插入排序可以提供O(n)的性能。

Timsort是混合了归并排序和插入排序的一种算法，在输入数据有序或者基本有序时也可以得到O(n)的性能，其最坏性能也不过O($nlog_2n$)，目前这是python的标准排序算法。

Introsort混合了快速排序和堆排序，首先会做快速排序，当发现递归层次过深时，就切换成堆排序。这是C++11中std::sort()优先使用的算法。

flashsort有O(n)的时间，如果输入数据服从某种分布的话。类似于基数排序，不过数据会根据概率放到不同的桶中。

### Optimization Patterns

#### Precomputation

把计算提前到进入热点之前、到链接时，到编译时，甚至设计时，通常来说，越早越好。

当然，预结算只能把与当前上下文无关的计算提前到上下文之前。

- C++编译器自动预计算常量表达式的值，包括const，constexpr，inline static等等。
- 模板函数调用会在编译器计算一些特定参数的值。
- 设计者能够找到一些特定的值，比方说月的天数等，这些数在设计时就可以确定。

#### Lazy Computation

延迟计算的目的是把计算推迟到接近使用的地方，比方说循环变量，if-then-else分支。还有

- 两步构造

    构造函数创建一个空白或者只包含最少必要信息的对象，然后在时机合适或者说需要的时候在调用一个初始化函数完成对象构造。通常在创建一个静态对象时，由于信息不足，只能采用这种方式。并且，延迟构造可以构造出一个高效的扁平的数据结构。

    有时需要检查一个延迟构造的值是否已经构造，这会带来一些代价，这个代价与检查一个指针是否有效是一样的。

- 写时复制

#### Batching

批处理的目的是一次处理尽可能多的对象。比方说

- 字符串处理时，把字符添加到一个缓冲中，直到缓冲满或者字符输入结束，然后再把这个缓冲传送到处理例程中，避免每一个字符都调用一次这个处理例程。
- 一个无序的数组转换成堆也是一个例子。每次向堆中插入一个元素的算法时间是O($nlog_2n$)，而一次构建堆的算法时间是O(n)。
- 多线程的任务队列。
- 后台保存或者更新。

#### Caching

缓存指的是保存然后重用计算的结果，而不是每次重新计算。比如

- 像`a[i][j]=a[i][j]+c;`这样的语句，编译器会处理成大概这样:`auto p=&a[i][j];*p=*p+c;`
- 缓存，现代cpu都有的
- C风格字符串的长度每次都需要计数，而std::string则把字符串长度保存起来。
- 线程池
- 动态规划算法

#### Specialization

特化是泛化的反义词。减少某个特定情况下不需要的计算，通过拿走一些限制，或者添加一些限制，比方说动态的变成静态，未绑定的变成绑定等等，在模板编程中就有很多这样的例子。

####Taking Bigger Bites

目的是减少重复操作的迭代次数。包括

- 向操作系统请求更大块的数据，或者向操作系统发送更大块的数据，以减少调用内核的代价。不过如果程序崩溃可能会丢失数据，特别是对于日志来说可能会是个问题。
- 按字或者长字移动和清除缓冲。当然需要缓冲是边界对齐的。
- 按字或者长字来比较字符串。只对大端的机器有效，或者说网络字节序的数据，小端的比方说x86就无效了。这个说法比较可疑，不同的字节序，当然不能直接比较，但对于同样的字节序，没有理由不能按字来比较。
- 一个线程尽可能做更多的工作，免得不断的切换上下文。
- 一些维持性的工作，比方说心跳什么的，不要每次迭代都执行一次，可以每10次，每100次执行一次。

#### Hinting

比方说，有一个std::map的插入函数，提供一个在哪里插入的参数，那么这个插入函数的时间就会是O(1)，否则，就是O($log_2n$)。

#### Optimizing the Expected Path

在分支语句中，把概率大的分支放在前面。

#### Hashing

哈希值可以缓存起来，避免重复计算。也可以与双重检查结合起来优化比较操作。

#### Double-Checking

先进性一次代价较小的检查，如果有必要再做一次代价比较大的检查。

- 双重检查经常与缓存结合使用。就是一级一级的往下检查数据是否存在，每下一级代价就会大一些。
- 比较两个std::string的时候可以先检查两者的长度，然后再做字符比较。
- 在哈希中使用，如果两个数据的哈希值不同，它们肯定是不同的，如果哈希值相同，再逐个字节比较。

### Summary

- Beware of strangers selling constant-time algorithms.They may be O(n).
- Multiple efficient algorithms can be combined in ways that make their overall run time O($n^2$) or worse.
- Binary search,an O($log_2n$),isn’t the fastest search.Interpolation search is O($log log n$) and hashing is constant-time.
- For small tables with <4 entries,all search algorithms examine about the same number of entries.

## Chapter 6 Optimize Dynamically Allocated Variables

除了低效算法，动态分配内存的不当使用是最大的性能杀手。

### C++ Variables Refresher

每一个变量在内存布局中都占有一个位置，其大小是编译时确定的。需要特别说明的是一个union的布局是依赖于实现的。

#### Storage Duration of Variables

每一个变量都有一个生存期。为变量分配内存的代价取决于变量的生存期。C++中变量的生存期是没有显式指定的，但是可以从变量的声明中推导出来

- 静态存储生存期

    在名字空间域声明，或者有static或者extern修饰声明的变量具有静态生存期。有静态生存期的变量位于编译器保留的存储区中，其地址和大小在编译期就决定了，其生存期直到程序结束。

    - 每一个全局的静态变量在进入main()前构建，离开main()之后销毁。
    - 函数域的静态变量只是在第一次进入函数执行前构建，可能与全局变量一样早，也可能仅仅只在第一次调用该函数的时候。

    每一个静态变量都用名字标识，可以通过指针或者引用访问。即使一个静态变量本身还没有赋给有意义的值或者其值已经被销毁，其名字、指针或者引用也可以出现在其他静态变量的构造和析构中，因为静态变量在其被使用前就必然已经存在了，这是标准规定并且编译器必须实现的。

    创建静态变量没有运行时代价，但是其存储区也不能复用。因此静态变量适用于持续整个程序生存的数据。

- 线程局部存储生存期

    在c++11中，用thread_local修饰声明的变量具有线程局部生存期。线程局部变量在进入线程时创建，离开线程时销毁，其生存期与所在线程相同。注意每一个线程都有其线程局部变量的副本。

    对线程局部变量的访问代价要高于静态变量。有些系统线程局部变量是线程分配的，与静态变量相比，对其访问的代价就只是一个额外的间接访问的代价。但有些系统中线程局部变量的访问是通过一个全局线程表进行的，虽然对这个表的访问是常量时间，但是需要一个函数调用和一些额外的计算，所以代价还是可观的。

- 自动存储生存期

    函数的实际参数有自动生存期，代码块中声明的变量如果没有其他修饰的话也具有自动生存期。具有自动生存期的变量位于栈中。自动变量的地址是浮动的，只有到了执行的时候才会有一个绝对地址。

    自动变量的生存期只限于包含该变量的代码块的执行期间。自动变量也是通过名字、指针或者引用访问，但是其名字只在创建之后销毁之前才是可见的。

    对自动变量的分配没有运行期代价，但是对于自动变量有个总数的限制，比方说在递归中可能会造成栈溢出就是所分配的内存总量超过了限制。

- 动态存储生存期

    由new表达式返回的变量就拥有动态存储生存期。动态生存期的变量其生存期由程序运行时决定。程序显式的调用new表达式获得存储空间并构造对象，显式的调用delete表达式销毁变量并返还存储空间。

    动态变量的地址是运行时决定的。一个动态变量没有名字，内存管理只是返回这个变量的指针，这个指针是必须的，只有通过这个指针，程序才有可能返回存储空间。

    动态变量的类型和数量没有限制。

    管理动态变量使用的内存是有显著的运行时代价的。

#### Ownership of Variables

C++变量的另外一个重要概念是所有权。一个变量的所有者可以决定这个变量什么时候创建和什么时候销毁。这与存储生存期是两个概念，要体会下两者的区别。

- 全局所有

    静态生存期的变量。

- 词法域所有

    词法域指的是由`{}`括起来的代码块，在其中的自动存储生存期的变量就具有这种所有权。

- 成员所有

    类类型的成员，其生存期与类实例的生存期一致。当然类静态成员除外。

- 动态变量所有

    动态变量没有预先指定的所有者。new表达式返回所创建动态变量的指针，这个指针必须由程序显式管理。而这个动态变量通过delete表达式返还内存管理。因此，动态变量的生存期是可编程的。动态变量对于优化是很重要的，因为动态变量并不由编译器控制，也不是C++定义，而是完全由开发者控制的。

#### Value Objects and Entity Objects

一些变量的意义来着于它们的内容，称为值对象。一些变量的意义来自于它们在程序中的角色，称之实体或者实体对象。

C++本身并不决定一个变量是值变量还是实体变量，这是由程序的逻辑决定的。比方说C++允许自定义复制构造函数和`operator==`，其所归属或者操作的变量的角色决定了程序员是否应该自定义这些操作。如果程序员没有采取步骤禁用没有意义的操作，编译器也不会抱怨说定义了无用的操作。

实体对象有一些公共的属性：

- 实体是唯一的。程序中的一些对象在概念上拥有一个唯一的标识。mutex变量就是个例子。
- 实体是可变的。不过改变一个实体的状态并不改变其在程序中的基本意义。
- 实体是不可复制的。实体的自然属性来自于它们的使用方式，而不是它们占据的二进制位。比方说复制一个mutex变量并没有什么意义。
- 实体是不可比较的。比方说比较两个mutex的意义在哪里呢？

对应的，值变量也有一些公共属性：

- 值是可以互换以及比较的。值的意义来自它们占据的二进制位而不是它们在程序中的使用。
- 值是不可改变的。比方说不可能把一个值4改成5，可以把一个值为4的变量的值改成5，但这里改动的是变量这个实体，而不是4这个值。
- 值是可以复制的。两个字符串可以有同样一个值”foo"，这并不会影响程序的正确性。

一个类成员变量是实体还是值决定了这个类的复制构造函数的写法，类实例可以共享一个实体，但是不能合法的复制这个实体。理解一个变量是实体对象还是值对象对于优化的重要性在于，实体对象通常由许多动态变量构成，如果可以复制的话，其代价是很大的。

### C++ Dynamic Variable API Refresher

- 指针和引用

    C++的动态变量没有名字，它们通过指针或者引用来访问。指针抽象了物理地址，隐藏了计算机体系架构的复杂性和易变性。一个指针变量的值并不总是指向可用的内存位置，比方说nullpt或者0。引用变量必须初始化，所以其总是可以指向一个合法的位置。

- new-和delete-表达式

    从指针本身并不能知道其指向的是一个标量还是一个数组，程序员只能自己记住，并使用配对的`new/delete`或者`new []/delete []`。

    ```c++
    {
        int n=100;
        char *p;
        Book* bp=nullptr;
        ...
        cp=new char[n]; //动态数组
        bp=new Book("Optimized C++"); //动态类实例
        ...
        char array[sizeof(Book)];
        Book* bp2=new(array) Book("Moby Dick"); //放置new
        ...
        delete [] cp;
        cp=new char;
        ...
        delete bp;
        delete cp;
        bp2->~Book();  
    }
    ```

- 内存管理函数

    就是重载的`operator new()/operator new[]()`和对应的`operator delete()/operator delete[]()`，以及C函数`malloc()`和`free()`等。

- 类构造函数和析构函数

    可以把new-和delete-表达式分别放到这两个函数中，这样可以实现某种程度上的自动内存管理。

- 智能指针

    这是一个类模板

- 分配器模板

    标准库提供的分配器模板，泛化了new-和delete-表达式，主要用在标准容器中。

#### Smart Pointers Automate Ownership of Dynamic Variables

有一种常见技巧是把一个指针作为类的私有成员，在构造函数中初始化，在析构函数中销毁，从而对这个指针的管理就由类来进行，这样就可以避免指针或者说动态变量所有者不明确的问题，同时也实现了某种安全机制。

智能指针事实上也是这种思想的体现。通常来说`std::unique_ptr<T>`的使用代价并不大，是一种值得使用的机制。

至于共享指针，一个很常见的场景就是把指针作为函数参数传入，这样一个变量就会有两个指针指向，在程序中这是不可避免的，这也是很多错误的源头，找出重复释放的指针很多时候是一件苦差事。因此标准引入了`std::shared_ptr<T>`，通过引用计数来避免前述问题，也很适用于多线程的程序中。`std::shared_ptr<T>`的问题只在于使用代价会高一些。

如果把一个C类型指针赋给多个智能指针，那么这个C类型指针有可能会被释放多次，从而造成标准所说的未定义行为。

另外，忘掉`std::auto_ptr<T>`吧。

#### Dynamic Variables Have Runtime Cost

为一个动态变量分配内存可能需要上千次的访问内存，因为这个工作包括了寻找空闲空间，如果没有足够的空闲空间还需要向操作系统申请空间等等。而释放空间也是一项很复杂的工作，同时收集空闲空间也耗费巨大。

### Reduce Use of Dynamic Variables

#### Create Class Instance Statically

尽可能的使用自动的类实例，而不是使用new来动态创建一个类实例。

对于类成员，主要是指成员类，可以使用两步初始化，也就是说在类实例化时，由于其成员类所需的资源可能还没有得到满足，可以先实例化一个空白类，到了将来资源齐备了，再通过一个初始化函数将其完全实例化。两步初始化并没有额外的代价，检查其是否完备与检查一个指针是否nullptr的代价是一样的。另外，初始化函数可以返回错误，而构造函数是没有的。

两步初始化特别适用于在初始化时会比较耗费时间的类，比方说读文件，获取http响应。

#### Use Static Data Structures

`std::string`、`std::vector`、`std::map`、`std::list`这些都是日常使用的容器，但是要意识到`std::string`和`std::vector`可能会自动重新分配空间，而`std::map`和`std::list`在每次插入数据时都会分配新的节点，因此，它们的效率可能不是那么高。

- 使用`std::array`取代`std::vector`。`std::array`是长度固定的，其接口也是兼容标准库容器的，所以如果能够大致确定数据量，使用`std::arry`或者直接使用数组都是可以的。但是要明白`std::vector`的优点就在其可变性，虽然这种灵活性需要代价。

- 在栈上创建大缓冲。就是把数据复制到本地创建的一个数组当中，然后对这个数组进行处理，最后可能需要把结果再复制一遍。总的来说，这种复制要比使用`std::string`快，一则，对数据内容的更改，就`std::string`而言，也是隐含着复制操作的，二则，复制操作的代价要小于动态内存分配。

- 静态创建链接的数据结构。比方说

    ```c++
    struct treenode
    {
        char const* name;
        treenode* left;
        treenode* right;
    } tree[]=
    	{
        	{"D",&tree[1],&tree[2]},  //tree[0]
        	{"B",&tree[3],&tree[4]},  //tree[1]
        	{"F",&tree[5],nullptr},   //tree[2]
        	{"A",nullptr,nullptr},    //tree[3]
        	{"C",nullptr,nullptr},	  //tree[4]
        	{"E",nullptr,nullptr}	  //tree[5]
    	};
    ```

    但是这种方式比较容易出错，而且没有扩展性。

    另外一种方式则是

    ```c++
    struct cyclenode
    {
        char const* name;
        cyclenode* next;
    };
    extern cyclenode first; //前向引用
    cyclenode fourth={"4",&first};
    cyclenode third={"3",&fourth};
    cyclenode second={"2",&third};
    cyclenode first={"1",&second};
    ```

    显然，这种方式更为实用一些，对于一个动态的数据结构而言。

    总之，就是在第一感需要动态变量的地方，可以再考虑下是否能够使用自动变量，比方说就上面这个例子，new一个节点，再修改节点指针，这一系列操作通过使用自动变量也是可以实现的。不过显见的问题调试的时候可能会看的眼花，没有动态变量那么直观。

- 在数组中创建二叉树。对于变化比较大的二叉树显然并不怎么合适，不过这种方式往往是堆的实现方式。

- 使用循环缓冲而不是deque。总体而言，循环缓冲的性能类似于`std::vector`。

#### Use `std::make_shared` instead of new
实际上一个共享指针，比方说`std::shared_ptr`包含两个指针，一个指向数据对象，一个指向引用计数对象。因此像这样的语句

    ```c++
    std::shared_ptr<MyClass> p(new MyClass("hello",123));
    ```
    
    调用了两次内存管理，一次创建MyClass的实例，一次创建引用计数对象。
    
    不过这个问题可以使用`std::make_shared`来解决
    
    ```c++
    auto p=std::make_shared<MyClass>("hello",123);
    ```

#### Dont't Shared Ownership Unnecessarily
`std::shared_ptr`是有代价的，对引用计数的操作不是简单的增减，而是对一个完全内存屏障的原子操作。

    如果有一个shared_ptr，对其的一个使用的生存期完全覆盖了另一个使用的生存期，那么对于第二个使用的耗费就是没有必要的。比方说
    
    ```c++
    void fiddle(std::shared_ptr<Foo> f);
    ...
    shared_ptr<Foo> myFoo=make_shared<Foo>();
    ...
    fiddle(myFoo);
    ```
    
    myFoo拥有Foo的一个动态实例，调用`fiddle()`时，又出现了对这个Foo的动态实例的第二个链接，从而增加了这个shared_ptr也就是myFoo的引用计数，这里要理解下c/c++函数参数的传递方式。当`fiddle()`返回时，第二个对myFoo引用，也就是fiddle的参数释放。调用者仍然拥有这个指针。显然这里对引用计数的原子操作是没有什么意义的，单次这样的操作虽然微不足道，但是就整个程序而言，如果这样的操作多了，代价也是可观的。
    
    这里直接使用普通指针就可以了
    
    ```c++
    void fiddle(Foo* f);
    ...
    shared_ptr<Foo> myFoo=make_shared<Foo>();
    ...
    fiddle(myFoo.get());
    ```
    
    如果不愿意使用指针，也可以使用引用。用引用作为函数参数通常表示两个涵义，调用者必须确保引用在调用函数执行期间是有效的，同时那个指针必须不空
    
    ```c++
    void fiddle(Foo& f);
    ...
    auto myFoo=make_shared<Foo>();
    ...
    if(myFoo)
        fiddle(*myFoo.get());    
    ```
    
    这里`*`是把`get()`返回的指针转换成一个Foo的引用。通常引用代表着“非所有、非空的指针”。

#### Use a "Master Pointer" to Own Dynamic Variables
shared_ptr很容易使用。但正如前面所说的，共享指针代价比较高，并且有时候共享是没有必要的。很多时候，一个数据结构在其整个生存期都拥有一个动态变量，这个动态变量通过引用或者指针来使用。可以用`std::unique_ptr`来实现一个所谓的主指针，然后通过指针或者引用来使用这个主指针，只要保持一致性，那么就可以认为这些裸指针或者引用并没有拥有这个动态变量，它们只是在使用。使用主指针可以有效的避免内存泄露。如果有明确的所有者，使用主指针会是很大的优化，否则，使用shared_ptr也不是不可以。

### Reduce Reallocation of Dynamic Variables

有一些技巧可以在使用标准容器时减少分配内存的次数，这些技巧对用户自定义数据结构往往也是适用的。

#### Preallocate Dynamic Variables to Prevent Reallocation

对于std::string和std::vector，随着数据不断增多，是肯定会重新分配缓冲的。

string和vector都有一个成员函数`reserve(size_t n)`，可以用来指定它们缓冲的最少应该满足的大小，正如前面优化例子里看到过的。使用`reserver()`之后，如果预先分配的空间没有完全利用，那么可以调用它们的成员函数`shrink_to_fit()`返回这部分空余空间。

`std::unordered_map`也有`reserver()`成员函数，但是`std::deque`没有。

#### Create Dynamic Variables Outside of Loops

对于string和vector这样的会动态调整底层存储大小的数据结构来说，将其变量的创建放到循环外，在循环中只是把存储内容的大小设置为0，比方说string的clear函数，这个函数并不重新调整空间。所以对于容器来来说，区分实际分配的空间大小和所存储数据占的空间大小是有意义的。

另外，把变量的创建放到循环外，还包括作为类成员或者全局变量的方式，实际上从性能的角度说，全局变量本身是有优势的。

### Eliminate Unneeded Copying

在C++中，假定有两个变量a和b，都是一个类Foo的实例，`a=b;`会调用Foo的`operator=()`，`Foo a=b;`会调用Foo的复制构造函数，这两个函数操作基本上是一样的，都有可能涉及大量的内存分配以及内存复制工作。

在浏览热点代码时，要特别注意赋值和声明，因为这些地方都是可能涉及大量复制的。

复制可能会出现下面这些地方：

- 初始化(调用构造函数)
- 赋值(调用赋值运算符)
- 函数参数，每一个实际参数都是通过移动或者复制来构造的
- 函数返回，同样是移动或者复制构造，可能会构造两次，一次是在函数返回语句中形成返回值，一次是把这个临时的对象赋给接收返回的对象
- 向标准容器插入数据项，数据项是移动或者复制构造的，
- 或者如果插入数据造成容器重新分配大小，那么容器中的原有数据也会移动或者复制到新的空间中

#### Disable Unwanted Copying in the Class Definition

许多作为实体的对象通常拥有大量的状态，例如一个有数千个字符串的vector，或者一个有几千个符号的表，这些对象的复制，比方说作为值参数传递到一个函数中是没有必要的。

如果一个类实例的复制是代价昂贵或者是不需要的，那么可以显式的禁用掉其复制构造函数和赋值运算符，在C++11前，可以把这两个成员函数声明为private的，并且不需要定义，从C++11开始，可以使用delete关键字显式的禁用这两类成员函数，并且这个声明应该放在pulic域中。

不过，如果复制构造或者赋值传递的是指针或者引用，允许存在也是没有问题的。但是必须要注意浅复制的问题。

#### Eliminate Copying on Function Call

调用函数时，每一个实际参数表达式都会被求值，每一个形式参数都会以对应的实际参数表达式的值为初始值进行构造。

构造的意思是调用形式参数的构造函数，对于基本类型，比方说`int`，`double`，或者`char*`，构造是概念性的，并不是一个真实的函数调用，编译器只是把它们的值复制到形式参数的存储空间中。而如果形式参数是个类实例，那么这个类的某一个构造函数就会被调用。比方说

```c++
int Sum(std::list<int> v)
{
    int sum=0;
    for(auto it:v)
        sum+=*it;
    return sum;
}
```

对于这样的调用`int total=Sum(MyList);`，实际参数MyList是一个list实例，形式参数v也是一个list，于是调用std::list的复制构造函数，把MyList的每一个元素都复制到v中，每一次复制都会调用一次内存管理，显然如果MyList很大，代价将是难以接受的。并且当函数返回，v的生存期结束时，其元素的空间也要一个一个的返回空闲内存列表。换句话说，要时刻记得动态内存分配不仅仅只是分配，还包括释放。

为了避免这样的代价，这里v可以声明为`std::list<int> const& v`，这样，v就由一个引用初始化，而不是一个庞大的数据结构本身。但是对于使用指针来实现的引用，解引用也是有代价的，前面优化的例子也有说明，不过这种代价是否可以规避或者可以减少或者可以容忍，取决于程序逻辑。

使用引用或者指针作为函数参数时要注意两点：
- 除了代价较小之外，const&参数效果上可以等同于传值，前者不允许修改所引用的参数，而后者对参数的修改并不影响函数外的值。
- 非const的引用有可能产生别名，比方说一个函数签名`void func(Foo& a,Foo& b);`，如果这样调用`func(x,x)`，显然，如果`func()`改变了a，那么b也会被改动。

#### Eliminate Copying on Function Return

当函数返回值时，这个值会被复制构造成一个返回类型的临时对象。比方说

```c++
std::vector<int> scalar_product(std::vector<int> const& v,int c)
{
    std::vector<int> result;
    result.reserve(v.size());
    for(auto val:v)
        result.puch_back(val*c);
    return result;
}
```

于是，对于`auto res=scalar_product(argarray,10);`这样的调用，在函数内部return语句中，用result初始化了一个临时对象，而在函数外部，又用这个临时对象复制构造了res，如果把其中的auto去掉，表示res是已声明过的变量，那么这里又会调用一次赋值操作符。

虽然返回引用也是一个选择，但是返回一个局部对象的引用是无效的。另外，在函数中计算然后把结果返回是一个常见用例，因此大多数函数返回值而不是引用。

上面提到的的双重复制构造是早期C++程序的性能杀手，为此C++委员会和许多编译器引入了一个优化，叫做copy elision或者return value optimization(RVO)，然而，RVO适用的条件比较苛刻，函数返回的必须是一个本地构造的对象，编译器必须能够知道所有控制路径返回的都是同一个对象，这个返回对象的类型必须与函数声明的返回类型一致。不过在C++17，重新对左右值做了界定之后，大体上已经解决了这个问题。

所以，与其期待编译器做RVO，不如自己解决。正如前面优化例子曾使用过的，增加一个引用参数作为函数的输出参数承载返回，而不是依赖于函数的返回语句

```c++
void scalar_product(std::vector<int> const& v,int c,vector<int>& result)
{
    result.clear();
    result.reserve(v.size());
    for(auto val:v)
        result.push_back(val*c);
}
```

使用输出参数有几个优点

- 函数调用的时候对象已经构造好了，虽然可能在函数内会做些清理或者重新初始化的工作，但是这些与构造对象相比代价是比较小的
- 在函数内更新的对象不必再复制到return语句中的匿名临时对象上
- 函数返回值的类型可以是void，或者别的什么可以用来表示状态和错误信息的类型
- 因为被更新的对象已经由调用者绑定在一个名字上，函数返回后也就不再需要赋值或者复制了。

但是，如果输出参数是像string，vector，哈希表这样的数据结构，如果一个函数被多次调用，其动态分配的底层数组会被重用，因此需要调用者另外保存函数的结果，也就是说可能还是需要把输出参数的值取出来放到别的地方去。不过，这里的代价还是要小于函数返回一个类实例。

除此之外，这种机制没有别的额外代价了。

在C++中只有一种情况必须用值返回一个对象，那就是运算符函数。比方说矩阵运算`A=B*C`，这种句法无法通过引用返回，不过由于矩阵运算的重要性，其运算符函数经过精心设计，通常都可以使用RVO或者移动语义来达到最高的效率。

#### Copy Free Libraries

但使用缓冲或者说数组，结构，或者其它数据结构作为函数的参数时，可以传递它们的引用或者指针，这样就可以用很小的代价穿透多层次的库函数调用，这种库也被称作免复制库。比方说C++标准库里的`istream::read()`的调用序列

```c++
istream& read(char*s,streamsize);
==>size_t fread(void* ptr,size_t size,size_t nmemb,FILE* stream);
	===>ssize_t read(int fd,void* buf,size_t count);
或者
    ===>BOOL ReadFile(HANDLE hFile,void* buf,DWORD n,DWORD* bytesread,OVERLAPPED* pOverlapped);
```

缓冲区的指针会一直传递到系统调用，而各个函数会返回函数调用的状态，这点前面提到过。

这种方式显然比返回一个值的效率要高，因为不需要在各个函数中分配空间，返回的时候再复制内容。既然从操作系统到语言本身都采用了这种机制，那么认为返回一个值从而减少一个函数参数会更自然的想法自然是站不住脚的。

#### Implement the “Copy on Write” Idiom

C++11禁止了在std::string中使用COW，自然是有其道理的。COW也并不总是能带来优化效果，所以使用的时候必须小心，即使咋看起来COW是那么美好。

COW开始于一个浅复制，当数据发生变化时再进行深复制。重要的是，在实现COW的类中使用`std::make_shared()`来构造动态变量，否则，会影响内存管理对引用计数的使用。

如果总是要制作许多数据副本，或者时不时的就要改变数据，那么使用COW并没有什么特别的好处。或者说，COW更适用于不怎么改变数据的场合。

#### Slice Data Structures

分片指的是把一个数据结构划分成块，这些块可以用变量直接指向。现代语言中已经广泛使用了，在C++中暂时还是需要手工实现。

### Implement Move Semantics

移动语义解决了C++中之前一直存在的一些回绕问题，比方说

- 一个对象赋给了一个变量，其中包含了代价不菲的复制操作，然后，这个对象就被销毁了。显然这里的复制是没有必要的
- 当把一个实体，比方说auto_ptr或者资源句柄，赋给一个变量时，这个赋值里的复制操作是未定义的，因为这些实体对象的标识是唯一的。

以vector为例，上面的第一条限制使得重新分配空间时的耗费超出了必须，第二条限制则使得像auto_ptr这样的实体不能放到vector中。

#### Nonstandard Copy Semantics:A Painful Hack

当一个变量作为实体时，其复制操作是未定义的。最好是把这个变量的复制构造和赋值操作禁用掉。而vector需要其元素可以复制，所以不能对这样的对象使用vector。为此，有这么一种做法

```c++
hacky_ptr& hacky_ptr::operator=(hacky_ptr& rhs)
{
    if(*this!=rhs)
    {
        this->ptr=rhs.ptr;
        rhs.ptr=nullptr;
    }
    return *this;
}
```

虽然函数的签名已经暗示了rhs会被更改，这点与通常的赋值不同，但是赋值本身并没有任何的提示。因此如果没有仔细看过函数定义，写出下面的代码就免不了要吃一惊了。

```c++
hacky_ptr p,q;
p=new Foo;
q=p;
...
p->foo_func(); //p已经是nullptr了
```

#### `std::swap()`:The Poor Man’s Move Semantics

对于实体来说，`swap()`是良好定义的的函数模板，不同的容器类会定义相应的swap函数，用户自定义类也可以自定义`swap()`函数模板的特化。

交换的问题在于，尽管其对有动态变量需要深复制的类效率比较高，但是对于其它的类就没有那么高了，因为不能只是交换指针了。

#### Shared Ownership of Entities

实体不能复制，但是指向实体的共享指针可以。因此，尽管在移动语义出现前不能创建一个`std::vector<std::mutex>`,但是可以创建一个`std::vector<std::shared<std::mutex>>`。复制一个shared_ptr是良好定义的：制作一个唯一对象的一个额外引用。但是，这会带来额外的复杂性和运行时代价。

#### The Moving Parts of Move Semantics

为了实现移动语义，编译器必须能够辨认出临时存在的变量，这样的实例是没有名字的，比如函数返回的对象，或者new表达式返回的结果。没有其它引用指向这个对象，这个对象可以用来初始化或者赋值给一个变量，可以作为表达式或者函数调用的参数，但在下一个序列点就会被摧毁。这种未命名的值被称为右值。需要提醒下，这里的传统的说法，按现在的概念，右值指的是表达式而不是值，并且这里说的的右值现在叫prvalue。下面是一个简单的例子

```c++
class Foo
{
    	std::unique_ptr<int> value;
	public:
    	Foo(Foo&& rhs)
        {
            value=rhs.release();
        }
    	Foo(Foo const& rhs):value(nullptr)
        {
            if(rhs.value)
                value=std::make_unique<int*>(*rhs.value);
        }
};
```

当一个类没有定义复制构造函数，或者赋值运算符，或者析构函数，并且没有禁用移动操作时，编译器会自动生成移动构造函数和移动赋值操作符。在这些自动生成的移动操作里，对每个定义了移动操作的成员会进行移动操作，否则就复制。

C++11之前，即使定义了析构函数，如果没有自定义复制构造函数和赋值操作符，编译器就会自动生成，不过C++11开始就不再强制了，所以最好的是如果不需要就显式的delete。

自动生成的规则非常复杂，所以对于这些特殊的函数：缺省构造函数、复制构造函数、复制赋值操作符、移动构造函数、移动赋值运算符、以及析构函数，要么全部显式定义(包括显式禁用其中的某或者某几个)，要么全部缺省定义，要么全部禁用，总之要明确的表示出程序员的意图，而不是让编译器去猜测。

#### Update Code to Use Move Semantics

这是一个简单的确认列表，可以用来帮助这个过程

- 确定一个可以受益于移动语义的问题。比如，一个把大量的时间用在复制构造和内存管理上的类，可能会受益于移动操作
- 使用支持移动语义的编译器。
- 检查第三方库，看看其是否有支持移动语义的新版本。
- 对所要处理的类定义移动构造和移动赋值。

#### Subtleties of Move Semantics

移动语义也有一些微妙的地方，使用的时候是要注意的。

##### Moving instances int std::vector

要把对象移动到std::vector中仅仅定义移动构造函数和移动赋值操作符是不够的，必须把它们声明为noexcept。因为std::vector提供了一种强异常安全保证：如果一个vector操作的进行过程中发生了异常，vector的状态会保留在该操作进行之前。复制构造函数不会影响源对象，而移动构造函数会摧毁源对象，在移动操作过程中的异常都会违反这项保证，这里的违反就是字面意义上的违反，没有其它任何保证。

如果这些移动操作没有声明为noexcept的，那么std::vector只会使用复制操作，这里不会有任何提示，除了慢一点之外。

noexcept是一个承诺，必须能保证不抛出异常，如果其中确实抛出了异常，通常就是程序中止。这是效率的代价。

##### Rvalue reference arguments are lvalues

当一个函数用一个右值引用作为实际参数，那么会把这个右值引用用来构造对应的形式参数，而形式参数是有名字的，于是一个右值引用变成了一个左值。这是概念上要清楚的。

不过，程序员可以通过`std::move()`显式的把一个左值转换成一个右值引用

```c++
std::string MoveExample(std::string&& s)
{
    std::string tmp(std::move(s));
    return tmp;
}

std::string s1="hello";
std::string s2="everone";
std::string s3=MoveExample(s1+s2);
```

s1+s2会生成一个临时对象，或者说一个右值引用，然后用这个右值引用初始化了MoveExample的形式参数s，变成为一个左值引用。最后返回的tmp会用来构造一个作为返回值的临时对象，最终会被删除，而这个临时对象用来复制构造s3。不过，编译器这里可能会使用RVO，于是形式参数s会被直接移动到s3的存储空间中。RVO通常比移动更有效率，但是如前面所述，RVO并不总是能够实现的

下面是一个使用移动语义的`std::swap()`函数

```c++
template<typename T>
void std::swap(T& a,T& b)
{
    T tmp(std::move(a));
    a=std::move(b);
    b=std::move(tmp);
}
```

这里T必须实现移动语义，否则会回归到复制。

##### Dont’t return an rvalue reference

返回右值引用会与RVO冲突。把一个右值引用作为返回值会产生两次移动，而返回一个值，在RVO的情况下只有一次移动。这依赖于RVO，而RVO的条件比较苛刻，所以这个还是看情况而定。如果确定RVO可行，那么不论是return语句的参数，还是函数声明的返回类型，都不应该是右值引用。

##### Moving bases and members

要对一个类实现移动语义，必须对其所有的基类和成员实现移动语义。

```c++
class Base{...};
clase Derived:Base
{
    std::unique_ptr<Foo> member;
    Bar* barmember;
};
Derived::Derived(Derived&& rhs)
    :Base(std::move(rhs)),member(std::move(rhs.member)),barmember(nullptr)
{
    std::swap(this->bamember,rhs.member);
}
void Derived::operator=(Derived&& rhs)
{
    Base::operator=(std::move(rhs));
    delete(this->barmember);
    this->barmember=rhs.barmember;
    rhs.barmember=nullptr;
}
```

这里假定Base有一个移动构造函数。要注意的是Base移动构造函数的右值引用参数是通过`std::move(rhs)`初始化的，但没有把rhs移动到一个临时的地方，只是生成一个指向rhs的引用，并没有发生所有权的转移，一个左值引用的所有权不可能转移到一个临时对象上去，因此，后面对`rhs.member`的引用还是有效的，但是要注意这时候rhs的状态属于有效但是未确定，对其的操作是不能有前置条件的。移动赋值操作符里边对Base的`operator=`的调用同理。而在移动赋值操作符里边没有使用swap的原因在于，`this`指针可能已经指向了一个分配了空间的对象，交换的时候，`*this`会被交换给rhs，而swap并不会摧毁rhs，除非rhs自己析构，从语义上来说，rhs不会再被使用，但是又没有被销毁，于是浪费了空间。

### Flatten Data Structures

如果一个数据结构的成员可以一起存放在一片连续的空间，那么就可称之为平面的。平面数据结构相比链接型数据结构有很多的优势：

- 内存管理代价更小，因为需要创建的动态对象更少，即使是同样O算法性能，`std::vector`和`std::array`的实际效率会更高。
- 所占内存空间更小。有利于缓存的效率。
- 配合移动语义，可以减少分配智能指针和分派指针指向的运行时代价。

### Summary

- Naive use of dynamically allocated variable is the greatest performance killer in C++ programs.When performance matters,**new** is not you friend.
- A developer can be an effective optimizer knowing nothing other than how to reduce calls into the memory manager.
- A program can globally change how memory is allocated by providing a difination of `::operator new()` and `::operator delete()`.
- A program can globally change hwo memory is managed by replacing `malloc()` and `free()`.
- Smart pointers automate ownership of dynamic variable.
- Shared ownership of dynamic variables is more expensive.
- Create class instances statically.
- Create class members statically and use two-part initialization if needed
- Use a master pointer to own dynamic variables,and unowned pointers instead of sharing ownership.
- Create copy-free functions that pass data out in out parameters.
- Implement move semantics.
- Prefer flat data structures.





