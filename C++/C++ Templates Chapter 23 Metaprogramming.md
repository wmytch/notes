# Chapter 23 Metaprogramming

[TOC]

元编程隐含着反射，依赖于特性和类型操作，尽可能的在翻译期完成用户定义的计算，背后的驱动力是性能和简化。大体上元编程指的就是这么个意思。

## 23.1 The State of Modern C++ Metaprogramming

### 23.1.1 Value Metaprogramming

~~~c++
template<typename T>
constexpr T sqrt(T x)
{
    if(x<=1)
    {
        return x;
    }
    
    T lo=0,hi=x;
    for(;;)
    {
        auto mid=(hi+lo)/2, midSquared=mid*mid;
        if(lo+1>=hi || midSquared==x)
        {
            return mid;
        }
        
        if(midSquared<x)
        {
            lo=mid;
        }
        else
        {
            hi=mid;
        }
    }
}
~~~

这个函数可以在编译期计算也可以在运行时调用

```c++
static_assert(sqrt(25)==5,""); //OK
static_assert(sqrt(40)==6,""); //OK,注意是int型
std::array<int,sqrt(40)+1> arr; //7个元素的array
long long l=43478;
std::cout<<sqrt(l)<<std::endl;
```

当然这个函数在运行时调用执行效率并不高，但是我们这里强调的是编译期计算，这时候就不存在运行时效率的问题了。这个函数是纯粹的C++代码，并没有什么特别的所谓的模板戏法，只是简单的在编译期计算一个值。

### 23.1.2 Type Metaprogramming

c++标准库提供了一个类型特性`std::remove_all_extents`，大体上的实现是这样的

```c++
template<typename T>
struct RemoveAllExtentsT
{
    using Type=T;
};
template<typename T,std::size_t SZ>
struct RemoveAllExtentsT<T[SZ]>
{
    using Type=typename RemoveAllExtentsT<T>::Type;
};
template<typename T>
struct RemoveAllExtentsT<T[]>
{
    using Type=typename RemoveAllExtentsT<T>::Type;
};
template<typename T>
using RemoveAllExtents=typename RemoveAllExtentsT<T>::Type;

int main()
{
   std::cout<<typeid(RemoveAllExtents<int[]>).name()<<std::endl;
   std::cout<<typeid(RemoveAllExtents<int[5][10]>).name()<<std::endl;
   std::cout<<typeid(RemoveAllExtents<int[][10]>).name()<<std::endl;
   std::cout<<typeid(RemoveAllExtents<int*[5]>).name()<<std::endl;
   std::cout<<typeid(RemoveAllExtents<int(*)[5]>).name()<<std::endl;
   std::cout<<typeid(RemoveAllExtents<int*()>).name()<<std::endl;   
   std::cout<<typeid(RemoveAllExtents<int(*)()>).name()<<std::endl; 
}                             
```

运行结果是

```bash
i
i
i
Pi
PA5_i
FPivE
PFivE
```

这里RemoveAllExtents就是个所谓的类型宏函数，可以递归的移除所谓的数组层，只留下元素的类型。不过从上面的结果我们看到对`int(*)[5]`产生的类型就是`int(*)[5]`，而不是`int(*)`，这是为什么呢，当然我们知道这是一个指针，这个指针指向一个有5个int元素的数组，也就是说生成的结果类型是正确的，而产生这个结果的原因是`int(*)[5]`不匹配任意一个特化。那么，为什么呢？我们先看看最后两个例子，显然这两个是不会匹配特化的，于是生成的类型一个是返回指针的函数，一个是函数指针，两个表达式的区别只是在于`*`有没有`()`括起来，也就是说，因为`()`的存在，使得编译器在推断类型时首先推断了`int(*)[5]`是个指针，自然就找不到匹配的特化。另外，不要想着`int(*)([5])`或者`(int *)[5]`这样的表达式，编译器会认为这是不合句法的lambda表达式，但是`int*([5])`或者`int(*[5])`在这里是合法的，只不过简单的把`()`忽略了并且不改变结果，都是`int*`。总之，这里对类型的判断是符合通常的规则的。

正如值容器，比方说数组，vector，哈希表等等，c++也有类型容器`Typelist<…>`，将来会看到。

### 23.1.3 Hybrid Metaprogramming

通过取值元编程和类型元编程我们可以在编译期计算值和类型。我们最终感兴趣的是运行时效果，因此在运行时代码中需要类型和常量的地方我们使用了类型或者取值元编程。然后我们通过例子来看看什么是混合元编程。

假定要计算两个`std::array`的点积，我们已知其是固定长度的容器：

```c++
namespace std
{
    template<typename T,size_t N> struct array;
}

template<typename T,std::size_t N>
auto dotProduct(std::array<T,N> const& x,std::array<T,N> const &y)
{
    T result{};
    for(std::size_t k=0;k<N;++k)
    {
        result+=x[k]*y[k];
    }
    return result;
}
```

然而在某些平台上这个循环语句可能会被编译成某种分支指令，相对来说代价要比顺序执行一系列的`result+=x[k]*y[k]`语句高一些。当然，现代编译器能够自行对循环进行优化，选择目标平台上最有效率的形式。

出于讨论的目的，可以重写一个避免了循环的算法，当然，显式避免循环的方式并不值得提倡，至少出于可移植性的理由，这个工作还是交给编译器来进行比较好。

```c++
template<typename T,std::size_t N>
struct DotProductT
{
    static inline T result(T* a,T* b)
    {
        return (*a)*(*b)+DotProductT<T,N-1>::result(a+1,b+1);
    }
};
template<typename T>
struct DotProductT<T,0>
{
    static inline T result(T*,T*)
    {
        return T{};
    }
}

template<typename T,std::size_t N>
auto dotProduct(std::array<T,N> const& x,std::array<T,N> const& y)
{
    return DotProductT<T,N>::result(x.begin(),y.begin());
}
```

我们知道在类定义内定义的函数是隐含为inline的，这里显式的定义result为inline的目的只是作为一个提示，这样一些编译器就会尽最大可能的将其处理成inline。

不过这里关注的中心是把编译期计算与运行时计算，也就是把对DotProductT的递归模板实例化与对result的调用结合起来。在这里，编译期计算决定了代码的整体结构，运行时计算决定了运行时效果。这就是所谓的混合型元编程。

如前面所看到的，类型容器极大增强了类型元编程的能力，上面例子中固定长度的数组对于混合元编程也是很有用的。不过在混合元编程中元组才是容器的主角。在C++中我们可以这样定义一个元组`std::tuple<int,std::string,bool> tVal{42,“answer”,true};`。

除了元组之外，c++还提供了类似于union的`std::variant`，也是可以用于混合元编程的。这两者与struct一样，都是异构的数据类型，所以混合元编程也被称为异构元编程。

### 23.1.4 Hybrid Metaprogramming for Unit Types

混合计算的另一个例子是对不同单位的值进行求值。

```c++
template<unsigned N,unsigned D=1>
struct Ratio
{   
    static constexpr unsigned num=N;  //分子
    static constexpr unsigned den=D;  //分母
    using Type=Ratio<num,den>;
};

template<typename R1,typename R2>
struct RatioAddImpl
{   
    private:
        static constexpr unsigned den=R1::den * R2::den;
        static constexpr unsigned num=R1::num*R2::den+R2::num*R1::den;
    public:
        using Type=Ratio<num,den>;
};
template<typename R1,typename R2>
using RatioAdd=typename RatioAddImpl<R1,R2>::Type;

template<typename T,typename U=Ratio<1>>
class Duration
{
    public:
        using ValueType=T;
        using UnitType=typename U::Type;
    private:
        ValueType val;
    public:
        constexpr Duration(ValueType v=0):val(v){}
        constexpr ValueType value() const
        {
            return val;
        }
};
template<typename T1,typename U1,typename T2,typename U2>
auto constexpr operator+(Duration<T1,U1> const& lhs,Duration<T2,U2> const& rhs)
{
    using VT=Ratio<1,RatioAdd<U1,U2>::den>;
    auto val=lhs.value()*VT::den/U1::den*U1::num+rhs.value()*VT::den/U2::den*U2::num;
    return Duration<decltype(val),VT>(val);
}

int main()
{
    int x=42;
    int y=77;

    auto a=Duration<int,Ratio<1,1000>>(x);  //x的单位是毫秒
    auto b=Duration<int,Ratio<2,3>>(y);  //y的单位是2/3秒
    auto c=a+b;
    std::cout<<c.value()<<std::endl;  //输出154126，单位是1/3000秒
}                   
```

先看程序本身。这个程序的主要工作由`operator+`完成，我们看`RatioAdd<U1,U2>::den`，其中U1为`Ratio<1,1000>`，U2为`Ratio<2,3>`，这个表达式就是`Ratio<1*3+2*1000,3*1000>::den=Ratio<2003,3000>::den=3000`，于是`VT=Ratio<1,3000>`，这里的计算都是在编译期进行的。而`val=42*3000/1000*1+77*3000/3*2=126+154000=154126`，这等价于`(42/1000+77*2/3)*3000`。事实上，结果的单位当然可以是任意的，只要是合理的，这里这样选取VT可以简化计算，否则比方说假如我们选择`VT=Ratio<2003,3000>`，那么就需要计算`(42/1000+77*2/3)*3000/2003 `，因此不需要纠结于VT的取值。另外，对`Duration<T,U>`，T可以是int，也可以是float，甚至也可以是混合类型，只要定义好了相关的运算。

程序说完，回到主题。上面已经提到了，VT的计算是在编译期，而对c的计算则是在运行期，这就是所谓的混合元编程。而实际上，由于`operator+`是constexpr，因此有可能对c的计算会在编译时完成，因为其所计算的对象的值在编译期就是已知的了。

标准库提供的`std::chrono`有类似的功能，不过更为精细，具体就看文档吧。

## 23.2 The Dimensions of Reflective Metaprogramming

前面描述了基于constexpr的取值元编程和基于模板递归实例化的类型元编程。但是，并不是说只能如此搭配，至少，通过模板递归实例化实现的取值元编程早就存在了。

```c++
template<int N,int LO=1,int HI=N>
struct Sqrt
{
    static constexpr auto mid=(LO+HI+1)/2;
    static constexpr auto value=(N<mid*mid) ? Sqrt<N,LO,mid-1>::value : Sqrt<N,mid,HI>::value;
};
template<int N,int M>
struct Sqrt<N,M,M>
{
    static constexpr auto value=M;
};
```

通常，一个完整的C++元编程解决方案需要从三个维度进行选择

- 计算
- 反射
- 生成

计算指的是普遍意义上的计算，反射指能够用程序来检查程序的特性，生成指的是可以为程序生成额外的代码。

我们已经看到了对计算有两个可选项，递归实例化和constexpr求值，需要了解的一点是类模板的递归实例化会占用大量的编译时资源，并且只有到了编译结束才会释放，而constexpr求值则意味着引入反射信息。

对于反射，通过类型特性也有部分的解决方案，但是这远远不够。而且通过特性解决反射仍然脱离不了模板递归实例化或者constexpr求值，上面所说的对反射也是同样的。

而生成，代码自动生成仍然是个挑战，当然模板实例化本身也是一种代码生成机制，此外，现代编译器对inline函数的处理也可以认为是一种代码生成机制。

事实上，前面的DotProductT例子已经体现了这里的考虑。

## 23.3 The Cost of Recursive Instantiation

这一节研究上面的Sqrt模板，算法不需要过多解释，形式上这是一个二分查找，但实际上并没有那么简单。

### 23.3.1 Tracking All Instantiations

比方说，要计算16的平方根，也就是`Sqrt<16>::value`，这会展开成`Sqrt<16,1,16>::value`。于是有

```c++
(16<9*9) ? Sqrt<16,1,8>::value : Sqrt<16,9,16>::value
```

事实上，编译器这里会实例化两个分支，并且由于使用了`::`运算符，所有的成员都会被实例化。因此，这里实际上会生成一棵完全二叉树，也就是说，生成的类实例的数量大概是2*N个，并且这些耗费的编译期资源只有到了编译结束才会释放。

因此，我们需要做些改进，比方说以前使用过的IfThenElse

```c++
template<int N,int LO=1,int HI=N>
struct Sqrt
{
    static constexpr auto mid=(LO+HI+1)/2;
    using SubT=IfThenElse<(N<mid*mid),Sqrt<N,LO,mid-1>,Sqrt<N,mid,HI>>;
    static const auto value=SubT::value;
};
template<int N,int S>
struct Sqrt<N,S,S>
{
    static constexpr auto value=S;
};
```

这里需要重申的只是，定义一个类模板的别名并不会让编译器完全实例化一个类模板。因此，这种方式生成的类实例只有`log(N)`个。

## 23.4 Computational Completeness

`Sqrt<>`例子说明了模板元编程可以包含

- 状态变量：类模板
- 循环结构：通过递归
- 执行路径选择：通过条件表达式或者特化
- 整数算术

因为使用了递归，所以就会有潜在的资源耗尽问题，所以使用模板元编程要小心。

## 23.5 Recursive Instantiation versus Recursive Template Arguments

考虑下面的递归模板

```c++
template<typename T,typename U>
struct Doublify{};

template<int N>
struct Trouble
{
    using LongType=Doublify<typename Trouble<N-1>::LongType,typename Trouble<N-1>::LongType>;
};

template<>
struct Trouble<0>
{
    using LongType=double;
};
Trouble<10>::LongType ouch;
```

于是，`Trouble<10>::LongType`会引发一系列的实例化，不仅仅是`Trouble<>`,还有相应的`Doublify<>`，或者说LongType所代表的类型的长度越来越长，而且是指数增长的。

|     Types Alias      | Underlying Type |
| :------------------: | :-------------: |
| Trouble<0>::LongType |     double      |
|Trouble<1>::LongType|Doublify<double,double>|
|Trouble<2>::LongType|Doublify<Doublify<double,double>,Doublify<double,double>>|

这种不断增长的名字对编译器来说也是一个负担。虽然这是编译器需要并且可以解决的问题，但是在编写模板递归实例化的代码时也最好避免嵌套的模板参数的递归。

## 23.6 Enumeration Values versus Static Constants

在C++早期，枚举值是唯一能够创建“真常量”的机制，这被称为*常量表达式*。比方说

```c++
template<int N>
struct Pow3
{
    enum{value=3*Pow3<N-1>::value};
};
template<>
struct Pow3<0>
{
    enum{value=1};
};
```

当然，这里只是复习一下。从C++98开始我们就可以使用类内的静态常量，所以可以这样

```c++
template<int N>
struct Pow3
{
    static int const value =3*Pow3<N-1>::value;
};
template<>
struct Pow3<0>
{
    static int const value=1;
};
```

但是这里有一个问题，静态常量成员是个左值，所以对于`void foo(int const&)`这样的函数，如果我们这样调用：`foo(Pow3<7>::value)`，那么编译器必须传递`Pow3<7>::value`的地址，这就使得编译器必须实例化这个静态成员并分配地址，或者说，这时候就不是一个纯粹的编译期实现的效果了，这里意思传入的不是一个在编译期就决定的值，而是获取一个地址，等到运行期才去这个地址获取其值。

而枚举值不是左值，也就是说其没有地址，因此，如果对其用引用传递，并不需要分配静态地址，就是传了一个值进去而已，注意，这里的引用是const的，所以传一个字面量作为参数是没有问题的。

到了C++11，引入了constexpr，这个并不是为了解决上述的地址问题，只是使类的静态数据成员不再限于整型，而C++17增加了inline静态数据成员，与constexpr配合使用，从而解决了上述的地址问题。

也就是说到了c++17才通过`inline static constexpr auto` 实现了对枚举值常量的完全替代。

