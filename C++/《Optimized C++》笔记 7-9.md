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

每次函数调用大体上会发生这些事情

- 执行代码向调用栈推入一个帧，这个帧里用来存储函数的调用参数和局部变量
- 每一个参数表达式求值然后将结果复制到上述栈帧里
- 当前执行地址被复制到栈帧以形成返回地址
- 执行代码将执行地址改变为函数体的第一条语句的地址
- 执行函数体的指令
- 返回地址从栈帧复制到指令地址，把控制权转移到函数调用语句的下一条语句
- 栈帧从栈中弹出

这只是对函数调用流程一个形象化的说明，并不代表每次调用的细节，并且要注意的是推入栈的栈帧包含的是数据和地址，而不是指令，冯·诺依曼体系的架构下指令和数据当然是分别存储的。另外inline函数可以就地展开而不是进行调用。

#### Basic cost of function calls

- 函数参数

    除了对参数表达式求值的代价外，把每一个参数值复制到栈也是有代价的。有时候有些参数可以复制到寄存器中，但是如果参数多的话还是需要通过栈传递的。但是由于参数处理的顺序可能会依平台而异，宽泛的说把较小的参数放在左边或者右边都是不合适的。

- 成员函数调用（与普通函数调用相比）

    每一个成员函数调用都有一个隐藏的参数，一个指向`this`的指针，或者说`*this`，或者说this指针，具体怎么说依情况而定。

- 调用和返回

    调用需要把执行地址写入到栈帧中以形成返回地址，返回则需要把返回地址读出来并放入到执行指针中。在调用和返回的时候，执行是在一个非连续的内存地址上，这会造成流水线堵塞以及缓存未命中。

    这两者都是程序执行时的纯负担，所以尽可能把函数的inline化。

#### Cost of virtual functions

每一个有虚函数的类实例都有一个指针，这个指针指向vtable，或者说虚函数表，这个指针通常通常存在于类实例的最前面，以减少访问代价。于是虚函数地址的查找分成两步，一个是找到虚表，一个是在虚表中找到虚函数，这样就增加了两次装载不连续内存的操作，从而增加了缓存未命中和流水线堵塞的概率。

另外，通常来说，由于动态性，虚函数是很难inline化的。

#### Member function calls in derived class

派生类的成员函数调用需要一些额外的工作

- 派生类中定义虚成员函数

    如果继承树上的基类都没有虚函数，那么定义了虚函数的派生类实例就需要增加一个虚表指针，并且这个虚表指针在这个类实例的位置会有个偏移，而不是在最前面，这会增加访问虚表的代价。

- 多重继承的派生类中定义的成员函数

    为了能够访问多个基类中的函数，同样也需要加上偏移，也会增加代价

- 多重继承的派生类中定义的虚函数

    对于一个多重继承的类，如果其某个不是最底层的基类定义了虚函数，那么为了寻找虚函数，那么就首先需要找到这个基类，然后找到这个基类的虚表，这里可能需要计算两次偏移量

- 虚多重继承

    为了找到被虚继承的基类，需要计算偏移量，如果要调用虚函数，那么同样也还要计算虚表的偏移

#### Cost of pointers to functions

对于函数指针，在函数调用和返回的基本代价之外，还有些额外的代价

- 函数指针（普通函数和静态成员函数）

    需要解引用，并且也不太可能inline化

- 成员函数指针

    成员函数指针需要足够通用以能够调用任意的成员函数。对于成员函数指针，做任何最糟糕的评估都不是没有理由的。

#### Summary of function call costs

不带参数的void的C风格的函数是代价最小的，如果可以inline化则没有任何代价，否则也只有两次内存访问加上两次非本地的执行跳转。

如果一个虚函数属于一个基类没有定义虚函数的派生类，而这个派生类又被虚继承，这是一种最糟糕但不太常见的情况，首先需要查找一个表，找到这个虚基类，然后再查找到这个虚基类的虚表，然后在虚表中查找到虚函数的地址。

到这里，坏消息是移除一次非序列内存访问并不能对优化产生什么太多的效果，除非这个函数会被多次调用，这里非序列内存访问指的是对内存的连续访问操作其内存地址不连续。好消息是profiler能够直接的支持最常访问的函数。

### Declare Brief Functions Inline

要inline化一个函数，编译器必须在函数的调用点能够访问到函数的定义。在类定义内部定义的函数隐含的声明为inline，而在外部定义的可以加上inline关键字。虽然标准只是说inline是个提示，但编译器总是尽可能的inline化一个函数的。实际上，一个程序的debug版本和Release版本的区别主要就在于debug版本关掉了inline。

### Define Functions Before First Use

在函数的调用点之前完整地定义函数可以让编译器有机会优化函数调用，这对于虚函数也是有效的，只要编译器在虚函数被调用时能够看到完整的函数体和实例化类变量、指针或者引用的代码。

### Eliminate Unused Polymorphism

如果必须在运行时对不同的实现做出选择，虚函数表是一种非常快捷的机制，尽管会需要两次额外的内存装载和潜在的流水线堵塞以及缓存未命中。如果实际上并不需要这种动态的多态机制，那么取消掉函数声明的virtual限定符也是可以考虑的，比方说在一个层级结构中，各层的类都以指针联系起来了，这个时候是不需要虚函数的。

### Discard Unused Interfaces

接口的好处之一是派生类必须实现接口中声明的方法，否则编译器就会报错。但是如果一个应用只提供了一个单一的实现，那么这时候可以考虑把接口变成一个普通的基类类，也就是把成员函数声明前的virtual去掉，并提供哪怕一个空实现，然后在单一的派生类的实现中实现所需的操作。将来需要的话，再加上virtual也是可以的，这时候并不需要改动原有的派生类实现。

#### Select interface implementation at link time

使用虚函数来实现接口的问题在于，对于一个设计期的问题，虚函数提供了一个运行时解决方案，这个方案带来了运行时代价。

有一些使用同一个接口的应用实际上是需要共存的，比方说一个对应文件系统的接口，Windows和linux的实现并不需要同时存在。这时候虚函数的代价是不必要的。这时候可以通过在链接的时候指定合适的翻译单元来完成构建。

比方说一个头文件file.h，里边声明了File类，但没有实现

```c++
class File
{
	...
};
```

对应这个头文件，提供两个各自对应两个平台的实现，比方说windowsfile.cpp和linux.cpp，在构建的时候就可以通过编译参数来决定链接哪个翻译单元。

这样的好处在于非常的通用化，不足之处在于需要.cpp文件和makefile来决定选择。

#### Select interface implementation at compile time

前面说明了由链接器来做选择的方法，实际上在实现文件中通过#ifdef可以实现在编译时选择，也就是说，在文件file.cpp中

```c++
#include "file.h"
#ifdef _WIN32
...
#else  //linux
...
#endif
```

这里的好处在于在一个文件中就可以看到所有的选择，但是不好的地方在于其繁杂和不是那么的面向对象，因为把所有的东西都放在了一个文件里。

以上两种方法的选择依各人所好。前一种方法在makefile中就可以指定需要编译和链接的翻译单元，并不存在多编译一个文件的问题。而第二种方法确实过于繁杂，个人不喜欢这种方式。

### Select Implementation at Compile Time with Templates

作为模板参数的类型，只有其被使用到的成员才需要实现，只要其成员没有被使用，编译器就不会对模板参数缺少定义报警，而程序员也可以自行选择实现哪些成员。

### Eliminate Uses of the PIMPL Idiom

PIMPL指的是“Pointer to IMPLementation”，可以用来避免一个头文件的改变触发多个源文件的重新编译。但是在运行时，这种方法只会带了延迟，一则可能本来可以inline化的函数需要一次函数调用，二则调用成员函数可能需要调用另一个类的成员函数，三则对于debug来说由于增加了函数调用层次而变得更加乏味。

当然，从设计模式的角度说，组合一个指向其它类的指针是许多设计模式的实现方式，但这不是出于减少重编译的初衷，而是出于解决问题的目的。

### Eliminate Calls int DLLS

对于动态库的调用，都需要使用一个指针，不论是Windows还是Linux。有时候调用动态库是必要的，比方说一个应用可能会使用第三方的库，或者一个库并不想暴露自己的源码。

但是对于并不需要向外提供库的应用来说，没有必要给自己找麻烦使用动态库。如果可能的话，使用静态链接而不是动态链接，这也是提高函数调用性能的方法之一。

### Use Static Member Functions Instead of Member Functions

对成员函数的调用会隐含的传入一个参数，通过这个this指针，可以访问类实例的成员以及虚表等等。但是如果一个成员函数并不需要访问类的其它成员，这个this指针就没有必要，这时候这个成员函数就可以定义成static的，静态成员函数不需要对隐含的this指针求值，本身也可以通过普通的函数指针引用，而不是代价高昂的成员函数指针。

### Move Virtual Destructor to Base Class

拥有子类的类的析构函数应该声明成virtual的，这样在析构一个对象时就可以找到正确的对象。另外一个原因是，在基类中声明虚函数，可以确保虚表指针位于一个类实例的最前面，以减少计算偏移的代价。

## Optimize Expressions

在语句层面之下是表达式，这是优化的最后机会。通常来说编译器已经很擅长与优化表达式了，但是只有在编译器能够证明不会改变语义的情况下才会去进行表达式的优化。所以在这方面程序员要比编译器能做的更多，但是这并不表明在现代计算机架构下这样的优化能有多大的效果。

### Simplify Expressions

写出最精简的表达式是程序员的责任，编译器只会按照内置的顺序对表达式求值。比如多项式求值$y=ax^3 + bx^2 +cx + d$，在C++中会写成`y=a*x*x*x+b*x*x+c*x+d`，然而这并不是效率最高的写法，效率更高的写法是`y=(((a*x+b)*x)_c)*x+d`，而编译器并不会把前一种写法自动的优化成后一种写法。编译器不自行改变算术表达式的顺序的原因是那样会比较危险，比方说对这样一个表达式`a/b*c`，由于三个操作数都会int型，那么很有可能计算结果是0，然而如果改变成`a*c/b`，又有溢出的可能，所以，对这样算术表达式的结果的预测，是程序员的责任。

### Group Constants Together

编译器可以做的一件事是计算常量表达式的值，跟通常想的不一样，或者说没有注意过的，对于一个表达式`seconds=24*60*60*days`或者`seconds=days*(24*60*60)`，编译时可以对这个表达式中的常量部分求值，也就是变成`seconds=86400*days`。然而，如果是`seconds=24*days*60*60`，那么这里的乘法计算就会在运行时进行了。

换句话说就是，把表达式中的常量部分的计算用括号括起来，或者把它们放到表达式的最左边，或者更好的是，把它们从表达式中剥离，用来初始化一个const的变量，或者放到一个constexpr的函数中。

### Use Less-Expensive Operator

乘法和除法的代价是比较高的，所以有时候可以把乘法和除法分别用左移和右移来代替，但是这需要程序员自己去把握，比方说`x*9`可以写成`(x<<3)+x`。

### Use Integer Arithmetic Instead of Floating Arithmetic

比方说要对除法的商四舍五入，通常会写成这样`(unsigned)round((double)n/(double)d)`，显然这样的耗费是比较大的。更好的方法是

```c++
inline unsigned div(unsigned n,unsigned d)
{
    unsigned q=n/d;
    unsigned r=n%d;
    return r>=(d>>1)?q+1:q;
}
```

以及

```c++
inline unsigned div(unsigned,unsigned d)
{
    return (n+(d>>1))/d;
}
```

### Double May Be Faster than Float

这只是一种可能，但是除非某些特别原因，能使用float的地方都应该以double替代，除了在某些架构比如X86机器上double会比float快之外，求值的时候编译器实际上通常也会自动把float转换成double以保证精度。

### Replace Iterative Computations with Closed Forms

很多时候需要针对位进行操作，这些操作可能需要$O(n)$的迭代，但是有时候可以写出更为紧凑的算法。比方说我们要判断一个整数是否2的幂，因为2的幂是只有一位为1的数，所以可以进行一个计数

```c++
inline bool is_power_2(unsigned n)
{
    for(unsigned one_bits=0;n!=0;n>>=1)
    {
        if((n&1)==1)
        {
            if(one_bits!=0)
                return false
            else
                one_bits+=1;
        }
    }
    return true;
}
```

当然上面有很多可以改进的地方，但是不管怎么改也是一个$O(n)$的算法。

然而，我们知道，在只考虑无符号数的情况下，2的幂必然是只有最高位为1，其余为0，所以有更为紧凑的算法

```c++
inline bool is_power_2(unsigned n)
{
    return (n!=0) && !(n&(n-1));
}
```

## Optimize Control Flow Idioms

计算要比控制快，因为跳转会照成流水线堵塞，所以编译器会尽可能的减少为此对指令指针的更改。

为了写出更快的代码需要了解编译器所做的事情。

### Use Switch Instead of if-elseif-else

平均来说，if-elseif-else语句会做$O(n)$的b比较，当然，对于分支较少的情况，把可能性最大的比较放在最前面是有效的。

而对于switch语句，编译器可以对switch需要比较的值做个排序，形成一个跳转表，当需要比较的值是连续的时候，switch语句只是做一个下标操作然后跳转，比较的次数是$O(1)$，当比较的值有空洞时，最糟糕的情况下也可以在$O(log_2n)$的时间找到分支。

### Use Virtual Functions Instead of `switch` or `if`

比方说这样的代码

```c++
Animal::move()
{
    if(this->animalType==TIGER)
    {
        pronce();
    }
    else if(this->animalType==RABBIT)
    {
        hop();
    }
    else if(...)
    ...
}
```

对于现代C++而言，这里的move可以使用虚函数，在虚函数表中查找函数的代价是个常量。

当然，从设计模式的角度说，创建对象还是免不了要判断类型参数，区别只在于是运行期还是编译期做这个类型判断。不过这已经是两个层面的事了。

### Use No-Cost Exception Handling

是否使用异常要依团队整体情况而定。

对于C++引入的noexcept，除了这是对之前异常机制的扬弃之外，还有一个重要的原因，如前面说过的，为了实现移动语义，编译器需要某些移动构造函数和移动赋值函数声明为noexcept，以表明对某些对象而言，这些函数的移动语义比强异常安全性更重要。

## Summary

- Optimizing at the statement level doesn’t provide enough improvement to make it worthwhile unless there are factors that magnify the cost of the statement
- The cost of statements in loops is magnified by the number of iterations of the loops
- The cost of statements in functions is magnified bye the number of times the functions are called
- The cost of a frequently used idiom is magnified by the number of times the idiom is used
- Some C++ statements (assignment,initialization,function argument evaluation) contain hidden function calls
- Function calls into the operating system are expensive
- An effective way to remove function all overhead is to inline the function
- There is currently little need for the PIMPL idiom.Compile times today are maybe 1% of what they were when PIMPL was invented
- `double` arithmetic may be faster than `float` arithmetic

# Chapter 8 Use Better Libraries

库是优化的焦点区域。通常来说库函数和类位于执行的最低端，所以会很热。

## Optimize Standard Library Use

C++为下面这些通用的用途提供了一个紧凑的标准库

- 决定应用依赖的行为，比方说每个数值类型的最大值和最小值
- 一些函数用C++来实现可能不是最好的，比方说`strcpy()`和`memmove()`
- 用起来很简单但是写起来很麻烦，并且需要考虑跨平台可移植性的一些数学函数，比方说sin、cos、log等等
- 除了内存分配之外，不依赖于操作系统的可移植的通用的数据结构，比如字符串，列表，和表。
- 用来查找、排序、变换数据的可移植的通用的算法
- 以一种独立于操作系统的方式来获取操作系统基本服务的函数，比方说内存分配，线程，时钟和流I/O等。这里包括为了保持兼容性从C语言继承来的一个函数库。

大多数C++标准库都包括模板类和模板函数，以提供高效的代码。

### Philosophy of the C++ Standard Library

C++为了保持其系统开发语言的特性，提供的标准库必须是简单，通用以及快速。从哲学上说，进入C++标准库的函数和类，要么是不能通过其它途径提供的，要么就是可以跨大多数操作系统使用的。

### Issues in Use of the C++ Standard Library

下面说的这些也适用于标准linux库，POSIX库，以及其它一些广泛使用的跨平台库。

- 标准库实现是有bug的
- 标准库实现不一定遵从C++标准
- 对标准库开发者而言性能不是最重要的，简单性、可维护性和可移植性才是
- 库实现可能会阻碍优化尝试
- C++标准库中不是所有部分都是同等有用的
- 标准库函数不具有最好的本地函数的效率。本地函数指的是操作系统或者专门库提供的函数。

## Optimize Existing Libraries

优化现有库就像扫雷，也有可能，也许有必要，但是很困难。下面是一些建议。

### Change as Little as Possible

换句话说就是一次不要改太多。一次解决一个问题，看看效果，然后再考虑下一个问题。

### Add Functions Rather than Change Functionality

比方说把循环放到库里边，增加对移动语义的支持。只是要注意添加的函数是否会与原有库函数同名。

## Design Optimized Libraries

### Code in Haste,Repent at Leisure

一个可交付库的核心是接口的稳定性。包括一致的调用和返回方式，一致的分配方式，一致的效率。

在开发像库这样关键的代码时，老式的学院派开发方法，包括前期的规格说明、设计、文档以及模块测试，都是有用的。

### Parsimony Is a Virtue in Library Design

意思就是一个库要专注于一个特定的任务，并且为此尽可能俭省。

比方说，一个读取文件的函数，接受一个有效的`std::istream`参数要比接受一个文件名然后在函数中打开它更俭省，处理操作系统依赖的文件名语言和处理I/O错误不是一个数据处理库的核心。

再比方说，接受一个指向内存缓冲的指针比分配空间并返回俭省，这也意味着库不需要处理内存不足的异常。

俭省是一致性使用良好的C++编程原则的终极结果，包括单一责任原则和接口分离原则。

### Make Memory Allocation Decisions Outside the Library

这也是俭省规则的一个实例。

在库函数外分配内存，一则可以让调用者重用而不是重新分配空间，二则可以减少内存复制的次数。

如果可能的话，在派生类中分配空间，基类只保持一个指针，这样派生类就可以自己决定怎样分配空间。

在设计库的时候就应该做出这种决定，因为这涉及到函数签名的改动，而函数一旦被使用了，再改就会造成麻烦。

### When in Doubt,Code Libraries for Speed

反正你也不知道你的库将来会用到哪里，所以从性能开始考虑吧。

### Functions Are Easier to Optimize than Frameworks

也就是说尽可能开发函数库，而不是框架。

### Flatten Inheritance Hierarchies

大多数情况下，就类继承而言，不需要超过三层抽象：一个定义公共函数的基类，一个或多个实现多态的派生类，或许还有一个使用多重继承的混入层，以对应复杂的现实。通常来说，一个过深的继承层次意味着考虑不周，引入了不必要的复杂性以及低性能。

比方说构造或者析构一个对象，需要遍历其多个基类，不管怎么说，这都是耗费。

### Flatten Calling Chains

同样，对于函数调用来说，大多数情况也不需要超过三层的嵌套：一个自由或者成员函数实现策略，需要调用某个类的成员函数，在其中调用类的公共或者私有的成员函数来实现某种抽象或者访问数据。

### Flatten Layered Designs

有时候一个抽象需要通过另外一个抽象实现，这就引入了设计层次。这可能会影响性能。

不过，有时候层次是必要的

- 比方说facade模式
- 封装一个现有的库
- 在返回错误码的函数与抛出异常的函数之间做个耦合
- 实现PIMPL
- 调用DLL或者插件

层次之间的耦合必然需要代价，下面是一些提示

- 一个项目中多个facade模式的实例可能意味着过度设计
- 层次太多的一个标志是一个给定的层多次出现，比方说一个返回错误码层调用一个异常处理层，而这个异常处理层又调用一个返回错误码层
- 嵌套的PIMPL实例会使得难以判定PIMPL的最根本的目的是避免重编译。大多数子系统根本就没有庞大的需要嵌套的PIMPL。
- 项目特定的DLL通常是为了封装对bug的解决，很少有项目会意识到这一点，于是bug的解决采用了批量发布，从而跨越了DLL边界。

减少层次应该在设计阶段解决。

### Avoid Dynamic Lookup

大程序包含很大的配置文件或者很长的注册表条目，复杂的数据文件比方说音频或者视频流文件通常包含元数据。数据不多的时候可以放到一个struct或者一个类中，数据很多的时候，设计者就会倾向于使用一个查找表。然而，动态搜索一个符号表是个性能杀手，因为

- 动态查找天生就是低效的。在json或者xml文件中查找项的效率与文件大小是线性关系，而基于表的查找项则是$O(log_2n)$，从一个struct中获取一个数据项则是$O(1)$。
- 库设计者不会知道元数据的所有访问方式。如果只是在启动时获取一次，代价当然是微不足道的。但是有些元数据是会不停地的读取的，所以库提供的查找元数据的方法不会比在一个key/value表中查找快。
- 如果已经有了一个基于表的查找，下一个问题就是一致性。为了保持表的一致性所做的检查，也是一个性能负担。
- 一个基于结构的仓库是自说明的，而一个符号表是许多值的集合，其意义需要查阅文档。

### Beware of ‘God Functions’

所谓”god function”指的是在高层次实现策略的函数，使得链接器向可执行文件中添加了许多库函数，增加了可执行文件的大小，在嵌入式系统中会耗尽物理内存，在桌面应用中会增加虚拟内存的分页。

实际上，在框架中这种函数是不可避免的。

另外，printf很多时候也是一个god function。

## summary

- Functions and classes enter the C++ standard library either because they cannot be provied any other way,or because they promote very wide reuse across multiple operaing systems.
- Standard library implementations have bugs.
- There is no such things as a “standard-conforming implementation”.
- The standard library is not as efficient as the best native functions.
- When updating a library,change as little as possible.
- Stability of interface is a core deliverable of a library.
- Test cases are critical for optimizing a library.
- Designing a library is like designing other C++ code,only the stakes are higher.
- Most abstractions require no more than three layers of class derivation.
- The implementation of most abstractions require no more than three nested function calls.

# Chapter 9 Optimize Searching and Sorting

以查找作为例子说明优化的普遍过程：把现有的解决方案分解成算法和数据结构，然后分别考虑它们的优化可能。

## Key/Value Tables Using std::map and std::string

假设有这样一个表

```c++
std::map<std::string,unsigned> const table{
    {"alpha",1},{"bravo",2},
    {"charlie",3},{"delta",4},
    {"echo",5},{"foxtrot",6},
    {"golf",7},{"hotel",8},
    {"india",9},{"juliet",10},
    {"kilo",11},{"lima",12},
    {"mike",13},{"november",14},
    {"oscar",15},{"papa",16},
    {"quebec",17},{"romeo",18},
    {"sierra",19},{"tango",20},
    {"uniform",21},{"victor",22},
    {"whiskey",23},{"x-ray",24},
    {"yankee",25},{"zulu",26}
};
```

## Toolkit to Improve Search Performance

```c++
void HotFunction(std::string const& key)
{
    ...
    auto it=table.find(key);
    if(it==table.end())
    {
        ...
    }
    else
    {
        ...
    }
}
```

如果profiler发现上面这样一段代码位于热点当中，可以怎么优化呢？从方法论的角度说

- 测量现有实现的性能，作为优化的基线
- 鉴别可以优化的抽象活动
- 把需要优化的活动分解为算法和数据结构
- 改变或者替代算法和数据结构，试验以确定改变是否有效

如果把需要优化的活动看成一个抽象，那么优化任务就是把立足于应用的基础抽象，然后不断的建立性能更好的更特殊的抽象。这个过程是迭代的，可以把它记下来，会有助于得到更多的发现。

### Make a Baseline Measurement

当然，需要确定一个全覆盖的测试案例，作为比较的基准。

### Identify the Activity to Be Optimized

什么是需要优化的活动，这是由程序员来判断的，不过也有一些线索。就本例而言，基准实现是一个用string做关键字的map，从热点分析可以找到find函数的调用。然后表在哪里构造和析构也是可以看看的，因为这些活动是需要耗费时间的。

于是，这里需要优化的活动很明显，在一个用string作为关键的map中，寻找指定关键字的值。以此为基础，可以构建出一个基准抽象：在一个用文本作为关键字的键/值表中查找一个值。

### Decompose the Activity to Be Optimized

把上面提到的基准抽象分解为算法和数据结构，得到

- 表，保存键和值的数据结构
- 键，一个文本型数据结构
- 比较键的算法
- 在表中进行搜索的算法
- 构造表或者向表中插入键的算法

上面这些条目是怎么来的呢？

1. 在基准方案中，表是`std::map`
2. 在基准方案中，键是`std::string`实例
3. 在`std::map`的模板定义中，有一个缺省参数用来指定键比较函数
4. 热点函数包含一个对`std::map::find()`的调用，而不是使用`operator[]`
5. 一个map必然需要构造和析构，map是通过红黑树实现的，因而是一个链接型数据结构，一定会有一个插入算法。

对于最后一项，构建和销毁一个对象的算法经常会被忽略，然而其耗费可能是很显著的。所以需要注意一下，以确认对象的创建和销毁是否有影响。

### Change or Replace Algorithms and Data Structures

可以从需要优化的活动得到一些启发

- 表这个数据结构可以更换或者采用更高效的实现。对表数据结构的选择限定了对查找和插入算法的选择，不同的表实现也会有不同的效率，比方说需要动态分配的实现会调用内存管理
- 键的数据结构也可以更换或者采用更高效的实现
- 比较算法也是
- 查找算法也是，不过查找算法的选择受限于表数据结构的选择
- 插入算法，以及在哪里何时创建和销毁数据结构，也是可以刚改或者采用更高效的实现的

于是，就有这样的考虑

- std::map是使用红黑树实现的，其查找算法时间是$O(log_2n)$的，所以，这个数据结构是有可替换的余地的
- std::map是个基于节点的数据结构，会频繁调用内存管理，在本例中，数据只需要插入而不需要删除，所以采用一个较少调用内存管理的数据结构也是合理的
- 对于键所使用的的数据结构，只需要保存这个键，以及对两个键作比较，使用std::string是过需求了。
- std:;string实例被作为值来使用，但string本身定义了比较方法，而map作为一个键值对的数据结构也提供了比较操作符，从这个角度来说，只要提供一个比较函数作为map的模板参数，那么可以选用更紧凑的数据结构来保存key
- 前面对map初始化所使用的的C++风格的列表初始化并非一个C风格的静态聚合初始化，还是需要调用内存分配创建string实例，然后把这些实例插入到map中，插入map也需要再次调用内存分配创建红黑树节点，然后把string实例复制到这些节点上。这种方法的好处在于形成的map是静态的。也许可以找到一个真正实现了C风格聚合初始化的数据结构。

### Using the Optimization Process on Custom Abstractions

上面的过程也可以适用于非标准库的抽象，不过可能需要更多的工作，毕竟`std::map`和`std::string`是良好定义并且文档齐全的。对于垃圾堆一样的代码，一方面优化会很麻烦，另一方面，越是混乱的代码，就越有优化的余地。

## Optimize Search Using std::map

### Use Fixed-Size Character Array Keys with std::map

当使用string作为key时，对于`val=table[“zulu”];`这样的语句，每一次查找都需要把一个`const char*`转换成`std::string`，于是需要动态分配一块内存，然后马上又释放掉。

如果定义这样的数据结构来存储键

```c++
template<unsigned N=10,typename T=char>
struct charbuf
{
    charbuf();
    charbuf(charbuf const& cb);
    charbuf(T const* p);
    charbuf& operator=(charbuf const& rhs);
    charbuf& operator=(T const* rhs);
    
    bool operator==(charbuf const& that) const;
    bool operator<(charbuf const& that) cosnt;
    
  private:
  	T data[N];
};
```

程序员必须考虑的是，这里操作处理定长的字符数组，换句话说就是适用性不好。

### Use C-Style String Keys with std::map

可以使用C类型的字符串作为map的键，唯一的问题就是在map冲存放的是一个指针，所以需要提供一个比较函数。

```c++
bool less_free(char const* p1,char const* p2)
{
    return strcmp(p1,p2)<0;
}

std::map<char const*,unsigned,bool(*)(char const*,char const*)> table(less_free);
```

当然，也可以这样

```c++
struct less_for_c_string
{
    bool operator()(char const* p1,char const* p2)
    {
        return strcmp(p1,p2)<0;
    }
};

std::map<char const*,unsigned,less_for_c_string> table;
```

或者使用lambda表达式

```c++
autot comp=[](char const* p1,char const* p2)
{
    return strcmp(p1,p2)<0;
}

std::map<char const*,unsigned,decltype(comp)> table(comp);
```

作者测量的结果是后面这两种方法的性能差不多，但是都比使用自由比较函数性能要高。这个需要自己实测，不作定论。

### Using Map’s Cousin std::set When the key is the Value

实际上，map内部会把键和值组合成一个pair，当然这是个struct，定义大概如此：

```c++
template <typename KeyType,typename ValueType> struct value_type
{
    KeyType const first;
    ValueType const second;
    ...
};
```

但是如果一个程序自定义了这样的一个结构，并不能直接用到map中，map的键和值必须分别定义。

如果一定要使用这样的结构，那么可以使用`std::set`，也就是说把这个结构整体作为set的元素放入set中，于是就需要对这个结构指定比较算法，不论是`std::less`，还是定义`operator<`，或者提供一个比较对象，都可以，只是个人风格或者依实测性能而定。

## Optimize Search Using the `<algorithm>` Header

标准库的算法大都是基于迭代器而不是具体的数据结构的。

### Key/Value Table for Search in Sequence Containers

使用线性容器而不是map或者set来实现键/值表是有很多理由的。一个从局部性原理的角度，另一个是算法复杂度的角度。所以前面的例子可以采用这样的数据结构

```c++
struct kv
{
    char const* key;
    unsigned value;
};

kv names[]={
    {"alpha",1},{"bravo",2},
    {"charlie",3},{"delta",4},
    {"echo",5},{"foxtrot",6},
    {"golf",7},{"hotel",8},
    {"india",9},{"juliet",10},
    {"kilo",11},{"lima",12},
    {"mike",13},{"november",14},
    {"oscar",15},{"papa",16},
    {"quebec",17},{"romeo",18},
    {"sierra",19},{"tango",20},
    {"uniform",21},{"victor",22},
    {"whiskey",23},{"x-ray",24},
    {"yankee",25},{"zulu",26}
};
```

对names数组的初始化时C风格的聚合初始化，可以在编译时进行，运行期代价是0。

还可以定义一些查找需要的函数

```c++
template<typename T,int N>
size_t size(T (&a)[N])
{
    return N;
}
template<typename T,int N>
T* begin(T (&a)[N])
{
    return &a[0];
}
template<typename T,int N>
T* end(T (&a)[N])
{
    return &a[N];
}
```

C++标准库在`<iterator>`中有类似的定义，直接拿来用也是可以的。

### `std::find()`: Obvious Name, $O(n)$ Time Cost

`<algorithm>`中find的定义是这样的

```c++
template<class It,class T>It find(It first,It last,const T& key)
{
    while (first!=last) 
    {
    	if (*first==key) return first;
    	++first;
  	}
  	return last;
}
```

find采用的是线性搜索，对数据存储方式没有要求，只需要数据可以做`==`比较，提供一个`==`的重载函数后，就可以使用find了

```c++
bool operator==(kv const& n1,char const* key)
{
    return strcmp(n1.key,key)==0;
}
kv * result=std::find(std::begin(names),std::end(names),key);
```

find还有一个变种find_if，其第四个参数是一个比较函数，这个参数可以使用lambda表达式，具体的可查阅文档。

### `std::binary_search()`: Does Not Return Values

names里的数据键字段是字母有序的，可以使用二分查找，但是这个函数只返回一个bool值，并不返回键对应的值，在这里并不适合。

### Binary Search Using `std::equal_range()`

`<algorithm>`中定义了函数`std::equal_range()`，因为使用了二分查找，所以其算法复杂度理论上是$O(log_2n)$

```c++
template <class ForwardIterator, class T>
  pair<ForwardIterator,ForwardIterator>
    equal_range (ForwardIterator first, ForwardIterator last, const T& val)
{
  ForwardIterator it = std::lower_bound (first,last,val);
  return std::make_pair ( it, std::upper_bound(it,last,val) );
}
```

在返回的pair中如果两个值相等，则表明结果集为空，于是

```c++
auto res=std::equal_range(std::begin(names),std::end(names),key);
kv* result=(res.first==res.second)?	std::end(names)	: res.first;
```

但是效果如何需要实测，原作者测试的效果还不如find。在上面的定义中，可以看到equal_range调用了lower_bound和upper_bound，实际上是进行了两次二分查找，所以效率在某些特定的数据集上不如线性查找也是可以理解的。

### Binary Search Using `std::lower_bound()`

就本问题而言，对upper_bound的调用是没有意义的，因为对指定的键,表中要么存在一个值，要么不存在，所以

```c++
kv* result=std::lower_bound(std::begin(names),std::end(names),key);
if(result!=std::end(names) && key<*result.key)
    result=std::end(names);
```

这个方法的时间复杂度，实际上与binary_search是一样的，除了binary_search没有返回值外，与使用map的最好的算法时间也是一样的，而且构造和销毁表的代价是0。

### Handcoded Binary Search

看看手工写的使用`operator<`的二分查找

```c++
kv* find_binary_lessthen(kv* start,kv* end,char const* key)
{
    kv* stop=end;
    while(start<stop)
    {
        auto mid=start+(stop-start)/2;
        if(*mid<key)
        {
            start=mid+1;
        }
        else
        {
            stop=mid;
        }
    }
    return (start==end||key<*start)?end:start;
}
```

这个方法的时间与low_bound或者binary_search没有本质差别。

### Handcoded Binary Search using `strcmp()`

最后看看使用strcmp实现的二分查找

```c++
kv* find_binary_strcmp(kv* start,kv* end,char const* key)
{
    auto stop=end;
    while(start<stop)
    {
        auto mid=start+(stop-start)/2;
        auto rc=strcmp(mid->key,key);
        if(rc<0)
        {
            start=mid+1;
        }
        else if(rc>0)
        {
            stop=mid;
        }
        else
        {
            return mid;
        }
    }
    return end;
}
```

这里多了一个分支，也就是在找到结果的时候就可以返回了，而不是像之前那样要直到最后才返回结果。所以时间上要比之前的算法好一些。

## Optimize Search in Hashed Key/Value Tables

哈希表的两个核心问题，一个是哈希函数，一个是冲突解决。伴随的问题就是冗余空间，大多数情况下这是不可避免的。C++定义了一个标准哈希函数`std::hash`。

### Hashing with a `std::unordered_map`

标准头文件`<unordered_map>`提供了一个哈希表。`std::unordered_map`不能像之前那样静态生成，必须逐个插入

```c++
std::unordered_map<std::string,unsigned> table;
for(auto it=names;it!=names+namessize;++it)
    table[it->key]=it->value;
...
auto result=table.find(key);
```

`std::unordered_map`使用的缺省哈希函数就是`std::hash`。但是这里实测的效果虽然比使用map要好，但好的有限。

### Hashing with Fixed Character Array Keys

使用前面提到过的charbuf作为键时，可能是标准哈希函数效率不佳，整体时间不好，就不多说了。

### Hashing with Null-Terminated String Keys

使用C风格字符串哈希并不是件简单的事情，`std::uordered_map`的b标准定义是

```c++
template<
		typename Key,
		typename Value,
		typename Hash=std::hash<Key>,
		typename KeyEqual=std::equal_to<Key>,
		typename Alloctor=std::alocator<std::pair<const Key,Value>>
        >
class unordered_map;        
```

可以看到几个问题，

- 缺省的Hash函数`std::hash`使用指针的值，而不是指针所指的字符串来进行哈希，于是查找时，指向待查找关键字的指针的哈希结果不大可能与表中的键相等，所以需要自定义一个哈希函数。
- 缺省的KeyEqual对象`std::equal_to`判定指针的值，而不是指针所指的字符串的相等性，所以需要提供一个判别相等的对象

于是需要

```c++
struct hash_c_string
{
    void hash_combine(size_t& seed,T const& v)
    {
        seed^=v+0x9e3779b9+(seed<<6)+(seed>>2);
    }
    std::size_t operator()(char const* p) const
    {
        size_t hash=0;
        for(;*p;++p)
            hash_combine(has,*p);
        return hash;
    }
};

struct comp_c_string
{
    bool operator()(char const* p1,char const* p2) const
    {
        return strcmp(p1,p2)==0;
    }
};

std::unordered_map<char const*,unsigned,hash_c_string,comp_c_string> table;
```

这个效果也不尽人意。

### Hashing with a Custom Hash Table

对于前面的例子，很容易根据首字母创建一个完美最小哈希函数，也就是没有冲突不会有冗余空间的哈希函数。

```c++
unsigned hash(char const* key)
{
    if(key[0]<'a'||key[0]>'z')
        return 0;
    return (key[0]-'a');
}

kv* find_hash(kv* first,kv* last,char const* key)
{
    unsigned i=hash(key);
    return strcmp(first[i].key,key)?last:first+i;
}
```

显然，这里的hash函数并没有什么通用性。

GNU项目有个命令gperf可以用来生成完美哈希函数，对于小数据集，通常也是最小的。

## Stepanov’s Abstraction Penalty

下面是之前算法的一些测试结果

|      | vs2010,i7,1m次迭代,ms | 比上一项提高% | 比基准提高% |
| :--- | :----------------- | :-------------- | :----------: |
| `map<string>` | 2307 |      |      |
| `map<char *> free function` | 1453 | 59 | 59 |
| `map<char *> function object` | 820 | 77 | |
| `map<char *> lambda` | 820 | 0 | 181 |
| `std::find()` | 1425 | | |
| `std::equal_range()` | 1806 | | |
| `std::lower_bound()` | 973 | 53 | |
| `find_binary_strcmp()` | 771 | 26 | 134 |
| `std::unordered_map()` | 509 | | |
|`find_hash()`|195|161|161|

很显然，迭代次数足够多的时候，结果是符合预期的，hash最好，二分查找次之，最后是线性查找。

但是，显然，出于标准库的立足点，标准库算法效率可能会大大低于手写算法。这种标准库算法和良好的手写算法性能上的差距被叫做Stepanov抽象惩罚，Alexander Stepanov是标准库算法和容器类最早的设计者，当初甚至没有编译器可以用。Stepanov抽象惩罚是不可避免的，这也不能说是不好的事情，只是每一个程序员都应该记住这事，也就是手写算法可以比标准库算法性能更好，毕竟立足点是不一样的。

## Optimize Sorting with the C++ Standard Library

在使用分治策略进行查找前，线性数据容器必须先排序。

`std::sort()`使用快速排序的某个变种，现代标准通常会混合使用比方说Timsort和introsort，以保证任何情况下都能达到$O(nlog_2n)$。

`std::stable_sort()`使用归并排序，同样为了保证任何情况下都能达到$O(nlog_2n)$，在递归层次没有那么深的时候使用归并排序，否则就使用堆排序。

## Summary

- C++之所以能够满足各种性能需求，在于其所提供各种各样的特性，在开发过程中可以不断的对这些特性做权衡和选择。

- 优化过程中注意记录，不要依赖人自己的记忆。

- 哈希表虽然比非哈希表搜索要快，但是可能并没有想象的那么快。

- Stepanov抽象惩罚在使用C++标准库时是不可避免的。
