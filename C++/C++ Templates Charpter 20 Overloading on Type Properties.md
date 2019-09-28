# Charpter 20 Overloading on Type Properties

[TOC]

我们知道，像下面这样的重载是无效的

```c++
template<typename Number> void f(Number);
template<typename Container> void f(Container);
```

不过，我们且看

## 20.1 Algorithm Specialization

```c++
template<typename T>
void swap(T& x,T& y)
{
    T tmp(x);
    x=y;
    y=tmp;
}
template<typename T>
void swap(Array<T>& x,Array<T>& y)
{
    swap(x.ptr,y.ptr);
    swap(x.len,y.len);
}
```

像这里的第二个模板这样的算法或者说函数，通常称之为算法特化。所谓算法特化，其参数比通用模板参数更为特别，其操作针对其参数，所以比通用模板更为有效。

当然，本质上还是重载，所以像下面这样的特化必然是无效的，当然，不要在意这里没有做边界检查。

```c++
template<typename InputIterator,typename Distance>
void advanceIter(InputIterator& x,Distance n)
{
    while(n>0)
    {
        ++x;
        --n;
    }
}

template<typename RandomIterator,typename Distance>
void advanceIter(RandomIterator& x,Distance n)
{
    x+=n;
}
```

原因就不必啰嗦了，不过这里体现出来的想法还是很有意义的，接下来的内容会讨论怎么实现。

另外，C++标准库提供了`std::advance()`来实现把一个迭代器移动n步的算法。

## 20.2 Tag Dispatching

其中的一个方法是提供一个独一的标签

```c++
template<typename Iterator,typename Distance>
void advanceIterImpl(Iterator& x,Distance n,std::input_iterator_tag)
{
    while(n>0)
    {
        ++x;
        --n;
    }
}

template<typename Iterator,typename Distance>
void advanceIterImpl(Iterator& x,Distance n,std::random_access_iterator_tag)
{
    x+=n;
}

template<typename Iterator,typename Distance>
void advanceIter(Iterator& x,Distance n)
{
    advanceIterImpl(x,n,
                   typename std::iterator_traits<Iterator>::iterator_category());
}
```

要注意`typename std::iterator_traits<Iterator>::iterator_category()`，这里的意思是`T()`,改成`{}`就更清晰了。

另外提一句，advanceIter通常要处理成inline的，这样就不会有额外的运行时代价了。

`iterator_traits<>::iterator_category`返回的是下面这些标签

```c++
namespace std
{
    struct input_iterator_tag {};
    struct output_interator_tag {};
    struct forward_iterator_tag:public input_iterator_tag {};
    struct bidirectional_iterator_tag:public forward_iterator_tag {};
    struct random_access_iterator_tag:public bidirectional_iterator_tag {};
}
```

虽然random_access_iterator_tag是input_iterator_tag的子类，但是更为特别，所以重载解析的时候会优先考虑。标签派遣特别适合这种有明显层次结构的情况。但是对于一些与类型紧密关联的属性就不那么方便了，比方说一个类型T是否有一个简单的复制赋值运算符，对于这样的问题，当然也是有办法的，实际上之前我们已经碰到过这种问题了。

另外，B.S.的C++PL里边一开始就使用了像这种定义一个空类做为标签的技巧，但是并没有深入讨论，其他的作者也是如此，毕竟，这种技巧不过是C++自定义类型的一个应用，并不是很高深的内容。而作为一个语言学习者通常来说对没有深入讨论的内容是很容易就会忽略掉的。

## 20.3 Enabling/Disabling Function Templates

```c++
template<bool,typename T=void>
struct EnableIfT{};
template<typename T>
struct EnableIfT<true,T>
{
    using Type=T;
};
template<bool Cond,typename T=void>
using EnableIf=typename EnableIfT<Cond,T>::Type;

template<typename Iterator>
constexpr bool IsRandomAccessIterator=
        IsConvertible<typename std::iterator_traits<Iterator>::iterator_category,std::random_access_iterator_tag>;

template<typename Iterator,typename Distance>
EnableIf<IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x,Distance n)
{
    x+=n;
}
```

这里EnableIf作为一个别名模板，如果条件为真，则会得到一个void类型，作为advanceIter的返回值。如果条件为假，因为并没有Type这么个成员存在，就会引发SFINAE，从而屏蔽掉这个advanceIter。

然而，如果我们提供了这样一个重载函数

```c++
template<typename Iterator,typename Distance>
void advanceIter(Iterator& x,Distance n)
{
    while(n>0)
    {
        ++x;
        --n;
    }
}
```

就会引发重载不明确的错误。因为这两个函数在`IsRandomAccessIterator<Iterator>`为true时会同时存在，并且确实是一样的。所以上面的函数需要改成这样

```c++
template<typename Iterator,typename Distance>
EnableIf<!IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x,Distance n)
{
    while(n>0)
    {
        ++x;
        --n;
    }
}
```

注意，这里并不是通过返回值来解析重载，而是通过SFINAE来屏蔽其中的一个函数模板。

### 20.3.1 Providing Multiple Specialization

上面的方法也适用于有多个可选函数的情况，只要每种情况都是独一的。假定需要一个可以迭代器后退的操作，那么

```c++
template<typename Iterator>
constexpr bool IsBidirectionalIterator=
        IsConvertible<typename std::iterator_traits<Iterator>::iterator_category,std::bidirectional_iterator_tag>;

template<typename Iterator,typename Distance>
EnableIf<IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x,Distance n)
{
    x+=n;
}

template<typename Iterator,typename Distance>
EnableIf<IsBidirectionalIterator<Iterator> && !IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x,Distance n)
{
    if(n>0)
    {
        for(;n>0;++x,--n){}
    }
    else
    {
        for(;n<0,--x,++n){}
    }
}
template<typename Iterator,typename Distance>
EnableIf<!IsBidirectionalIterator<Iterator>>
advanceIter(Iterator& x,Distance n)
{
    if(n<0)
    {
        throw "advanceIter():invalid iterator category for negative n";
    }
    while(n>0)
    {
        ++x;
        --n;
    }
}
```

就三种情况：

- 随机访问迭代器：随机访问，可前向可后向，常数时间
- 双向迭代器，不能随机访问：双向访问，线性时间
- 输入迭代器，不支持双向：普通情况，前向访问，线性时间

前面我们已经看到了iterator_category的定义， IsRandomAccessIterator是IsBidirectionalIterator的子类，所以在第三个advanceIter只判断非IsBidirectionalIterator就可以了，并不存在可随机访问但不能双向访问的迭代器。

这里可以看出来使用EnableIf的不足之处：需要为每一种情况添加一个判断，并且需要仔细检查条件是排他的。而使用标签派遣只需要添加一个使用`std::bidirectional_iterator_tag`标签的advanceIterImpl函数。

这两种方法各有适用情况，标签派遣支持基于层次结构的简单派遣，而EnableIf适用于由类型特性决定的任意属性集合的高级派遣。

### 20.3.2 Where Does the EnableIf Go?

EnableIf通常适用于函数模板的返回值，但是对于构造函数和转换函数就不适用了。另外，EnableIf会使得返回值很难辨认。因此，可以把EnableIf作为一个缺省模板参数。

```c++
template<typename Iterator>
constexpr bool IsInputIterator=IsConvertible<
				typename std::iterator_traits<Iterator>::iterator_category,
				std::input_iterator_tag>;

template<typename T>
class Container
{
    public:
    	template<typename Iterator, typename=EnableIf<IsInputIterator<Iterator>>>
    	Container(Iterator first,Iterator last);
    
    	template<typename U,typename = EnableIf<IsConvertible<T,U>>>
    	operator Container<U>() const; //转换函数
};
```

然而，如果增加其他重载构造函数，比如

```c++
template<typename Iterator,
		 typename = EnableIf<IsInputIterator<Iterator> &&
             		         !IsRandomAcessIterator<Iterator>>>
Container(Iterator first,Iterator last);

template<typename Iterator, 
		 typename=EnableIf<IsRandomAccessIterator<Iterator>>>
Container(Iterator first,Iterator last); //Error
```

这里的问题在于，对这两个重载构造函数，除了缺省参数之外是一样的，而缺省参数并不会用来判定两个模板是否一样，所以编译器会先报出重复定义的错误，这并不是在实例化模板函数时报的错，更不会引发SFINAE，因为还没到那一步。

解决这个问题的办法是给其中一个函数增加一个模板参数

```c++
template<typename Iterator,
		 typename = EnableIf<IsInputIterator<Iterator> &&
             		         !IsRandomAcessIterator<Iterator>>>
Container(Iterator first,Iterator last);

template<typename Iterator, 
		 typename=EnableIf<IsRandomAccessIterator<Iterator>>,
		 typename=int>
Container(Iterator first,Iterator last); 
```

于是这两个函数模板就有区别了，即使增加的这个参数并不会使用到，这样就可以通过SFINAE来获取合适的构造函数了。

### 20.3.3 Compile-Time `if`

必须指出，c++17提供的*constexpr if*使得在某些情况下可以不使用EnableIf。例如

```c++
template<typename Iterator,typename Distance>
void advanceIter(Iterator& x,Distance n)
{
    if constexpr(IsRandomAccessIterator<Iterator>)
    {
        x+=n;
    }
    else if constexpr(IsBidirectionalIterator<Iterator>)
    {
        if(n>0)
        {
            for(;n>0;++x,--n){}
        }
        else
        {
            for(;n<0;--x,++n){}
        }
    }
    else
    {
        if(n<0)
        {
            throw "advanceIter(): invalid iterator category for negative n";
        }
        while(n>0)
        {
            ++x;
            --n;
        }
    }
```

这种结构很清晰，然而需要能够在函数体中完整的列出各种条件并且能够表达出其差异，否则有可能产生一个空函数或者出现不明确的分支。

对于下面的情况，我们还是需要EnableIf的，注意EnableIf代表的是一种选择模板的方法。

- 需要包含不同的接口
- 需要不同的类定义
- 对于某些模板实参列表就不做对应的模板实例化操作。

对前两种情况，如果使用*constexpr if*，如前面说的，可能会产生空函数或者不明确的分支，因为可能无法完整的罗列出所有的情况及表达这些情况的差异。

对第三种情况，我们看个例子

```c++
template<typename T>
void f(T p)
{
    if constexpr(condition<T>::value)
    {
        
    }
    else
    {
        static_assert(condition<T>::value,"can't call f() for such a T");
    }
}
```

这里的问题在于不能配合SFINAE，`f()`总是存在的，从而可能会参与重载解析，而使用EnableIf就可以把这个函数完全移除掉。

### 20.3.4 Concepts

到目前为止介绍的方法都是有效的，但是总有些笨拙，占用了相当多的编译资源，并且，如果出错的话，调试信息会很难看。于是，将来可能会在语言中引入concepts。例如，前面的Container例子可能就会变成这样

```c++
template<typename T>
class Container
{
    template<typename Iterator>
    requires IsInputIterator<Iterator>
    Container(Iterator first,Iterator last);
    
    template<typename Iterator>
    requires IsRandomAccessIterator<Iterator>
    Container(Iterator first,Iterator last);
    
    template<typename U>
    requires IsConvertible<T,U>
    operator Container<U>() const;
    
    requires HasLess<T>
    void sort() {...};
};
```
除了EnableIf，concepts还可以取代标签派遣。不过目前concepts还只是个计划，就不多说了。

## 20.4 Class Specialization

函数模板重载和类模板特化是可以相提并论的提供可选项的两种机制。

```c++
template<typename Key,typename Value>
class Dictionary
{
    private:
    	vector<pair<Key const,Value>> data;
    pubilc:
        //原书上这里是value，显然是不对的。
    	Value& operator[](Key const& key)
        {
            for(auto& element :data)
            {
                if(element.first==key)
                    return element.second;
            }
            data.push_back(pair<Key const,Value>(key,Value()));
            return data.back().second;
        }
};
```

如果Key的类型支持`<`运算符，那么我们可以使用map来做容器。如果Key支持哈希运算，那么我们可以使用unordered_map来做容器。当然，上面只是个示例，所以不要纠结Value是否能够缺省构造，如果觉得不合适，可以自己用declval来处理。

### 20.4.1 Enabling/Disabling Class Templates

为了能够使用EnableIf来实现前面提到的改进，首先给上面的Dictionary增加一个缺省模板参数

```c++
template<typename Key,typename Value,typename=void>
class Dictionary
{
 //与上面相同
};
```

这个新的模板参数作为EnableIf的锚点。于是我们可以提供一个使用map的特化。

```c++
template<typename Key,typename Value>
class Dictionary<Key,Value,EnableIf<HasLess<Key>>>
{
	private:
		map<Key,Value> data;
	public:
		Value& operator[](Key const& key)
		{
			return data[key];
		}
		...
};
```

与函数重载不同，这里不需要屏蔽主模板，因为部分特化总是优先于主模板。如果我们添加一个部分特化，那么就需要显式的排除某些条件了

```c++
template<typename Key,typename Value,typename=void>
class Dictionary
{
 //与上面相同
};
template<typename Key,typename Value>
class Dictionary<Key,Value,EnableIf<HasLess<Key> && !HasHash<Key>>>
{
//同上
};
template<typename Key,typename Value>
class Dictionary<Key,Value,EnableIf<HasHash<Key>>>
{
	private:
		unordered_map<Key,Value> data;
	public:
		Value& operator[](Key const& key)
		{
			return data[key];
		}
};
```

注意HasLess和HasHash都分别只是一个返回bool值的特性，但并不需要与书上以前实现过的同名的特性相同。

这里仍然是通过SFINAE来剔除不匹配的特化，而不在于EnableIf具体得到的是啥，Enable如果条件为真返回的必然是void，而对于类模板来说，相同的参数列表则优先选择特化模板。如果条件为假，则引发SFINAE，剔除掉了对应的特化。

### 20.4.2 Tag Dispatching for Class Templates

```c++
//主模板声明，特意未做定义
template<typename Iterator,
		 typename Tag =
             BestMatchInSet<typename std::iterator_traits<Iterator>::iterator_category,
							std::input_iterator_tag,
							std::bidirectional_iterator_tag,
							std::random_access_iterator_tag>>
class Advance;

template<typename Iterator>
class Advance<Iterator,std::input_iterator_tag>
{
	public:
		using DifferenceType=typename std::iterator_traits<Iterator>::difference_type;
		void operator()(Iterator& x,DifferenceType n) const
		{
			while(n>0)
			{
				++x;
				--n;
			}
		}
};

template<typename Iterator>
class Advance<Iterator,std::bidirectional_iterator_tag>
{
	public:
		using DifferenceType=typename std::iterator_traits<Iterator>::difference_type;
		void operator()(Iterator& x,DifferenceType n) const
		{
			if(n>0)
			{
				while(n>0)
				{
					++x;
					--n;
				}
			}
			else
			{
				while(n<0)
				{
					--x;
					++n;
				}
			}
		}
};

template<typename Iterator>
class Advance<Iterator,std::random_access_iterator_tag>
{
	public:
		using DifferenceType=typename std::iterator_traits<Iterator>::difference_type;
		void operator()(Iterator& x,DifferenceType n) const
		{
			x+=n;
		}
};
```

函数对象类型通常是作为一个参数传入到算法或者说函数中。但不管怎样，如果是事先声明的一个变量，那么大体上会是`Advance<Iterator> adv`，如果是在调用表达式中生成的一个临时变量，那么大体上会是`Advance<Iterator>()`，这里并不会提供一个迭代器类型标签的参数，而是依赖编译器实例化时对第二个参数的计算，也就是对BestMatchInSet这个特性的估值，然后由于特化优先，从而选中了正确的模板来实例化。

于是问题就归结到了BestMatchInSet这个特性的编写上。

```c++
template<typename... Types>
struct MatchOverLoads;
template<>
struct MatchOverLoads<>
{
	static void match(...);
};
template<typename T1,typename... Rest>
struct MatchOverLoads<T1,Rest...>:public MatchOverLoads<Rest...>
{
	static T1 match(T1);
	using MatchOverLoads<Rest...>::match;  
};
template<typename T,typename... Types>
struct BestMatchInSetT
{
    using Type=decltype(MatchOverLoads<Types...>::match(declval<T>()));
};
template<typename T,typename... Types>
using BestMatchInSet=typename BestMatchInSetT<T,Types...>::Type;
```

我们来看看这里发生了什么，无关理解的就略去了

```c++
BestMatchInSet<typename std::iterator_traits<Iterator>::iterator_category,
							std::input_iterator_tag,
							std::bidirectional_iterator_tag,
							std::random_access_iterator_tag>>
=>BestMatchInSetT<typename std::iterator_traits<Iterator>::iterator_category,
							std::input_iterator_tag,
							std::bidirectional_iterator_tag,
							std::random_access_iterator_tag>>::Type
=>delctype(MatchOverLoads<std::input_iterator_tag,
						  std::bidirectional_iterator_tag,
					      std::random_access_iterator_tag
						 >::match(
                             declval<typename std::iterator_traits<Iterator>::iterator_category>()
                            )
          )
```

于是这里变成两个问题，一个是MatchOverLoads类，一个是match成员函数，本质上也就是选择了哪个match，从而获得对应的标签。

正如以前说过的，decltype和declval并不依赖于真实的调用或者说对表达式的计算，而是通过分析句法，得到一个值或者类型。因此，这里的match函数都只是声明而没有定义，并且由于是声明成static的，所以也不需要一个MatchOverLoads实例。

对于`MatchOverLoads<std::input_iterator_tag,std::bidirectional_iterator_tag,std::random_access_iterator_tag>`，从定义可以看到有个递归继承，类层次大体上如下

```c++
MatchOverloads<>
<=MatchOverloads<std::random_access_iterator_tag>
<=MatchOverloads<std::bidirectional_iterator_tag>
<=MatchOverloads<std::input_iterator_tag>
```

而`MatchOverloads<std::input_iterator_tag>`又通过using引入了其一系列基类中的match函数，于是其定义大体上可以看做如下，当然省略了相应的基类类名前缀

```c++
struct MatchOverLoads<std::input_iterator_tag>
{
	static std::input_iterator_tag match(std::input_iterator_tag);
	static std::random_access_iterator_tag match(std::random_access_iterator_tag);
	static std::bidirectional_iterator_tag match(std::bidirectional_iterator_tag);
	static void match(...);
};
```

于是，问题就转入了match的重载解析，这个问题就不需要再解释了。额外说一句，实际上MatchOverloads的继承顺序是没有关系的，这里的关键是通过using引入一系列的重载函数。

当然，上面代码至少有两个改进之处。一个是使得BestMatchInSetT是SFINAE友好的，最直接的方法就是判断`MatchOverLoads<Types...>::match(declval<T>())`的返回值类型，如果是void，那么就不声明Type，这个判断可以通过增加一个特化，也可以放在using前面做个模板。当然方法还是很多的，原则是没有匹配的时候就没有返回。另外一个是像下一节那样把返回值用一个Identity之类的类封装起来，因为有些类型是不存在像Type这样的返回类型的，比方说数组和函数对象。

总之，通过类模板来实现标签派遣，关键在于，一个是递归继承，一个是使用using引入一系列重载函数。而BestMatchInSetT则是把函数重载的结果封装为一个特性。

## 20.5 Instantiation-Safe Templates

可以使用EnableIf来实现所谓的实例化安全模板，也就是通过EnableIf检查一些条件，只有符合条件的参数才会用来实例化一个模板。比方说

```c++
#include <utility>
#include <type_traits>

template<typename T1,typename T2>
class HasLess
{
    template<typename T> struct Identity;
    template<typename U1,typename U2>
    static std::true_type test(Identity<decltype(std::declval<U1>() < std::declval<U2>())>*);
    template<typename U1,typename U2>
    static std::false_type test(...);
   public:
    static constexpr bool value=decltype(test<T1,T2>(nullptr))::value;

};

//注意这里的HasLess是一个非类型参数，不是上面定义的HasLess类模板
template<typename T1,typename T2,bool HasLess>
class LessResultImpl
{
    public:
        using Type=decltype( std::declval<T1>() < std::declval<T2>() );
};
template<typename T1,typename T2>
class LessResultImpl<T1,T2,false>
{
};

template<typename T1,typename T2>
class LessResultT:public LessResultImpl<T1,T2,HasLess<T1,T2>::value>
{
};
template<typename T1,typename T2>
using LessResult=typename LessResultT<T1,T2>::Type;

template<typename T>
class IsContextualBoolT
{
    template<typename U> struct Identity;
    template<typename U>
    static std::true_type test(Identity<decltype(std::declval<U>()?0:1)>*);
    template<typename U>
    static std::false_type test(...);
   public:
    static constexpr bool value=decltype(test<T>(nullptr))::value;
};
template<typename T>
constexpr bool IsContextualBool=IsContextualBoolT<T>::value;

struct X1{};
bool operator<(X1 const&,X1 const&) {return true;}
struct X2{};
bool operator<(X2 ,X2 ) {return true;}
struct X3{};
bool operator<(X3&,X3&) {return true;}
struct X4{};

struct BoolConvertible
{
    operator bool() const {return true;}
};
struct X5{};
BoolConvertible operator<(X5 const&,X5 const&)
{
    return BoolConvertible();
}

struct NotBoolConvertible
{
};
struct X6{};
NotBoolConvertible operator<(X6 const&,X6 const&)
{
    return NotBoolConvertible();
}
struct BoolLike
{
    explicit operator bool() const {return true;}
};
struct X7{};
BoolLike operator<(X7 const&,X7 const&){return BoolLike();}

template<typename T>
std::enable_if_t<IsContextualBool<LessResult<T const&,T const&>>,T const&>
min(T const& x,T const& y)
{
    if(y<x)
    {
        return y;
    }
    return x;
}

int main()
{
    min(X1(),X1());
    min(X2(),X2());
    min(X3(),X3());//编译错误
    min(X4(),X4());//编译错误
    min(X5(),X5());
    min(X6(),X6());//编译错误
    min(X7(),X7());
}             
```

这个程序出错如注释所示，但这几个不是在min当中出错，而是报找不到匹配的min函数的错误。因为对这三个参数的实例化触发了SFINAE，当然，如果我们另外提供了对应的重载函数那又是另外一回事。

我们逐个来看看。对于X4，没有定义在其上的`operator<`，对于X6，不能隐含的转换成bool，这两个错误很直接就不多说了。

对于X3，`min(T const&,T const&)`函数类型推断的结果T是X3，于是有

```c++
LessResult<X3 const&,X3 const&>
==>LessResultT<X3 const&,X3 const&>
==>LessResultImpl<X3 const&,X3 const&,HasLess<X3 const&,X3 const&>::value>
==>decltype(test<X3 const&>(nullptr))::value
==>decltype( std::declval<X3 const&>() < std::declval<X3 const&>() 
```

显然我们没有定义`operator<(X3 const&,X3 const&)`，于是引发SFINAE，最终没有实例化`min<X3&,X3&>()`这个函数。

只有为什么X2没有出错，其实也不为什么，重载的时候`f(T)`与`f(T const&)`是等价的。

接下来我们看IsContextualBool。

所谓的 contextually converted to bool指的是在控制流语句(if,while,for和do)的条件表达式、`!`、`&&`、`||`以及`?:`当中，可以隐式的转换一个本来是显式的向bool的转换。

具体地说，比如上面的X7，定义了显式的bool转换符，因此通常情况下必须使用bool转换符进行转换，而不能像X5那样可以隐式的转换，但是在上面提到的这些上下文当中，可以隐式的转换成bool值。因此，如果我们使用19.5介绍的IsConvertible，其中是通过`std::declval<F>()`来做的判断，显然这时候由于X7不能隐式转换，于是会引发错误。而在IsContextualBool中使用的是`std::declval<U>()?0:1)`，这时候是可以隐式转换的。

那么，为什么不使用控制流条件表达式或者逻辑运算符？因为控制流不属于SFINAE上下文，注意控制流的条件表达式不可能单独存在，而逻辑运算符有可能会被重载，只有`?:`既是一个表达式也不能重载，所以是个合适的选择。

总之，我们可以使用EnableIf来实现所谓的实例化安全，但是编写条件要小心，不要过分限制，也不要限制不足。

## 

