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



