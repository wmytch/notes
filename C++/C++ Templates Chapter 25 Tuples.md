# Chapter 25 Tuples

[TOC]

元组与结构的区别在于元组成员通过位置来引用，比方说0，1，2等等，结构的成员通过名字来引用。元组一个重要的性质是可以很容易的依据一个类型列表来构造，加上位置化的接口，使得元组很适合用于基于模板的元编程。

元组也可以看成是在可执行程序中类型列表的一个表现形式。例如，类型列表`Typelist<int,double,std::string>`表示了一个在编译期处理的类型序列，而元组`Tuple<int,double,std::string>`则表示了一个可以在运行时处理的存储单元。

```c++
template<typename... Types>
class Tuple
{
    ...
};
Tuple<int,double,std::string> t(17,3.14,"Hello,World!");
```

因此，在模板元编程中通常都会利用类型列表生成元组，以便用来保存数据。

标准提供了`std::tuple`。接下来看看其大体的实现。

## 25.1 Basic Tuple Design

### 25.1.1 Storage

```c++
template<typename... Types>
class Tuple;
template<typename Head,typename... Tail>
class Tuple<Head,Tail...>
{
    private:
        Head head;
        Tuple<Tail...> tail;
    public:
        Tuple(){}
        Tuple(Head const& head,Tuple<Tail...> const& tail):head(head),tail(tail){} //#1
        Tuple(Head const& head,Tail const&... tail):head(head),tail(tail...){}  //#2

        Head& getHead(){ return head; }
        Head const& getHead() const { return head; }

        Tuple<Tail...>& getTail() { return tail; }
        Tuple<Tail...> const& getTail() const { return tail; }
};
template<>
class Tuple<>{};

template<unsigned N>
struct TupleGet
{
    template<typename Head,typename... Tail>
    static auto apply(Tuple<Head,Tail...> const& t)
    {
        return TupleGet<N-1>::apply(t.getTail());
    }
};

template<>
struct TupleGet<0>
{
    template<typename Head,typename... Tail>
    static Head const& apply(Tuple<Head,Tail...> const& t)
    {
        return t.getHead();
    }
};

template<unsigned N,typename... Types>
auto get(Tuple<Types...> const& t)
{
    return TupleGet<N>::apply(t);
}
```

这里要注意的是

- `Tuple<int,float,std::string> t(17,3.4,“Hello,World!”)`会调用3次#2构造函数，而不是调用一次#1，构造出来的t的存储结构是`Tuple<int,Tuple<float,Tuple<std::string>>>`。3次调用#2其中两次是递归构造tail。那么为什么没有调用#1呢？因为#1需要2个参数，而这里提供了3个。
- `Tuple<char,Tuple<int,float,std::string>> t1('c',t);`会调用2次#2，构造出来的t1的存储结构是`Tuple<char,Tuple<Tuple<int,float,std::string>>>`，所以只能获取`get<0>(t1)`，其他编号会出错。为什么这里不调用#1呢？就参数个数而言两个函数都是匹配的，于是就需要看参数类型，而第一个参数都一样，看第二个参数，#1的第二个形式参数要求是一个`Tuple<Tuple<int,float,std::string>>`，而#2的第二个形式参数要求是一个`Tuple<int,float,std::string>`，显然t匹配#2。第二次调用#2的时候head被初始化为t，而tail是`Tuple<>`，于是递归构造终止。
- `Tuple<char,int,float,std::string> t2('c',t);`会调用一次#1。t2存储结构是`Tuple<char,Tuple<int,Tuple<float,Tuple<std::string>>>>`。这时候#1的第二个形式参数类型是`Tuple<int,float,std::string>`，#2的则是个可变长参数列表，显然t更加匹配#1，并且由于t是一个单独的值，#1中初始化tail时也就不需要再次递归构造了。
- get不能直接在运行期迭代调用。要想在运行期迭代一个Tuple中的元素，只能另想办法。

### 25.1.2 Construction

前面所说的两个构造函数还不是最通用的接口，用户可能想要通过移动构造来初始化一些元素或者通过其它类型的值来构造元素对此可以使用完美转发。为了便于讨论，把类模板定义再列一下，#3和#4是使用完美转发的两个构造函数。另外增加了一个复制构造函数，只是为了说明问题，不讨论一个正确的复制构造函数应该怎么写。

```c++
template<typename Head,typename... Tail>
class Tuple<Head,Tail...>
{
    private:
        Head head;
        Tuple<Tail...> tail;
    public:
        Tuple(){}
    /*
    	//复制构造函数
		Tuple(Tuple const& t):head(t.getHead()),tail(t.getTail())
		{
		}
	*/
    	//#1
        Tuple(Head const& head,Tuple<Tail...> const& tail):head(head),tail(tail){} 
    	//#2
        Tuple(Head const& head,Tail const&... tail):head(head),tail(tail...){}  
    	//#3
		template<typename VHead,typename... VTail>
		Tuple(VHead&& vhead,typename&&... VTail)
    		:head(std::forward<VHead>(vhead)),tail(std::forward<VTail>(vtail)...)
		{
		}
		//#4
		template<typename VHead,typename... VTail>
		Tuple(Tuple<VHead,VTail...> const& other):head(other.getHead()),tail(other.getTail())
		{
		}
    ...
};
```
先看

```c++
Tuple<int,float,std::string> t(1,3.4,"hello,world"); //调用3次#3
```
这里不会考虑#1，因为#1签名是两个形式参数，而t提供了3个实际参数。而选择#3而不是#2的原因是t的参数是3个右值所以会优先选择#3。如果我们给t传送三个已经初始化的左值变量，并且如果把#2的参数的const修饰去掉，这时候是会调用#2的。否则即便t的三个参数是左值，调用的也是#3，正如以前说过的，解析函数重载时，const修饰的左值等同于右值，如果都是右值，自然会优先选择移动构造函数。
```c++
Tuple<long,long double,std::string> t3(t);  //出错
```

上面错误提示是`no viable conversion from 'Tuple<int, float, std::__1::basic_string<char> >' to ‘long’`，也就是说，这时候编译器选择了#3。不考虑#1的原因仍然还是参数个数不同。不选#4的原因是编译器认为一系列独立的值比一个代表了一串值的Tuple更加匹配t，而t也可以是一个独立的值。不选择#2的原因如前面所述。于是在初始化head时出错。

我们再看
```c++
Tuple<char,long,float,std::string> t2('c',t);
```
错误提示是` no viable conversion from 'Tuple<int, float, std::__1::basic_string<char> >' to 'long'`，显然这时候选择的仍然是\#3，只不过这里是在构造tail时出错，t还是一个独立的值，而不是一串值的Tuple。另外模板参数里边的float和std::string用不上便用不上了，这里并没有错误，只是要注意t2的类型仍然是`Tuple<char,long,float,std::string>`而不会是`Tuple<char,long>`，假如t2构造成功的话。

而对于
```c++
Tuple<char,Tuple<long,float,std::string>> t1('c',t);
```
错误提示仍然是`no viable conversion from 'Tuple<int, float, std::__1::basic_string<char>>' to ‘long’`，仍然是\#3，这时候模板参数Tail是`Tuple<long,float,std::string>`，t1.tail的类型就是`Tuple<Tuple<long,float,std::string>>`，需要继续初始化t1.tail，可以知道t1.tail.head的类型是`Tuple<long,float,std::string>`，t1.tail.tail的类型就是`Tuple<>`，仍然需要继续初始化t1.tail.head，由于到此只剩一个参数t，所以之后t总是被作为head的初始化参数，最终t1.tail.head.head的类型是long，而t的类型是`Tuple<long,float,std::string>`，显然不能把t的类型从Tuple转换成long。

因此，需要做些改进。

```c++

//#5
template<typename VHead,typename... VTail,typename=std::enable_if_t<sizeof...(VTail)==sizeof...(Tail)>>
Tuple(VHead&& vhead,VTail&&... vtail):head(std::forward<VHead>(vhead)),tail(std::forward<VTail>(vtail)...)
{
}
//#6
template<typename VHead,typename... VTail,typename=std::enable_if_t<sizeof...(VTail)==sizeof...(Tail)>>
Tuple(Tuple<VHead,VTail...> const &other):head(other.getHead()),tail(other.getTail())
{
}             
```

注意一下,这里两个构造函数是替换上面\#3和\#4的，不能与它们共存，否则会引发重载不明确错误。这样，在复制构造的时候只有长度符合的Tuple才能作为参数。下面举些例子，注释里的说明是按各自构造时的调用顺序排列的。

```c++
Tuple<int,float,std::string> t(1,3.4,"hello,world"); //3次#5
Tuple<char,Tuple<long,float,std::string>> t1('c',t); //2次#5，1次#6
Tuple<char,long,float,std::string> t2('c',t); //1次#5，1次#6
Tuple<long,long double,std::string> t3(t);//2次#6
Tuple<int,float,std::string> t4(t); 
```

我们来看下这里发生了什么。

- 1选择#5的原因与之前选择#3的原因是一致的，之后就不讨论关于#2的问题了。

- 2对#6的调用是在初始化t1.tail.head时，对于#6，Tail的长度是2，VTail的长度也是2，而对于#5，Tail的长度是2，VTail的长度则是0。

- 5实际上调用了缺省复制构造函数，比较有趣的是，如果我们自己提供一个复制构造函数，就如注释掉的那个，就会调用三次这个自定义复制构造函数，前面的2、3、4也有对其的调用。结果都是正确的，然而缺省复制构造函数所做的工作比这个自定义的复制构造函数所做的要多一些。

最后，可以提供一个makeTuple函数模板，通过参数自动推断简化Tuple的生成。

```c++
template<typename... Types>
auto makeTuple(Types&&... elems)
{
    return Tuple<std::decay_t<Types>...>(std::forward<Types>(elems)...);
}
```

## 25.2 Basic Tuple Operations

### 25.2.1 Comparison

```c++
bool operator==(Tuple<> const&,Tuple<> const&)
{
    return true;              
}  
template<typename  Head1,typename... Tail1,
         typename  Head2,typename... Tail2,
         typename=std::enable_if_t<sizeof...(Tail1)==sizeof...(Tail2)>>
bool operator==(Tuple<Head1,Tail1...> const& lhs,Tuple<Head2,Tail2...> const& rhs)
{                             
    return lhs.getHead()==rhs.getHead()&&lhs.getTail()==rhs.getTail();
}
```

代码本身没什么好说的，总之是通过递归，这对于一个本身就是递归定义的数据结构来说是自然的方法。另外要注意的是如果两个Tuple长度不等，这里`==`是没有定义的。如果要改进这一点，最简单当然就是

```c++
template<typename  Head1,typename... Tail1,
         typename  Head2,typename... Tail2>
bool operator==(Tuple<Head1,Tail1...> const& lhs,Tuple<Head2,Tail2...> const& rhs)
{       
        if constexpr (sizeof...(Tail1)==sizeof...(Tail2)){
            return lhs.getHead()==rhs.getHead()&&lhs.getTail()==rhs.getTail();
        }
        else
            return false;
}
```

当然，如果要使用特性也是可以的，这里就不说了。

其他关系运算符`!=`、`>`、`<`、`>=`、`<=`都可以类似进行定义。

### 25.2.2 Output

```c++
void printTuple(std::ostream& strm,Tuple<>const&,bool isFirst=true)
{
    strm<<(isFirst?'(':')');
}
template<typename Head,typename... Tail>
void printTuple(std::ostream& strm,Tuple<Head,Tail...> const& t,bool isFirst=true)
{
    strm<<(isFirst?"(":",");
    strm<<t.getHead();
    printTuple(strm,t.getTail(),false);
}
template<typename... Types>
std::ostream& operator<<(std::ostream& strm,Tuple<Types...> const& t)
{
    printTuple(strm,t);
    return strm;
}
```

这段代码是真正没什么可说的了，除了不要纠结`()`和`,`是用`''`还是`""`外。

## 25.3 Tuple Algorithms

元组是一种容器，通过get来访问和改变其元素，直接或者通过makeTuple来创建，通过getHead和getTail可以把一个元组划分成头和尾。元组算法的有趣之处在于既需要编译期计算也需要运行时计算，所以必须注意生成的代码的执行效率。

### 25.3.1 Tuples as Typelists

如果忽略掉Tuple模板的运行时部分，那么Tuple与Typelist的结构是一样的，都接受任意数量的模板类型参数。实际上，通过一些特化，可以把一个Tuple转换成一个完全的类型列表。

```c++
template<typename List>                                                                        
class IsEmpty 
{
    public:
        static constexpr bool value=false;
};                                                                                             
template<>
struct IsEmpty<Tuple<>>                                                                        
{
    static constexpr bool value=true;
};                                                                                             
        
template<typename List>
class FrontT; 
template<typename Head,typename... Tail>                                                       
class FrontT<Tuple<Head,Tail...>>
{
    public:
        using Type=Head;
};                                                                                             
    
template<typename List>
class PopFrontT;                                                                               
template<typename Head,typename... Tail>                                                       
class PopFrontT<Tuple<Head,Tail...>>
{
    public:
        using Type=Tuple<Tail...>;
};           
template<typename List,typename NewElement>
class PushFrontT;
template<typename... Types,typename Element>
class PushFrontT<Tuple<Types...>,Element>
{
    public:
        using Type=Tuple<Element,Types...>;
};

template<typename List,typename NewElement>
class PushBackT;
template<typename... Types,typename Element>
class PushBackT<Tuple<Types...>,Element>
{
    public:
        using Type=Tuple<Types...,Element>;
};

int main()
{
    Tuple<int,float,std::string> t(1,3.4,"hello,world");
    //注意下面这里跟原书的区别，懒得复制别名模板的定义了。
    using T7=PopFrontT<PushBackT<decltype(t),bool>::Type>::Type;
   	T7 t7(get<1>(t),get<2>(t),true);
   	std::cout<<t7<<std::endl; // 重载的<<运算符参见之前的定义，输出结果是(3.4,hello,world,1)  
}

```

把类型类表的算法应用到元组上通常可以用来帮助决定元组算法的结果的类型。

### 25.3.2 Adding to and Removing from a Tuple

对于元组算法，把一个元素添加到元组的头部或者尾部是基础。与类型列表一样，在头部插入元素比在尾部添加元素要容易得多。

```c++
template<typename... Types,typename V>
auto pushFront(Tuple<Types...> const& tuple,V const& value)
{
    return PushFront<Tuple<Types...>,V>(value,tuple);
}

template<typename V>
auto pushBack(Tuple<> const&,V const& value)
{
    return Tuple<V>(value);
}

template<typename Head,typename... Tail,typename V>
auto pushBack(Tuple<Head,Tail...> const& tuple,V const& value)
{
    return Tuple<Head,Tail...,V>(tuple.getHead(),pushBack(tuple.getTail(),value));
}

template<typename... Types>
auto popFront(Tuple<Types...> const& tuple)
{
    return tuple.getTail();
}
```

auto很好用，另外前面提到过的自定义复制构造函数与缺省复制构造函数有区别就发生在这里，编译器会在返回`Tuple<bool>(true)`后再次调用自定义复制构造函数，于是出现不能把一个`Tuple<bool>`转换成bool的错误，而缺省复制构造函数却不会出这个错误。

### 25.3.3 Reversing a Tuple

```c++
auto reverse(Tuple<> const& t)
{
    return t;
}
template<typename Head,typename... Tail>
auto reverse(Tuple<Head,Tail...> const& t)
{
    return pushBack(reverse(t.getTail()),t.getHead());
}

template<typename... Types>
auto popBack(Tuple<Types...> const& tuple)
{
    return reverse(popFront(reverse(tuple)));
}
```

这里没什么要说的了。

### 25.3.4 Index Lists

上一节reverse函数的问题是运行时效率不佳，主要的原因不在于reverse的递归，而是pushBack的递归，每次都会生成一个新的元组，从而会生成许多Tuple实例。

对于已知长度的元组，可以这样

```c++
Tuple<..> copies; //假定copies有5个元素
auto reversed=makeTuple(get<4>(copies),get<3>(copies),
                        get<2>(copies),get<1>(copies),
                        get<1>(copies));
```

这就是之前在24.4提到过的索引列表的应用。换句话说，就是可以把索引序列作为一个参数包，通过包展开调用get来生成一个新的元组，上面的例子是已经展开的结果。或者说，索引列表的展开是在编译期进行的，而对其的使用是在运行时。

标准类型`std::integer_sequence`通常用来表示索引列表。

### 25.3.5 Reversal with Index Lists

可以使用24.3节定义的`Valuelist<typename T,T… values>`来表示一个索引列表，比如`Valuelist<unsigned,4,3,2,1,0>`。生成这个列表的方法之一可以如下

```c++
template<typename T,T Value>
struct CTValue
{
    static constexpr T value=Value;
};

template<typename T,T... Values>
struct Valuelist
{};

template<unsigned N,typename Result=Valuelist<unsigned>>
struct MakeIndexListT:MakeIndexListT<N-1,PushFront<Result,CTValue<unsigned,N-1>>>
{};

template<typename Result>
struct MakeIndexListT<0,Result>
{
    using Type=Result;
};

template<unsigned N>
using MakeIndexList=typename MakeIndexListT<N>::Type;
```

最初，Result是`Valuelist<unsigned>`，通过递归继承到了`MakeIndexListT<0,Result>`时，Result就会是`Valuelist<unsigned,0,1,…N-1>`。这里需要说明的是，把一个CTValue中的值取出来放到了Result中是通过PushFront的一个特化实现的，其定义是这样的

```c++
template<typename List,typename NewElement>
class PushFrontT;
template<typename T,T... Values,T New>
struct PushFrontT<Valuelist<T,Values...>,CTValue<T,New>>
{   
    using Type=Valuelist<T,New,Values...>;
};
```
因此，如果要把结果反转，就这样

```c++
using MyIndexList=Reverse<MakeIndexList<5>>;  //相当于Valuelist<signed,4,3,2,1,0>
```

这里的Reverse的递归步的定义是

```c++
template<typename List>
class ReverseT<List,false>:public PushBackT<Reverse<PopFront<List>>,Front<List>>{};  
```

而PushBackT，PopFront和Front都使用了前面所定义的针对Valuelist的特化模板。

标准定义的模板的make_index_sequence生成一个元素类型为std::size_t的列表，而更为通用的模板make_integer_sequence则生成一个可以指定元素类型的列表。

于是，reverse就变成了这样

```c++
/*
ReverseT的通用模板定义是
template<typename List,bool Empty=IsEmpty<List>::value>
class ReverseT;
所以需要定义一个IsEmpty<Valuelist<T,Values...>>
*/
template<typename T,T... Values>
struct IsEmpty<Valuelist<T,Values...>>
{
    static constexpr bool value=sizeof...(Values)==0;
};
//reverseImpl的第二个参数只是用来推断模板参数Indices，参数本身函数中并不会使用到，所以没有提供名字。
template<typename... Elements,unsigned... Indices>
auto reverseImpl(Tuple<Elements...> const& t,Valuelist<unsigned,Indices...>)
{
    return makeTuple(get<Indices>(t)...);
}

template<typename... Elements>
auto reverse(Tuple<Elements...> const& t)
{
    //Reverse<...>()实例化一个对象，改成Reverse<...>{}似乎更好理解些
    return reverseImpl(t,Reverse<MakeIndexList<sizeof...(Elements)>>());
}
```

列表本身是在编译期生成，reverseImpl在运行期调用时其参数已经生成。另外注意一下调用makeTuple时参数包展开的句法。

### 25.3.6 Shuffle and Select

上一节的reverseImpl函数本质上根据一个索引列表来选择一个元组中对应的元素，进而生成一个新的元组返回，因此可以将其作为一个通用的算法，而不是一个只能被reverse使用的函数

```c++
template<typename... Elements,unsigned... Indices>                                             
auto select(Tuple<Elements...> const& t,Valuelist<unsigned,Indices...>)                        
{                                                                                              
    return makeTuple(get<Indices>(t)...);                                                      
}                                             
```

除了函数名字不同外，与reverseImpl并没有什么区别。下面是个例子，对一个元组指定位置的元素重复特定次数生成一个新元组

```c++
template<unsigned I,unsigned N,typename IndexList=Valuelist<unsigned>>                         
class ReplicatedIndexListT;                                                                    
                                                                                               
template<unsigned I,unsigned N,unsigned... Indices>                                            
class ReplicatedIndexListT<I,N,Valuelist<unsigned,Indices...>>                                 
    :public ReplicatedIndexListT<I,N-1,Valuelist<unsigned,Indices...,I>>                       
{};                                                                                            
                                                                                               
template<unsigned I,unsigned... Indices>                                                       
class ReplicatedIndexListT<I,0,Valuelist<unsigned,Indices...>>                                 
{                                                                                              
    public:                                                                                    
        using Type=Valuelist<unsigned,Indices...>;                                             
};                                                                                             
                                                                                               
template<unsigned I,unsigned N>                                                                
using ReplicatedIndexList=typename ReplicatedIndexListT<I,N>::Type;                            
                                                                                               
template<unsigned I,unsigned N,typename... Elements>                                           
auto splat(Tuple<Elements...> const& t)                                                        
{                                                                                              
    return select(t,ReplicatedIndexList<I,N>());                                               
}                               
...
Tuple<int,float,std::string> t(1,3.4,"hello,world");
auto t10=splat<1,4>(t);
std::cout<<t10<<std::endl; //(3.4,3.4,3.4,3.4)
```

与前一节同样，调用select函数时的第二个参数是在编译时生成的，使用这个参数的目的并不是要使用这个列表本身，而是要从这个列表来推导出select的模板参数。这里生成索引列表的关键步骤是`ReplicatedIndexListT<I,N-1,Valuelist<unsigned,Indices...,I>>`，从一个`Valuelist<unsigned>`开始，不断的向其中添加索引值，直到指定长度或者说重复次数满足。可以与前一节的MakeIndexListT做个对比，以加强对这里思路的理解。

另外，可以回忆下在24章实现的一个SelectT类模板，通过一个Valuelist来生成一个Typelist，而这里的select函数则是通过一个Valuelist生成一个元组。所以这里生成Valuelist的思路也是适用于SelectT模板的。

再看一个对元组元素根据其类型大小排序的例子，使用了以前实现的插入排序元程序。

```c++
template<typename List,template<typename T,typename U> typename F>
class MetafunOfNthElementT
{
    public:
        template<typename T,typename U> class Apply;
        template<unsigned N,unsigned M>
        class Apply<CTValue<unsigned, M>,CTValue<unsigned, N>>
            :public F<NthElement<List,M>,NthElement<List,N>>{};
};

template<template<typename T,typename U> typename Compare,typename... Elements>
auto sort(Tuple<Elements...> const& t)
{
    return select(t,
                InsertionSort<MakeIndexList<sizeof...(Elements)>,
                              MetafunOfNthElementT<
                                Tuple<Elements...>,Compare>::template Apply>());
}
```

已知select的第二个参数是一个索引列表，所以这里的关键就是这个索引列表是怎么生成的，也就是说怎么理解这里的InsertionSort。可以看到其第一个参数是一个列表，这里是由MakeIndexList生成的索引列表`Valuelist<unsigned，0,1,2,…N-1>`，其第二个参数是用来作比较操作的元函数，因此，问题进而就到了怎么理解`MetafunOfNthElementT<Tuple<Elements...>,Compare>::template Apply`。

首先看这里用template对Apply的修饰，因为InsertionSort的第二个模板参数要求是一个模板，而Apply是一个在类模板中定义的类模板，或者说是一个非独立模板，必须在’::’后加上template，以告诉编译器这是一个模板，或者说，在第一步查找的时候，编译器并不会深入到MetafunOfNthElement内部去寻找Apply的定义，其定义只有在第二步实例化InsertionSort时才会去找，如果第一步出错，就不会等到第二步了。详细的说明可以参见13.3.3。

替换MetafunOfNthElementT的参数之后Apply主模板可以忽略，特化模板的定义就变成

```c++
template<unsigned M,unsigned N>
class Apply<CTValue<unsigned,M>,CTValue<unsigned N>>
    :public Compare<NthElement<Tuple<Elements...>,M>,NthElement<Tuple<Elements...>,N>>
```

我们需要看一下InsertionSort的两个关键模板的定义，详细的讨论可以参见24.2.7。

```c++
template<typename List,typename Element,template<typename T,typename U> typename Compare>
class InsertSortedT<List,Element,Compare,false>
{       
        using NewTail=typename IfThenElse<Compare<Element,Front<List>>::value,IdentityT<List>,InsertSortedT<PopFront<List>,Element,Compare>>::Type;
        using NewHead=IfThenElse<Compare<Element,Front<List>>::value,Element,Front<List>>;
    
    public:
        using Type=PushFront<NewTail,NewHead>;
};
template<typename List,template<typename T,typename U> typename Compare>
class InsertionSortT<List,Compare,false>:public InsertSortedT<InsertionSort<PopFront<List>,Compare>,Front<List>,Compare>
{   
};  
```

首先,这里的List参数由`Valuelist<unsigned,0,1,…N-1>`替换，于是PopFront和Front都是针对这个索引列表操作，我们知道`Front<Valuelist…>`得到的类型是一个`CTValue<...>`。

然后，实例化Compare也就是实例化`Apply<Element,Front<Valuelist<…>>>`，最终结果是`Compare<NthElement<Tuple<Elements...>,M>,NthElement<Tuple<Elements...>,N>>`，也就是说在这个地方实际进行了对Tuple中的元素的比较，比较的标准是Tuple的元素的类型的大小，注意不是值的大小。根据比较结果，实现了对Valuelist进行排序。当然，再次提醒下，这里是生成了一个新的索引列表。另外，`Tuple<Elements…>`是由sort的参数推导出来的，这一点也不要忘记。

最后，sort的结果是在编译期生成的，没有运行期代价。

## 25.4 Expanding Tuples

元组很适合用来把一系列类型不同的值保存到一个值当中，但有时候，我们也需要单独访问其中的元素。

```c++
struct Print
{
    void operator()()
    {}
    template<typename T,typename... Types>                                                     
    void operator()(T const& firstArg,Types const&... args)                                    
    {
        std::cout<<firstArg<<std::endl;
        operator()(args...);                                                                   
    }
};    
template<typename F,typename... Elements,unsigned... Indices>                                  
auto applyImpl(F& f,Tuple<Elements...> const& t,Valuelist<unsigned,Indices...>)                
-> decltype(f(get<Indices>(t)...))                                                             
{                                                                                              
    return f(get<Indices>(t)...);
}
template<typename F,typename... Elements,unsigned N=sizeof...(Elements)>
auto apply(F& f,Tuple<Elements...> &t) -> decltype(applyImpl(f,t,MakeIndexList<N>()))
{
    return applyImpl(f,t,MakeIndexList<N>());
}

...
Tuple<std::string,char const*,int,char> t("Pi","is roughly",3,'\n');
Print print;
apply(print,t); 
```

而原书上的代码是这样的

```c++
template<typename F,typename... Elements,unsigned... Indices>                                  
auto applyImpl(F f,Tuple<Elements...> const& t,Valuelist<unsigned,Indices...>)                
-> decltype(f(get<Indices>(t)...))                                                             
{                                                                                              
    return f(get<Indices>(t)...);
}
template<typename F,typename... Elements,unsigned N=sizeof...(Elements)>
auto apply(F f,Tuple<Elements...> const& t) -> decltype(applyImpl(f,t,MakeIndexList<N>()))
{
    return applyImpl(f,t,MakeIndexList<N>());
}
...
Tuple<std::string,char const*,int,char> t("Pi","is roughly",3,'\n');
apply(print,t); 
```

不清楚原书这里对print的实现到底是怎样的，如果用本节之前书上第4章里print的实现，编译无法通过，因为不能通过print来推断apply模板的参数F的类型。而出于这里的目的，apply对F的自动推断是apply的意义所在，所以不打算针对print定义重载函数，于是定义了一个Print类，以及这个类的`operator()`，print作为Print的一个实例传入，这样就没问题了。另外就是需要把apply函数第二个参数`Tuple<Elements...> const& t`的const去掉，否则也会引发编译错误，但这个不知道是不是Xcode编译后端的问题。

## 25.5 Optimizing Tuple

所谓优化，对于运行期指的是存储空间和执行时间，对于编译器指的是模板实例的数量。

### 25.5.1 Tuples and the EBCO

目前所实现的Tuple类的问题在于浪费了一些空间，原因是tail成员最终会有一个空类作为结束，而我们知道，一个成员必须至少占一个字节的存储空间。可以通过所谓的EBCO(empty base class optimization，21.1)，也就是继承一个空基类而不是将其作为成员来优化这个问题。

~~~c++
//255.h
template<typename... Types>
class Tuple;
template<>
class Tuple<>{};
template<typename Head,typename... Tail>
class Tuple<Head,Tail...>:private Tuple<Tail...>
{
    private:
    	Head head;
    public:
    	Head& getHead(){return head;}
    	Head const& getHead() const {return head;}
    	Tuple<Tail...>& getTail(){return this;}
    	Tuple<Tail...> const& getTail() const{return *this;}
};
~~~

这种方式定义的Tuple其存储形式大体上就是`<<...<<headN>,headN-1>,...>,head0>`。跟之前的定义相比，除了没有存储空基类之外，还可以看到新定义的存储是反向的，并且tail要先于head初始化。我们可以验证一下

```c++
//255.cpp
#include <algorithm>
#include <iostream>           
#include "255.h"              

struct A                      
{                             
    A()
    {
        std::cout<<"A()\n";   
    }                         
};
struct B
{
    B()
    {
        std::cout<<"B()\n";
    }
};

int main()
{
    Tuple<A,char,A,char,B> t1;
    std::cout<<sizeof(t1)<<" bytes\n";
}
```

结果是

```bash
B()
A()
A()
5 bytes
```

这个算不算问题，我觉得其实无所谓的，也许某些场合这种方式更好。

我们可以这样来处理下

```c++
template<typename... Types>                                                                    
class Tuple;
template<typename T>                                                                           
class TupleElt                                                                                 
{
        T value;                                                                               
    public:                                                                                    
        TupleElt()=default;                                                                    
        template<typename U>                                                                   
        TupleElt(U&& other):value(std::forward<U>(other)){}                                    
        T& get() {return value;}                                                               
        T const& get() const{return value;}                                                    

};  
template<typename Head,typename... Tail>                                                       
class Tuple<Head,Tail...>:private TupleElt<Head>,private Tuple<Tail...>                        
{   
    public:                                                                                    
        Head& getHead()                                                                        
        {
            return static_cast<TupleElt<Head> *>(this)->get();                                 
        }
        Head const& getHead() const
        {                                                                                      
            return static_cast<TupleElt<Head> const*>(this)->get(); 
        }
        Tuple<Tail...>& getTail(){return *this;}
        Tuple<Tail...> const& getTail() const {return *this;}
};
template<>
class Tuple<>{};
```

同样用上面的程序，会引发编译警告，不过还是可以运行的，但是要注意声明t1时并没有提供元素值，否则就会编译错误了，当然这个错误是另外一个问题，跟这里的讨论关系不大，就不管它了。运行结果是

```bash
A()
A()
B()
5 bytes
```

可见这种方式解决了初始化顺序的问题。然而，前面提到会引发编译警告，实际上这意味着这种方式引入了一个新的更为糟糕的问题，造成的结果是无法从有相同类型元素的元组中提取元素。我们来看看什么情况，其中一条编译警告是这样的

```bash
warning: direct base 'TupleElt<char>' is inaccessible due to ambiguity:
    class Tuple<char, struct A, char, struct B> -> TupleElt<char>
    class Tuple<char, struct A, char, struct B> -> Tuple<struct A, char, struct B> -> Tuple<char, struct B> -> TupleElt<char> [-Winaccessible-base]
class Tuple<Head,Tail...>:private TupleElt<Head>,private Tuple<Tail...>

```

这里的意思是当继承来到`Tuple<char,A,char,B>`时，其两个基类分别是`TupleElt<char>`和`Tuple<struct A, char, struct B>`，而后者最终也会归结到一个`TupleElt<char> `基类上，于是，一个派生类的继承树上有两个各自独立的相同的基类，并且共享同一存储空间，因此，如果我们要提取其中的char类型的元素，那么提取到的是哪一个？

为了解决这个问题，事实上，前面章节也提到过类似的问题，那里的解决方法也是可以用到这里来的，就是给TupleElt模板增加一个表示高度的参数，这样一个模板类就可以用两个参数来确定，而不是仅仅通过一个类型参数。

```c++
template<unsigned Height,typename T>
class TupleElt
{
        T value;
    public:
        TupleElt()=default;
        template<typename U>
        TupleElt(U&& other):value(std::forward<U>(other)){}
        T& get() {return value;}
        T const& get() const{return value;}

};
template<typename Head,typename... Tail>
class Tuple<Head,Tail...>:private TupleElt<sizeof...(Tail),Head>,private Tuple<Tail...>
{
        using HeadElt=TupleElt<sizeof...(Tail),Head>;
    public:
        Head& getHead()
        {
            return static_cast<HeadElt *>(this)->get();  
        }
        Head const& getHead() const
        {
            return static_cast<HeadElt const*>(this)->get(); 
        }
        Tuple<Tail...>& getTail(){return *this;}
        Tuple<Tail...> const& getTail() const {return *this;}
};
```

于是问题解决。不过我们还可以继续做些优化，只需要修改TupleElt：

```c++
template<unsigned Height,typename T,bool=std::is_class<T>::value&&!std::is_final<T>::value>
class TupleElt;
template<unsigned Height,typename T>
class TupleElt<Height,T,false>
{
        T value;
    public:
        TupleElt()=default;
        template<typename U>
        TupleElt(U&& other):value(std::forward<U>(other)){}

        T& get() {return value;}
        T const& get() const{return value;}
};
template<unsigned Height,typename T>
class TupleElt<Height,T,true>:private T
{
    public:
        TupleElt()=default;
        template<typename U>
        TupleElt(U&& other):T(std::forward<U>(other)){}

        T& get(){return *this;}
        T const& get() const {return *this;}
};
```

运行结果会是

```bash
A()
A()
B()
2 bytes
```

原因是A和B都是空类并且不是final类，于是`TupleElt<Height,T,true>`继承了两个空类，编译器对此就可以应用EBCO。

### 25.5.2 Constant-time get()

由于`get()`是递归实现的，会实例化一系列的模板，这一系列模板实例化是线性时间的，从而会影响到编译的时间。不过可以通过EBCO来实现一个常量时间的get函数。

```c++
template<unsigned H,typename T>
T& getHeight(TupleElt<H,T>& te)
{
    return te.get();
}
template<typename... Types>
class Tuple;
template<unsigned I,typename... Elements>
auto get(Tuple<Elements...>& t)
{
    return getHeight<sizeof...(Elements)-I-1>(t);
}
```

这里H的计算以及对T的推断都是常数时间的，另外，getHeight必须声明为Tuple的友元函数，以便把一个Tuple隐含转换成其私有的基类TupleElt。Tuple的存储镜像大体上是`(TupleElt<H,T0>,TupleElt<H-1,T1>,…<TupleElt<0,TN>)`，于是`getHeight<I>`就直接获取了其中的某一个TupleElt。

## 25.6 Tuple Subscript

理论上，元组也是可以定义`operator[]`的，但是与`std::vector`不同，元组的元素的类型各不相同，所以元组的`operator[]`必须是个模板，为了能够依据函数参数推断出模板参数，从而实例化出来独一的函数，可以使用CTValue来做索引

```c++
template<typename T,T Index>
auto& operator[](CTValue<T,Index>)
{
    return get<Index>(*this);
}
```

于是

```c++
auto t=makeTuple(0,'1',2.2f,std::string{"hello"});
auto a=t[CTValue<unsigned,2>{}]; //2.2f
auto b=t[CTValue<unsigned,3>{}]; //"hello"
```

这里要注意一个问题，就是由于`operator[]`的返回值类型是`auto&`，所以get的返回值也需要是个引用类型，或者把这里返回值的引用去掉，相应的get也就不需要返回一个引用了。

另外要注意的是我们不能这样定义`operator[]`

```c++
auto operator[](unsigned Index) 
{                     
	return get<Index>(*this);      
} 
```

原因在于作为`operator[]`参数的Index是个运行期才确定的值，而作为get模板实参的Index是个编译期就必须确定的值。这也是这里使用CTValue的原因，可以把一个值封装成一个类型，然后get就可以做一些参数推断，从而选出或者说实例化出一个合适的函数模板实例。

显然像`CTValue<unsigned,2>{}`这样的表达式不太方便，我们可以实现一个所谓的字面量运算符后缀`_c`，来简化这个表达式的表示方式

```c++
constexpr int toInt(char c)
{   
    if(c>='A' && c<='F')
    {   
        return static_cast<int>(c)-static_cast<int>('A')+10;
    }
    if(c>='a' && c<='f')
    {   
        return static_cast<int>(c)-static_cast<int>('a')+10;
    }
    assert(c>='0' && c<='9');
    return static_cast<int>(c)-static_cast<int>('0');
}
template<std::size_t N>
constexpr int parseInt(char const (&arr)[N])
{   
    int base=10;
    int offset=0;
    
    if(N>2 &&arr[0]=='0')
    {   
        switch(arr[1])
        {   
            case 'x':
            case 'X':
                base=16;
                offset=2;
                break;
            case 'b':
            case 'B':
                base=2;
                offset=2;
                break;
            default:
                base=8;
                offset=1;
                break;
        }
    }
    int value=0;
    int multiplier=1;
    for(std::size_t i=0;i<N-offset;++i)
    {
        if(arr[N-1-i]!='\n')
        {
            value+=toInt(arr[N-1-i])*multiplier;
            multiplier*=base;
        }   
    }   
    return value;
}   
template<char... cs>
constexpr auto operator"" _c()
{
    return CTValue<int,parseInt<sizeof...(cs)>({cs...})>{};
}

auto t=makeTuple(0,'1',2.2f,std::string{"hello"});
auto a=t[2_c]; //2.2f
auto b=t[3_c]; //"hello"
```

除了`_c`定义外，这里没什么需要说明的，至于`_c`，可以参见15.5.1。