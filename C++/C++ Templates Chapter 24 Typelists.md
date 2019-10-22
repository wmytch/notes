# Chapter 24 Typelists

[TOC]

类型元编程的中心数据结构是*typelist*，如名字所示，就是一个类型的列表。

## 24.1 Anatomy of a Typelist

类型列表与普通列表相同的是可以迭代，可以增加元素，可以移除元素。但与大多数运行时数据结构相比，比方说`std::list`，其不同之处在于类型列表不允许更改。对于`std::list`来说，增加一个元素意味着改变了列表本身，这种改变对程序中的其它访问这个列表的部分是可见的。而对于类型列表，增加一个元素，并不会改变原始的列表，而是创建了一个新的列表。所以，这里不允许更改的意思就是一个对象初始化之后自身就不能再改变了。

类型列表是个典型的类模板特化，其所包含的类型及类型出现的顺序通过模板参数来表示。

```c++
template<typename... Elements>
class Typelist{};
```

于是一个空的类型列表就用`Typelist<>`表示，一个只包含int的类型列表用`Typelist<int>`表示，或者

```c++
using SignedIntegralTypes = Typelist<signed char,short,int,long,long long>;
```

看到这样的句法形式，应该能更好的理解为什么说类型列表不可更改了。

我们可以定义一些列表操作。

```c++
template<typename List>
class FrontT;
template<typename Head,typename... Tail>
class FrontT<Typelist<Head,Tail...>>
{
    public:
    	using Type=Head;
};
template<typename List>
using Front=typename FrontT<List>::Type;

template<typename List>
class PopFrontT;
template<typename Head,typename... Tail>
class PopFrontT<Typelist<Head,Tail...>>
{
    public:
    	using Type=Typelist<Tail...>;
};
template<typename List>
using PopFront=typename PopFrontT<List>::Type;

template<typename List,typename NewElement>
class PushFrontT;
template<typename... Elements,typename NewElement>
class PushFrontT<Typelist<Elements...>,NewElement>
{
    public:
    	using Type=Typelist<NewElement,Elements...>;
};
template<typename List,typename NewElemet>
using PushFront=typename PushFrontT<List,NewElement>::Type;
```

于是，`Front<SingedIntergralTypes>`产生的结果是`signed char`，`PopFront<SignedIntegralTypes>`生成一个新的类型列表`Typelist<short,int,long,long long>`，`PushFront<SignedIntegralTypes,bool>`也是生成一个新的类型列表`Typelist<bool,signed char,short,int,long,long long>`。

到这里，我们可以发现，类型列表的操作，很类似于一些弱类型语言包括函数式语言中列表的操作，只不过C++是通过模板，而这些语言则是语言本身的特性，或者说一级特性。

## 24.2 Typelist Algorithms

通过基本操作，我们可以构造出很多类型列表操作来，比方说将一个类型压入一个类型列表的首部。

```c++
using Type=PushFront<PopFront<SignedIntegralTypes>,bool>;
```

注意，这里不论是中间结果还是最后结果都会各自创建新的类型列表。

### 24.2.1 Indexing

```C++
template<typename List,unsigned N>
class NthElementT:public NthElementT<PopFront<List>,N-1>{};
template<typename List>
class NthElementT<List,0>:public FrontT<List>{};
template<typename List,unsigned N>
using NthElement=typename NthElementT<List,N>::Type;
```

显然，这里使用了递归实例化。说到这里，必须要明确的是编译器不会看到模板定义就去实例化，而是在碰到使用模板的时候才会去实例化，只是重申一下加强印象，不然可能潜意识当中对理解代码造成某种障碍。

`NthElement<List,0>`是递归的终点，继承了`FrontT<List>`，因而也就间接的提供了Type。

因此，
```c++
NthElementT<Typelist<short,int,long>,2>
=> NthElementT<Typelist<int,long>,1>
=> NthElementT<Typelist<long>,0>
=> FrontT<Typelist<long>>
=> long
```

### 24.2.2 Finding the Best Match

```c++
template<typename List>
class LargestTypeT
{
    private:
    	using First=Front<List>;
    	using Rest=typename LargestTypeT<PopFront<List>>::Type;
    public:
    	using Type=IfThenElse<(sizeof(First)>=sizeof(Rest)),First,Rest>;
};
template<>
class LargestTypeT<Typelist<>>
{
    public:
    	using Type=char;
};
template<typename List>
using LargestType=typename LargestTypeT<List>::Type;
```

程序本身没有什么好说的，只是要注意几点

- 对于`Typelist<>`参数，也就是在递归终点，返回的类型是char，因为通常所有的类型事实上都是以char为单位计量大小的。
- 递归终点使用`Typelist<>`，这样就显式的排除了其他形式的类型列表，不过这是可以改进的，下面会看到。
- 一些类型并不支持sizeof，比如void，这时候编译器会报错。如以前所说的，void，nullptr是模板编程的边界条件，需要考虑对边界条件会不会出现沉默的错误，也就是编译器通过了，然而运行时结果却不是想要的。能编译出错的问题就不是问题。

为了解决上面第二条所说的问题，可以引入一个元函数IsEmpty

```C++
template<typename List>
class IsEmpty
{
    public:
    	static constexpr bool value=false;
};
template<>
class IsEmpty<Typelist<>>
{
    public:
    	static constexpr bool value=true;
};

template<typename List,bool Empty=IsEmpty<List>::value>
class LargestTypeT;
template<typename List>
class LargestTypeT<List,false>
{
    private:
    	using Contender=Front<List>;
    	using Best=typename LargestTypeT<PopFront<List>>::Type;
    public:
    	using Type=IfThenElse<(sizeof(Contender)>=sizeof(Best)),Contender,Best>;
};
template<typename List>
class LargestTypeT<List,true>
{
    public:
    	using Type=char;
};
template<typename List>
using LargestType=typename LargestTypeT<List>::Type;
```

对于任意形式的类型列表，只要实现了Front，PopFront，IsEmpty，就可以使用这个LargestType原函数。这里的实现指的是对特定形式的类型列表，提供了这几个模板的特化，比方说上面IsEmpty对Typelist提供了一个完全特化，如果有其他形式的类型列表，同样提供一个即可。除此外，这段代码就没什么好解释的了。

### 24.2.3 Appending to a Typelist

```c++
template<typename List,typename NewElement>
class PushBackT;
template<typename... Elements,typename NewElement>
class PushBackT<Typelist<Elements...>,NewElement>
{
    public:
    	using Type=Typelist<Elements...,NewElement>;
};
template<typename List,typename NewElement>
using PushBack=typename PushBackT<List,NewElement>::Type;
```

这里的问题跟上面LargestType最初版本的问题一样，绑定了Typelist，当然我们也可以用同样的方法来解决这个问题。

```c++
template<typename List,typename NewElement,bool=IsEmpty<List>::value>
class PushBackRecT;
template<typename List,typename NewElement>
class PushBackRecT<List,NewElement,false>
{
        using Head=Front<List>;
        using Tail=PopFront<List>;
        using NewTail=typename PushBackRecT<Tail,NewElement>::Type;
    public:
    	//注意，原书上这里是PushFront<Head,NewTail>,与前面PushFrontT的定义是不符的
    	//改这里还是那里就随意了。
        using Type=PushFront<NewTail,Head>;
};
template<typename List,typename NewElement>
class PushBackRecT<List,NewElement,true>
{
    public:
        using Type=PushFront<List,NewElement>;
};

template<typename List,typename NewElement>
class PushBackT:public PushBackRecT<List,NewElement>{};
template <typename List,typename NewElement>
using PushBack=typename PushBackT<List,NewElement>::Type;
```

除了上面注释说的问题外，别的也没什么好说的了。另外原书上说要移除PushBack的对Typelist的特化，从后面的讨论应该是最初版本的PushBackT的特化，而不是PushBack，不过这个无关紧要。

最后还是回到资源占用问题，对于一个长度为N的Typelist，会实例化出来N+1个PushBackRecT和PushFrontT，以及N个FrontT和PopFrontT，也就是总共要实例化出来4N+2个类实例。如果使用了前面板本的对Typelist的PushBackT的特化，那么资源占用就会变成N个类实例。这个问题必须心里有数。

### 24.2.4 Reversing a Typelist

```c++
template<typename List,bool Empty=IsEmpty<List>::value>
class ReverseT;
//注意一下，别名模板在这里定义，因为下面直接用了别名
template<typename List>
using Reverse=typename ReverseT<List>::Type;

template<typename List>
class ReverseT<List,false>:public PushBackT<Reverse<PopFront<List>>,Front<List>>{};
template<typename List>
class ReverseT<List,true>
{
    public:
        using Type=List;
};

```

很常见递归算法。以ReverseT为基础，可以得到一个PopBackT

```c++
template<typename List>
class PopBackT
{
    public:
    	using Type=Reverse<PopFront<Reverse<List>>>;
};
template<typename List>
using PopBack=typename PopBackT<List>::Type;
```

算法不解释了，唯一要说的是，这里如果从纯粹的算法角度说，效率是不高的，但是这是编译期计算，所以不用担心运行时效率的问题，要考虑的只是编译资源占用问题，比方说这里显见的就有3个List的副本，在生成这3个副本的时候，也会有前面说过的中间副本生成，这些副本占用的资源什么时候释放则要依情况而定。

### 24.2.5 Transfoming a Typelist

有时候需要转化类型列表的元素，比方说增加const修饰。在函数式语言社区中，这种操作称为map，同样也是大数据处理中的map，不是指数据结构中的map。

```c++
template<typename T>
struct AddConstT
{
    using Type=T const;
};
template<typename T>
using AddConst=typename AddConstT<T>::Type;

template<typename List,template<typename T> typename MetaFun,bool Empty=IsEmpty<List>::value>
class TransformT;
template<typename List,template<typename T> typename MetaFun>
class TransformT<List,MetaFun,false>
    :public PushFrontT<typename TransformT<PopFront<List>,MetaFun>::Type,typename MetaFun<Front<List>>::Type> 
{
};
template<typename List,template<typename T> typename MetaFun>
class TransformT<List,MetaFun,true>
{
    public:
        using Type=List;
};
template<typename List,template<typename T> typename MetaFun>
using Transform=typename TransformT<List,MetaFun>::Type;
```

然后就可以这样使用：`using TF=Transform<SignedIntegralTypes,AddConstT>;`

注意这里是AddConstT，而不能是AddConst。我们来看看为什么。首先明确一点，每一个`TransformT<List,MetaFun,false>`，都会实例化一个对应的`PushFrontT<typename TransformT<PopFront<List>,MetaFun>::Type,typename MetaFun<Front<List>>::Type>`，然后在这个PushFrontT中进行递归，这是要牢记的。回到问题，如果我们使用AddConst，假设有`Typelist<long,signed char,int,short,bool,float,double>`，当递归来到`TransformT<Typelist<double>,AddConst,false>`时，对应的就需要实例化一个`PushFrontT<typename TransformT<PopFront<Typelist<double>>,AddConst>::Type,typename AddConst<Front<Typelist<double>>>::Type>`，即

```c++
PushFrontT<typename TransformT<PopFront<Typelist<double>>,AddConst>::Type,
		   typename AddConst<Front<Typelist<double>>>::Type>
==> PushFrontT<typename TransformT<Typelist<>,AddConst>>::Type,typename AddConst<double>::Type>
==> PushFrontT<Typelist<>,typename AddConst<double>::Type>
==> PushFrontT<Typelist<>,typename AddConstT<double>::Type::Type>
==> PushFrontT<Typelist<>,typename double const::Type>
```

也就是说最后多了一个Type，对于double const来说，当然是没有这个属性的，于是就出错了。

如果我们一定要使用别名模板，也就是要把`::Type`去掉，可以把PushFrontT的实际参数中的typename去掉。也就是变成这样的定义

```c++
template<typename List,template<typename T> typename MetaFun>
class TransformT<List,MetaFun,false>
    :public PushFrontT<Transform<PopFront<List>,MetaFun>,MetaFun<Front<List>>> 
{
};
```

这问题本身只是一个风格选择的问题，但是其中也体现了typename的使用以及实例化的两阶段查找机制。

首先回忆一下编译器对模板实例化两步查找机制，即出现的地方和使用的地方。其次，typename使用的句法和语义，在C++17，如果使用typename，后面的类型的名字必须是有依赖性或者说被限定的，也就是需要有`::`来修饰的，在C++11及之前，如果类型名字没有限定，则这个typename是冗余的。

我们已经知道了问题不在于AddConst而是在于PushFrontT的实例化，而这个实例化使用了PushFrontT对Typelist的一个特化，即`PushFrontT<Typelist<Elements...>,NewElement>`。接下来看对PushFrontT不同参数的产生的结果

| 参数                                                         | 结果                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `typename TransformT<PopFront<List>,MetaFun>::Type,typename MetaFun<Front<List>>::Type` | 正确，因为Type不属于实例化上下文，所以编译器并不会去寻找TransformT中Type的定义，而对于MetaFun则是根本不可能找得到，加上typename就是告诉编译器后面跟着的是类型，不用去找了。 |
| `TransformT<PopFront<List>,MetaFun>::Type,MetaFun<Front<List>>::Type` | 正如上面所说，编译器不知道Type是啥，于是出错。但到了第二步是没有错的。 |
| `TransformT<PopFront<List>,MetaFun>,MetaFun<Front<List>>`    | 这里的问题是没有与`TransformT<PopFront<List>`参数匹配的PushFrontT模板定义，也就是第二步出错。如果对此定义一个特化，也是可以的，当然这是另外一个问题了。 |
| `typename TransformT<PopFront<List>,MetaFun>,typename MetaFun<Front<List>>` | 这里问题是两个，第一个涉及到typename使用的句法和语义，如上面说过的，对于c++17，会是第一步的问题，c++11则是第二步的问题。第二个则是找不到匹配的模板，这是第二步的问题 |
| `Transform<PopFront<List>,MetaFun>,MetaFun<Front<List>>`     | 正确，编译器可以看到Transform的定义，PushFrontT的实际参数中没有不属于实例化上下文的对象，所以第一步通过，第二步在替换完参数后也是没有问题的。 |
| `Transform<PopFront<List>::Type,MetaFun>,MetaFun<Front<List>>::Type`。 | 首先编译器不知道Type是啥，其次，Transform展开之后发现`Typelist<>`没有Type成员。也就是说，两步都出错了。 |
| `typename Transform<PopFront<List>::Type,MetaFun>,typename MetaFun<Front<List>>::Type`。 | 最后发现`Typelist<>`没有Type成员，也就是在第二步出错了。     |
| `typename Transform<PopFront<List>,MetaFun>,typename MetaFun<Front<List>>`。 | 纯粹的由于typename的句法和语义引发的第一步的错误，第二步是没有错的。 |
最后可以看到，以上出错的地方主要是在于TransformT或者Transform，MetaFun只是一个模板参数，除了typename句法引发的错误外，只有在用实际参数替换之后最终才会知道是否出错，就如前面只能使用AddConstT而不能使用AddConst一样，这点已经解释过，就不再多说了。

综合上面的讨论，就能很容易地理解为什么下面代码是正确的了
```c++
template<typename List,template<typename T> typename MetaFun>
class TransformT<List,MetaFun,false>
    :public PushFrontT<typename ::Transform<PopFront<List>,MetaFun>,MetaFun<Front<List>>> 
{
};
```
`::Transform<>`前面有没有typename都没有影响，然而`MetaFun`不能改成`typename ::MetaFun`或者`::MetaFun`，因为确实没有MetaFun的定义，它只是个模板的形式参数，因此不能用`::`修饰。

### 24.2.6 Accumulating Typelists

Accumulate指的是，给定一个类型列表`T<T1,T2,…TN>`，一个初始类型`I`，一个元函数`F`，这个函数接受两个类型然后返回一个类型，即$$F(F(F(…F(I,T_1),T_2)…,T_\mathbf{N-1},T_N)$$，其中第`i`步为$$F(F_\mathbf{i-1},T_i)$$。类似的，在函数式编程社区中称为reduce，也是大数据处理中的reduce。通过选择`F`，我们可以达到不同的效果，比方说如果`F`的操作是选择两个输入中较大的那个，我们就可以得到LargestType，如果`F`是接受一个类型列表和一个类型，把类型放到列表最后输出，那么得到Reverse。

```c++
template<typename List,template<typename X,typename Y> typename F,
		 typename I,bool=IsEmpty<List>::value>
class AccumulateT;

template<typename List,template<typename X,typename Y> typename F,typename I>
class AccumulateT<List,F,I,false>
    :public AccumulateT<PopFront<List>,F,typename F<I,Front<List>>::Type>
{
};

template<typename List,template<typename X,typename Y> typename F,typename I>
class AccumulateT<List,F,I,true>
{   
    public:
        using Type=I;
};

template<typename List,template<typename X,typename Y> typename F,typename I>
using Accumulate=typename AccumulateT<List,F,I>::Type;
```
注意这里AccumulateT虽然形式上是递归，但是并不会回头，而是从队首元素开始，逐个的向前推进，直到最后返回结果，所以在递归步中并没有声明Type，只是在递归基中声明。其算法大体上便是这样

```c++
typedef Element& Func(Element& ,Element&);
void accumulate(Typelist& list,Func f,Element elem,Element& result)
{
    if(!list.empty())
    {
        elem=f(front(list),elem);
        accumulate(popFront(list),f,elem,result)
    }
    else
    {
        result=elem;
    }
}
```

当然，对于一个函数而言，由于其调用的形式，是不可能没有回头的，只不过这里的回头只是跳转，而不再做任何处理，类似于尾递归。

然后就可以实现LargestType了

```c++
template<typename T,typename U>
class LargerTypeT:public IfThenElseT<sizeof(T)>=sizeof(U),T,U>
{
};

template<typename Typelist,bool=IsEmpty<Typelist>::value>
class LargestTypeAccT;

template<typename Typelist>
class LargestTypeAccT<Typelist,false>
    :public AccumulateT<PopFront<Typelist>,LargerTypeT,Front<Typelist>>
{
};

template<typename Typelist>
class LargestTypeAccT<Typelist,true>
{
};

template<typename Typelist>
using LargestTypeAcc=typename LargestTypeAccT<Typelist>::Type;

```

所谓的元函数LargerTypeT，顾名思义就是找出较大的那个元素。如前面所述，最后返回的结果是在算法过程中找到的最大值，这个值在执行过程中一直向前推进，直到队列为空，于是返回这个值，并不需要回头。大体上就是

```c++
Element& larger(Element& first,Element& second)
{
    if(first>second)
        return first;
    else 
        return second;
}
Element largest(Typelist& list)
{
    Element result;
    Element init=front(list);
    accumulate(popFront(list),larger,init,result);
    return result;
}
```



程序本身没什么可说的，不过可以与没有使用SFINAE的LargestTypeAccT比较下

```c++
template<typename Typelist>
class LargestTypeAccT
    :public AccumulateT<PopFront<Typelist>,LargerTypeT,Front<Typelist>>
{
};
```

只有一个主模板，使用SFINAE的话，则需要一个主模板的声明，两个分支特化定义。

实现Reverse则直接一些

```c++
using Result=Accumulate<SignedIntegralType,PushFrontT,Typelist<>>;
```

同样，这里也没有回头，只是逐个的把SignedIntegralType中的元素push到一个新队列的头部，当然，每一步都会生成一个新的队列，这点是不能忘记的。

因此，Accumulate可以认为是类型列表操作的主运算，操纵了处理的框架。

### 24.2.7 Insertion Sort

```c++
template<typename T>
struct IdentityT
{   
    using Type=T;
};

template<typename List,typename Element,template<typename T,typename U> typename Compare,bool=IsEmpty<List>::value>
class InsertSortedT;

template<typename List,typename Element,template<typename T,typename U> typename Compare>
class InsertSortedT<List,Element,Compare,false>
{       
        using NewTail=typename IfThenElse<Compare<Element,Front<List>>::value,IdentityT<List>,
    									  InsertSortedT<PopFront<List>,Element,Compare>>::Type;
        using NewHead=IfThenElse<Compare<Element,Front<List>>::value,Element,Front<List>>;
    
    public:
        using Type=PushFront<NewTail,NewHead>;
};

template<typename List,typename Element,template<typename T,typename U> typename Compare>
class InsertSortedT<List,Element,Compare,true>:public PushFrontT<List,Element>
{
};

template<typename List,typename Element,template<typename T,typename U> typename Compare>
using InsertSorted=typename InsertSortedT<List,Element,Compare>::Type;

template<typename List,template<typename T,typename U> typename Compare,bool=IsEmpty<List>::value>
class InsertionSortT;

template<typename List,template<typename I,typename U> typename Compare>
using InsertionSort=typename InsertionSortT<List,Compare>::Type;

template<typename List,template<typename T,typename U> typename Compare>
class InsertionSortT<List,Compare,false>:public InsertSortedT<InsertionSort<PopFront<List>,Compare>,Front<List>,Compare>
{
};

template<typename List,template<typename T,typename U> typename Compare>
class InsertionSortT<List,Compare,true>
{
    public:
        using Type=List;
};

template<typename T,typename U>
struct SmallerThanT
{
    static constexpr bool value=sizeof(T)<sizeof(U);
};

void testInsertionSort()
{
    using Types=Typelist<int,char,short,double>;
    using ST=InsertionSort<Types,SmallerThanT>;
    std::cout<<std::is_same<ST,Typelist<char,short,int,double>>::value<<std::endl;
}
```

这个算法大体上是这样

```c++
typedef bool Func(Element&,Element&);

void insertSorted(Typelist& list,Element& head,Func f)
{
    if(!list.empty())
    {
        Element& newHead=front(list);
        if(f(newHead,head))
        {
            swap(newHead,head);
            popFront(list);
        	insertSorted(list,newHead,f);
       	}
	}
    pushFront(list,head);
 }
Typelist& insertionSort(Typelist& list,Func smallThan)
{    
    if(!list.empty())
    {
        Element head=front(list);
    	popFront(list);
       	insertSorted(insertionSort(list,smallThan),head,smallThan); 
   	}
    return list;
}
```

稍微解释一下，insertionSort总是返回一个已经有序的list，然后在insertSorted中将head插入到list的合适位置，这是这里的不变量。如果insertionSort返回的list的首元素小于head，insertSorted会交换这两个元素，再对交换首元素之后的list进行排序，直到list重新有序。最后将head与list合并，最终返回这个有序的list。

上面解释只是为了更好理解算法思路，但更重要的是如何用模板来实现，毕竟这是一部讲模板的书。

`InsertSortedT<List,Element,Compare,false>`还有另外一种实现

```c++
template<typename List,typename Element,template<typename T,typename U> typename Compare>
class InsertSortedT<List,Element,Compare,false>
    :public IfThenElse<Compare<Element,Front<List>>::value,
					   PushFront<List,Element>,
					   PushFront<InsertSorted<PopFront<list>,Element,Compare>,Front<List>>>
{       
};
```

这种实现的问题就是之前说过的编译时资源占用问题。

## 24.3 Nontype Typelists

类型列表可以用来描述和处理一个类型序列，不过有时候也可以用来处理一些编译时计算的值序列，比方说多维数组的边界或者下标。

可以定义一个CTValue(compile-time value)来生成一个编译时计算的值的类型列表。

```c++
template<typename T,T Value>
struct CTValue
{
    static constexpr T value=Value;
};
template<typename T,T... Values>
using CTTypelist=Typelist<CTValue<T,Values...>;
using Primes=CTTypelist<int,2,3,5,7,11>;
```
这里如果直接使用`using Primes=Typelist<CTValue<int,2>,CTValue<int,3>,CTValue<int,5>,CTValue<int,7>>`的方式定义Primes显然比较繁琐。

我们可以定义一个乘法，然后在Acuumulate中使用

```c++
template<typename T,typename U>
struct MultiplyT;
template<typename T,T Value1,T Value2>
struct MultiplyT<CTValue<T,Value1>,CTValue<T,Value2>>
{
    public:
    	using Type=CTValue<T,Value1*Value2>;
};
template<typename T,typename U>
using Multiply=typename MultiplyT<T,U>::Type;
```
`Accumulate<Primes,MultiplyT,CTV<int,1>>::value`生成Primes中所有元素相乘的积。

我们也可以定义一个新的类型列表Valuelist，并且对其排序

```c++
template<typename T,T... Values>
struct Valuelist{};

template<typename T,T... Values>
struct IsEmpty<Valuelist<T,Values...>>
{   
    static constexpr bool value=sizeof...(Values)==0;
};

template<typename T,T Head,T... Tail>
struct FrontT<Valuelist<T,Head,Tail...>>
{   
    using Type=CTValue<T,Head>;
    static constexpr T value=Head;
};
 
template<typename T,T Head,T... Tail> 
struct PopFrontT<Valuelist<T,Head,Tail...>>                                                                                                                                                    
{                             
    using Type=Valuelist<T,Tail...>;                                                                                                                                                           
};                            
 
template<typename T,T... Values,T New>
struct PushFrontT<Valuelist<T,Values...>,CTValue<T,New>>                                                                                                                                       
{                             
    using Type=Valuelist<T,New,Values...>;                                                                                                                                                     
};                            
 
template<typename T,T... Values,T New>
struct PushBackT<Valuelist<T,Values...>,CTValue<T,New>>                                                                                                                                        
{                             
    using Type=Valuelist<T,Values...,New>;                                                                                                                                                     
};

template<typename T,typename U>
struct GreaterThanT;
template<typename T,T First,T Second>
struct GreaterThanT<CTValue<T,First>,CTValue<T,Second>>
{
    static constexpr bool value=First>Second;
};

void valuelisttest()
{
    using Integers=Valuelist<int,6,2,4,9,5,2,1,7,10>;
    using SortedIntegers=InsertionSort<Integers,GreaterThanT>;
    static_assert(std::is_same_v<SortedIntegers,Valuelist<int,10,9,7,6,5,4,2,2,1>>,"insertion sort failed");
}
```

### 24.3.1 Deducible Nontype Parameters

在C++17中，CTValue可以直接使用auto来推断非类型参数的参数类型

```c++
template<auto Value>
struct CTValue
{
    static constexpr auto value=Value;
};
using Primes=Typelist<CTValue<2>,CTValue<3>,CTValue<5>,CTValue<7>,CTValue<11>>;
```

当然，这样也是可以的

```c++
template<auto... Values>      
using CTTypelist=Typelist<CTValue<Values>...>;
using Primes=CTTypelist<2,3,5,7,11>;
```

用这种方式，我们可以生成不同类型混合的Valuelist

```c++
template<auto... Values>
class Valuelist{};
int x;
using MyValueList=Valuelist<1,'a',true,&x>;
```

## 24.4 Optimizing Algorithms with Pach Expansions

使用包扩展来优化一些使用递归的情况，比方说24.2.5的TransFormT，原先的定义如下

```c++
template<typename List,template<typename T> typename MetaFun,bool Empty=IsEmpty<List>::value>
class TransformT;
//递归步
template<typename List,template<typename T> typename MetaFun>
class TransformT<List,MetaFun,false>
    :public PushFrontT<typename TransformT<PopFront<List>,MetaFun>::Type,typename MetaFun<Front<List>>::Type> 
{
};
//递归基
template<typename List,template<typename T> typename MetaFun>
class TransformT<List,MetaFun,true>
{
    public:
        using Type=List;
};
```

可以增加一个针对Typelist的特化，由于不需要递归，不论是编译效率还是编译资源占用都是更优化的。当然，对于MetaFun的多次实例化是不可避免的。

```c++
template<typename... Elements,template<typename T> typename MetaFun>
class TransformT<Typelist<Elements...>,MetaFun,false>
{
    public:
    	using Type=Typelist<typename MetaFun<Elements>::Type...>;
};

```

当然，包扩展还可以应用到其他需要递归实例化的算法上，比方说24.2.3针对Typelist的PushBack的特化，可以使得Reverse是线性的，而通用版本的PushBack使得Reverse变成二次方的了。**于是我们看到，通过包扩展不但提高了算法效率，还减少了空间占用。**也就是说，算法的时空矛盾，在这里扩展为时间、空间和通用性三者的矛盾。

再看一个算法，从一个列表中选取指定下标的元素生成一个新的列表

```c++
template<typename Types,typename Indices>
class SelectT;
template<typename Types,unsigned... Indices>
class SelectT<Types,Valuelist<unsigned,Indices...>>
{
    public:
        using Type=Typelist<NthElement<Types,Indices>...>; 
};
template<typename Types,typename Indices>
using Select=typename SelectT<Types,Indices>::Type;

using SignedIntegralTypes = Typelist<long,signed char,int,short,bool,float,double>;
using SelectedSignedIntegralTypes =Select<SignedIntegralTypes,Valuelist<unsigned,0,2,4,6>>; 
using ReverseSignedIntegralTypes =Select<SelectedSignedIntegralTypes,Valuelist<unsigned,3,2,1,0>>; 

static_assert(std::is_same_v<SelectedSignedIntegralTypes,Typelist<long,int,bool,double>>,"selecting failed");
static_assert(std::is_same_v<ReverseSignedIntegralTypes  ,Typelist<double,bool,int,long>>,"Reversing failed");
```

像这种包含其他列表下标的非类型列表，通常称为下标列表，或者下标序列，可以用来简化或者消除递归，比如这里的Reverse。

## 24.5 Cons-style Typelists

在一些语言，主要是函数式语言当中，对于列表，有一种叫做cons cell的递归数据结构模型，每一个cons cell包含一个值，这个值是列表的首元素，和一个嵌套的列表，这个列表可以是另一个cons cell，也可以是空表nil。在C++这种结构也可以直接表示为

```c++
class Nil {};
template<typename HeadT,typename TailT=Nil>
class Cons
{
    public:
    	using Head=HeadT;
    	using Tail=TailT;
};
```

于是一个空表就写为Nil，只有一个元素int的列表就是`Cons<int,Nil>`，或者`Cons<int>`，更长的列表则需要嵌套：`using TwoShort=Cons<short,Cons<unsigned short>>;`，更长的列表则依次进行更深的嵌套。

```c++
template<typename List>
class FrontT
{
    public:
        using Type=typename List::Head;
};
template<typename List>
using Front=typename FrontT<List>::Type;

template<typename List,typename Element>
class PushFrontT
{
    public:
        using Type=Cons<Element,List>;
};
template<typename List,typename Element>
using PushFront=typename PushFrontT<List,Element>::Type;

template<typename List>
class PopFrontT
{
    public:
        using Type=typename List::Tail;
};
template<typename List>
using PopFront=typename PopFrontT<List>::Type;

template<typename List>
struct IsEmpty
{
    static constexpr bool value=false;
};
template<>
struct IsEmpty<Nil>
{
    static constexpr bool value=true;
};

void conslisttest()
{
    using ConsList=Cons<int,Cons<char,Cons<short,Cons<double>>>>;
    using SortedTypes=InsertionSort<ConsList,SmallerThanT>;
    using Expected=Cons<char,Cons<short,Cons<int,Cons<double>>>>;
    std::cout<<std::is_same<SortedTypes,Expected>::value<<std::endl;
}
```

比较一下24.2.7及之前的算法，可以看到这里只改动了与Cons直接绑定的操作，当然，把这些操作写成特化也是可以的，而其他并不直接关联Cons的操作是没有改变的。

使用cons风格的列表相对于可变长参数模板有一些不足的地方，首先，对于比较长的列表用嵌套来定义是很邪恶的，其次，对于可变长参数模板，可以使用包扩展来提高效率，最后，可变长参数模板很适合表示混合类型容器。



