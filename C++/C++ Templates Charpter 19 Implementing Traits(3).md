# Charpter 19 Implementing Traits(3)

[TOC]

## 19.6 Detecting Member 

### 19.6.1 Detecting Member Types

```c++
template<typename...> using VoidT=void;
template<typename,typename=VoidT<>>
struct HasSizeTypeT:std::false_type
{
};
template<typename T>
struct HasSizeTypeT<T,VoidT<typename T::size_type>>:std::true_type
{
};
```

这里也没什么特别好解释的，如果T有个size_type的类型存在，那么就会优先选择部分特化版本，即使主模板也能匹配，这点是特别需要提醒的。这个部分特化继承了`std::true_type`，因此其value为true。如果T没有size_type类型，那么由于SFINAE，部分特化被剔除，只有主模板被选中。

另外就是如果这个size_type是个私有成员，那么也会选择主模板，因为HasSizeTypeT无法访问，因此，这个特性更准确的是判断能否访问某个类型。比方说

```c++
class CY
{   
    using size_type=std::size_t;
};
```

同时也可以看到，不论是using还是typedef都是有访问属性的。

需要提醒一下的是这里改成下面这样是不对的，因为无论如何，由于部分特化优先，总是会选择特化版本。

```c++
template<typename...> using VoidT=void;
template<typename T,typename=VoidT<typename T::size_type>>
struct HasSizeTypeT:std::true_type
{
};
template<typename T>
struct HasSizeTypeT<T,VoidT<>>:std::false_type
{
};
```
还有就是，这里直接使用了T，而不是`decltype(declval<T>())`，原因是declval返回的是T&&的一个对象，decltype因此也返回的是T&&类型，`T&&::size_type`是不合句法的。
#### Dealing with Reference Types

对于像HasSizeTypeT这样的特性，对于引用会有一些可能令人惊讶的结果。比方说

```c++
struct CXR
{
    using size_type=char&;
};

std::cout<<HasSizeTypeT<CXR>::value; //true
std::cout<<HasSizeTypeT<CX&>::value;  //false
std::cout<<HasSizeTypeT<CXR&>::value; //false
```

注意这里指的是作为模板参数的引用，而不是CXR里边对size_type定义出现的引用。

所以这里需要去除掉引用

```c++
template<typename T>
struct HasSizeTypeT<T,VoidT<typename RemoveReference<T>::size_type>>:std::true_type
{
};
```

#### Injected Class Names

```c++
struct size_type{};
struct Sizeable:size_type{};
static_assert(HasSizeTypeT<Sizeable>::value,"compiler bug:Injected class name missing");
```

这个static_assert是可以通过的，也就是说不会引发编译错误，程序运行也不会中止。因为这里size_type作为一个被继承的基类，本身是会被Sizeable引入作为一个成员类型的名字，因此`HasSizeTypeT<Sizeable>::value`会是true。而如果直接传入size_type，那么`HasSizeTypeT<size_type>::value`会是false，因为这时候size_type里边确实没有size_type这么个类型。

### 19.6.2 Detecting Arbitrary Member Types

实际上，如果我们需要检测任意的成员的类型，可以通过宏来实现，虽然标准委员会已经在考虑反射了。

```C++
#include <type_traits>
#include <iostream>
#include <vector>

#define DEFINE_HAS_TYPE(MemType)                  \
    template<typename,typename=std::void_t<>>     \
    struct HasTypeT_##MemType:std::false_type {}; \
    template<typename T>                          \
    struct HasTypeT_##MemType<T,std::void_t<typename T::MemType>> \
    :std::true_type {}

DEFINE_HAS_TYPE(value_type);
DEFINE_HAS_TYPE(char_type);

int main()
{
    std::cout<<"int::value_type: "<<HasTypeT_value_type<int>::value<<std::endl;
    std::cout<<"std::vector<int>::value_type: "<<HasTypeT_value_type<std::vector<int>>::value<<std::endl;
    std::cout<<"std::iostream::value_type: "<<HasTypeT_value_type<std::iostream>::value<<std::endl;
    std::cout<<"std::iostream::char_type "<<HasTypeT_char_type<std::iostream>::value<<std::endl;
}
```

运行结果是

```bash
int::value_type: 0
std::vector<int>::value_type: 1
std::iostream::value_type: 0
std::iostream::char_type 1
```

### 19.6.3 Detecting Nontype Members

还可以检测数据成员和成员函数

```C++
#include <type_traits>
#include <iostream>
#include <vector>

#define DEFINE_HAS_MEMBER(Member)                                   \
    template<typename,typename=std::void_t<>>                       \
    struct HasMemberT_##Member:std::false_type {};                  \
    template<typename T>                                            \
    struct HasMemberT_##Member<T,std::void_t<decltype(&T::Member)>>  \
    :std::true_type {};

DEFINE_HAS_MEMBER(size);
DEFINE_HAS_MEMBER(first);

int main()
{
    std::cout<<"int::size: "<<HasMemberT_size<int>::value<<std::endl;
    std::cout<<"std::vector<int>::size: "
        <<HasMemberT_size<std::vector<int>>::value<<std::endl;
    std::cout<<"std::pair<int,int>::first: "
        <<HasMemberT_first<std::pair<int,int>>::value<<std::endl;
}
```

运行结果是

```bash
int::size: 0
std::vector<int>::size: 1
std::pair<int,int>::first: 1
```

首先解释下`&T::Member`，虽然这种表达式被称为指向成员的指针，但这个时候不存在取Member变量地址的事情，这只是表示Member在T的定义中的偏移量，或者说，可以据此判断T中有没有Member这个变量的声明。如果没有&，那么就表示Member是存在于T中的一个类型标识符，就像前面的例子那样。

要使`&T::Member`合法，需要满足一些条件，注意我们指的是不引发SFINAE。

- Member必须是T的一个没有任何歧义的成员的标识符，也就是说不能有同名的重载成员函数，或者由于多重继承而来的重名的成员。
- 这个成员必须可访问。
- 这个成员必须是个非类型，非枚举的成员，否则&前缀就是非法的。
- 如果T::Member是个静态数据成员，那么Member本身类型的operator&不能使得&T::Member失效，比方说定义这个&运算符为private的，

原书上说可以很容易修改部分特化，以排除`&T::Member`不是指向成员的指针类型的情况，换句话就是排除Member是个静态变量的情况。并且，也可以通过排除或者限定指向成员函数的指针来限制这个特性只判断数据成员或者成员函数存在。但是，我现在并不知道应该怎么做，就先略过吧。

#### Detecting Member Functions

上面的HasMember特性无法判断多个同名的成员，比方说重载函数，不过，我们可以通过对函数的调用，来引发SFINAE，从而可以判断某个名字的函数是否存在。

```c++
template<typename,typename=std::void_t<>>
struct HasBeginT:std::false_type
{
};
template<typename T>
struct HasBeginT<T,std::void_t<decltype(std::declval<T>().begin())>>
:std::true_type 
{
};
```

因此，如果T有个`begin()`函数，不管有几个吧，必然是可以调用成功的，也就是选择了部分特化，从而返回true_type。在这里只要能调用就可以了，其返回值是什么并不重要。而且，对于decltype来说，并不需要完成其参数中的调用过程才能得到其返回类型。但是，如果使用`decltype(std::declval<T>().begin(),0)`这样的表达式，那么其中对begin的调用是需要完成的，这时候decltype运算的结果并不是begin的返回值，而是里边的逗号表达式的结果，也就是一个int。

#### Detecting Other Expressions

~~~c++
template<typename,typename,typename=std::void_t<>>
struct HasLessT:std::false_type
{
};
template<typename T1,typename T2>
struct HasLessT<T1,T2,std::void_t<decltype(std::declval<T1>()<std::declval<T2>())>>
:std::true_type
{
};
~~~

代码很直白，就不多说了，不过可以再次看到几位作者的不同的编码风格。但是必须要说的是，HasLessT需要提供T1和T2，所以书上的那个例子是不对的。

还可以组合判断

```c++
template<typename,typename=std::void_t<>>
struct HasVariousT:std::false_type
{
};
template<typename T>
struct HasVariousT<T,std::void_t<decltype(std::declval<T>().begin()),typename T::difference_type,typename T::iterator>>
:std::true_type
{
};
```

### 19.6.4 Using Generic Lambdas to Detect Members

除了宏之外，还可以使用泛型lambda表达式来检测成员，也就是通过前面介绍的isValid来生成一个判断。

```c++
#include <iostream>
#include <utility>

template<typename F,typename... Args,
    typename=decltype(std::declval<F>()(std::declval<Args&&>()...))>
std::true_type isValidImpl(void*);

template</*typename F,*/typename... Args>
std::false_type isValidImpl(...);

inline constexpr
auto isValid=[](auto f){
  return [](auto&&... args){
    return decltype(isValidImpl<decltype(f),decltype(args)&&...>(nullptr)){};
  };
};

constexpr auto hasFirst
    =isValid([](auto&& x) -> decltype((void)&x.first) {} );
template<typename T>
using HasFirstT=decltype(hasFirst(std::declval<T>()));

constexpr auto hasSizeType
    =isValid([](auto&& x) -> typename std::decay_t<decltype(x)>::size_type {} );
template<typename T>
using HasSizeTypeT=decltype(hasSizeType(std::declval<T>()));

constexpr auto hasLess
    =isValid([](auto&& x,auto&& y) -> decltype(x<y) {} );
template<typename T1,typename T2>
using HasLessT=decltype(hasLess(std::declval<T1>(),std::declval<T2>()));

int main()
{
    using namespace std;
    cout<<boolalpha;
    
    cout<<"hasFirst: "<<HasFirstT<pair<int,int>>::value<<"\n"; //true                                                                                                                            
    
    struct CX
    {
        using size_type=std::size_t;
    };
    cout<<"hasSize: "<<HasSizeTypeT<CX>::value<<'\n';  //true
    cout<<"hasSize: "<<HasSizeTypeT<int>::value<<'\n';  //false                                                                                                                                
    
    cout<<HasLessT<int,char>::value<<endl; //true
    cout<<HasLessT<string,string>::value<<endl; //true
    cout<<HasLessT<string,int>::value<<endl; //false
    cout<<HasLessT<string,char*>::value<<endl;  //true      
}
```

这段代码的运行结果与注释一致，并且原书上说到的使用`std::decay`以移除引用指的也是这段代码。细节之前都说过了，不过我们可以加深下对isValid的理解，即需要判断的内容，是作为其参数的那个lambda表达式的返回值表达式的一部分。

然而，书上的第一段代码，也就是下面这段代码，对于hasLess的结果与注释并不一致。

```c++
int main()
{
    using namespace std;
    cout<<boolalpha;
    
    constexpr auto hasFirst
        =isValid([](auto x) -> decltype((void)valueT(x).first) {} );
    cout<<"hasFirst: "<<hasFirst(type<pair<int,int>>)<<"\n"; //true
    
    constexpr auto hasSizeType
        =isValid([](auto x) -> typename std::decay_t<decltype(valueT(x))>::size_type {} );
    struct CX
    {
        using size_type=std::size_t;
    };
    cout<<"hasSize: "<<hasSizeType(type<CX>)<<'\n';  //true
    
    if constexpr(!hasSizeType(type<int>))
    {
        cout<<"int has no size_type\n";
    }
    
    constexpr auto hasLess
        =isValid([](auto x,auto y) -> decltype(valueT(x) < valueT(y)) {} );
    cout<<hasLess(42,type<char>)<<endl; //true
    cout<<hasLess(type<string>,type<string>)<<endl; //true
    cout<<hasLess(type<string>,type<int>)<<endl; //false
    cout<<hasLess(type<string>,"hello")<<endl;  //true
}
```

上面对于hasLess的结果实际上是这样的

```bash
false
true
false
false
```

甚至对于`hasLess1(42,34)`结果也是false。但是对于

```c++
cout<<hasLess(type<int>,type<char>)<<endl; //true
cout<<hasLess(type<int>,type<double>)<<endl; //true
cout<<hasLess(type<string>,type<char*>)<<endl;  //true
```

结果便如注释所示。所以这里问题所在可以从`[](auto x,auto y) -> decltype(valueT(x) < valueT(y)) {} `找到。具体地说，通过之前的说明，我们知道，hasLess的参数就是上面这个lambda表达式的参数`(auto x,auto y)`，而`valueT(x)`的参数`x`是一个`type<T>`的对象，所以如果hasLess的参数不是`type<T>`对象，那么在匹配isValidImpl时就会触发SFINAE从而选择后备模板。

## 19.7 Other Traits Techniques

### 19.7.1 If-Then-Else

首先回顾一下前面19.4.4的一个例子

```c++
template<typename,typename,typename=std::void_t<>>
struct HasPlusT:std::false_type
{
};
template<typename T1,typename T2>
struct HasPlusT<T1,T2,std::void_t<decltype(std::declval<T1>() + std::declval<T2>())>>
:std::true_type
{
};
template<typename T1,typename T2,bool=HasPlusT<T1,T2>::value>
struct PlusResultT
{
    using Type=decltype(std::declval<T1>()+std::declval<T2>()));
};
template<typename T1,typename T2>
struct PlusResultT<T1,T2,false>
{
};
```

这里只是作为引入下面结构的一个参照。

```c++
template<bool COND,typename TrueType,typename FalseType>
struct IfThenElseT
{
    using Type=TrueType;
};
template<typename TrueType,typename FalseType>
struct IfThenElseT<false,TrueType,FalseType>
{
	using Type=FalseType;
};
template<bool COND,typename TrueType,typename FalseType>
using IfThenElse=typename IfThenElseT<COND,TrueType,FalseType>::Type;
```

特别提醒下，如果在类定义体中使用bool值，那将是个运行期的行为，而在模板参数中使用bool值，是编译期的行为。

我们可以这样来使用IfThenElseT

```c++
template<auto N>
struct SmallestIntT
{
    using Type= 
        typename IfThenElseT<N<=std::numeric_limits<char>::max(),char,   //if() {char}  
        typename IfThenElseT<N<=std::numeirc_limits<short>::max(),short, //else if() {short}	
    	typename IfThenElseT<N<=std::numeric_limits<int>::max(),int,     //else if() {int}
    	typename IfThenElseT<N<=std::numeric_limits<long>::max(),long,	 //else if() {long}
    	typename IfThenElseT<N<=std::numeric_limits<long long>::max(),long long, //else if() {long long}
            	void		//else {void}
                >::Type>::Type>::Type>::Type>::Type>;  
};
```

这里把嵌套关系搞清楚，另外要了解这里由于要获取Type，编译器会在选择分支之前做出一系列估值。比方说对于某个`std::numeirc_limits<short>::max()` < `N` <= `std::numeric_limits<int>::max()`，那么这时候Type大体会如下进行推断：

```c++
using Type=typename IfThenElseT<false,char,
			typename IfThenElseT<false,short,
			typename IfThenElseT<true,int,
			typename IfThenElseT<true,long,
			typename IfThenElseT<true,long long,void>::Type>::Type>::Type>::Type>::Type;
==>using Type=typename IfThenElseT<false,char,
				typename IfThenElseT<false,short,
				typename IfThenElseT<true,int,
				typename IfThenElseT<true,long,long long>::Type>::Type>::Type>::Type;
==>using Type=typename IfThenElseT<false,char,
				typename IfThenElseT<false,short,
				typename IfThenElseT<true,int,long>::Type>::Type>::Type;
==>using Type=typename IfThenElseT<false,char,
				typename IfThenElseT<false,short,int>::Type>::Type;
==>using Type=typename IfThenElseT<false,char,int>::Type;
==>using Type=int;				
```

注意这里的分支选择，是通过选择不同的模板来实现的，并因而得到不同的Type值。

有时候需要把一个有符号数转换成无符号数，`std::make_unsigned`的问题是，传入参数的必须是一个有符号整型数，并且也不支持bool类型，否则行为就是未定义的。

为了处理bool型参数，下面的方法是不正确的

```c++
template<typename T>
struct UnsignedT
{
    using Type=IfThenElse<std::is_integral<T>::value && !std::is_same<T,bool>::value,
    						typename std::make_unsigned<T>::type,T>;
};
```

不正确的原因在于对于`UnsignedT<bool>`，由于模板参数中要获取type，所以无论如何都会处理`make_unsigend<bool>`，从而引发未定义的行为。注意这里是IfThenElse，不是IfThenElseT。当然换成后者也是一样的，这里关键是要获取type。

为此,我们需要这样

```c++
template<typename T>
struct IdentityT
{
    using Type=T;
};
template<typename T>
struct MakeUnsignedT
{
    using Type=typename std::make_unsigned<T>::type;
};
template<typename T>
struct UnsignedT
{
   using Type=typename IfThenElse<std::is_integral<T>::value && !std::is_same<T,bool>::value,
    			MakeUnsignedT<T>,IdentityT<T>>::Type;
};
```

这里的关键在于Type的位置，虽然模板参数都会进行替换，但是也只是做了替换，只有在解析Type时才会真正去做估值，也就是只有在这个时候，才会根据第一个参数的值去选择一个模板进行实例化，从而避开了`make_unsigned<bool>`引发的问题。或者说，在解析Type之前，模板参数里的两个模板都只是做了部分实例化，也就是只对其参数做了替换，因为没有使用其定义中的内容，编译器就不管了，替换完之后，开始对第一个参数进行估值，然后选择一个模板生成实例。如果在模板参数中使用了Type，那么就会对其进行完全实例化，从而引发前面说的无定义行为。因此，再**强调一下，这里的关键是Type的位置**。

至于这里的句法，可以回头看下IfThenElse和IfThenElseT的定义，就知道为什么要在IdentityT和MakeUnsignedT各自声明一个Type了。具体地说，`IfThenElseT<COND,TrueType,FalseType>::Type`就是TrueType或者FalseType本身，对于`UnsignedT<bool>`，会有
```c++
using Type=typename IfThenElse<false,MakeUnsignedT<bool>,IdentityT<bool>>::Type;
==>using Type=typename IfThenElseT<false,MakeUnsignedT<bool>,IdentityT<bool>>::Type::Type;
==>using Type=typename IdentityT<bool>::Type;
==>using Type=bool;
```

对应的c++标准库有`std::conditional<>`，因此可以这样使用

```c++
template<typename T>
struct UnsignedT
{
    using Type=
        typename std::conditional_t<std::is_integral<T>::value && !std::is_same<T,bool>::value,
    	MakeUnsignedT<T>,IdentityT<T>>::Type;
};
```

### 19.7.2 Detecting Nonthrowing Operations

```c++
template<typename T1,typename T2>
class Pair
{
    T1 first;
    T2 second;
   public:
    Pair(Pair&& other):first(std::forward<T1>(other.first)),second(std::forward<T2>(other.second))
    {}
};
```

Pair的移动构造函数中，如果T1或者T2的移动操作抛出异常，那么这个移动构造函数就会抛出异常。如果我们不希望移动构造函数抛出异常，那么

```c++
Pair(Pair&& other) noexcept(IsNothrowMoveConstructibleT<T1>::value &&
                           	IsNothrowMoveConstructibleT<T2>::value)
:first(std::forward<T1>(other.first)),second(std::forward<T2>(other.second))
{}
```

先看一个非SFINAE友好的IsNothrowMoveConstructibleT实现

```c++
template<typename T>
struct IsNothrowMoveConstructibleT:std::bool_constant<noexcept(T(std::declval<T>()))>
{
};
```

在C++17前，需要使用`std::integral_constant<bool,…>`来代替`std::bool_constant<…>`，注意这里的noexcept是个运算符，用来在编译阶段检查一个表达式是否不抛出异常，并且直接返回bool值，因此可以作为bool_constant的参数来构造一个true_type或者false_type。这里的问题在于，如果T没有移动或者赋值构造函数，那么`template<typename T>`就会引发编译错误，而不是SFINAE。

如前面($19.4.4)说过的，我们需要先检查表达式是否合法然后再去计算其值。跟以前一样，先在模板参数中检查表达式的合法性，如果合法，选择部分特化，然后计算表达式的值，如果不合法，则选择后备模板，不再计算。

```c++
template<typename T,typename = std::void_t<>>
struct IsNothrowMoveConstructibleT:std::false_type
{
};
template<typename T>
struct IsNothrowMoveConstructibleT<T,std::void_t<decltype(T(std::declval<T>()))>>
:std::bool_const<noexcept(T(std::declval<T>()))>
{
};
```

这里另外要注意的是，只能通过直接调用一个移动构造函数来检查其是否抛出异常，也就是说，除了这个移动构造函数必须是public并且没有被删除外，其对应类型必须不能是抽象类成员，不过对抽象类的指针或者引用是可以的，比方说上面的T不能是个抽象类，但是如果T是个抽象类的引用或者指针是可以的。因此这个特性的名字是Is开头而不是Has开头，表明只是检测某个对象是否可以而不是是否拥有，虽然对某些鸭子类型语言这两者没什么区别，但在C++是有区别的。

对应的标准库有`std::is_move_constructible<>`。

### 19.7.3 Traits Convinience

对于特性来说，比较令人烦躁的是许多的Type和typename，比如

```c++
template<typename T1,typename T2>
Array<typename RemoveCVT<typename RemoveReferenceT<typename PlusResultT<T1,T2>::Type>::Type>::Type>
operator+ (Array<T1> const&,Array<T2> const&);
```

#### Alias Templates and Traits

不过，我们可以使用别名

```c++
template<typename T>
using RemoveCV=typename RemoveCVT<T>::Type;
template<typename T>
using RemoveReference=typename RemoveReferenceT<T>::Type;
template<typename T1,typename T2>
using PlusResult=typename PlusResultT<T1,T2>::Type;

//注意原书上这里是PlusResultT,显然是不对的
template<typename T1,typename T2>
Array<RemoveCV<RemoveReference<PlusResult<T1,T2>>>>  
operator+(Array<T1> const&,Array<T2> const&);
```

别名也有一些不足：

1. 不能特化。
2. 一些特性其本意就是由使用者进行特化的，比方说描述一个操作是否可交换的特性，当这样的特性与别名一起使用时会对类模板的特化产生一些迷惑。这一条且看。
3. 别名模板总是会对其原初模板进行实例化，这可能会引发一些问题。

建议是，别名模板尽可能的与原模板形式上一样，除了名字略有区别外。比方说到目前为止所看到的，原模板总是带T，而别名模板则把T去掉，除此之外，模板参数保持一致，包括顺序和个数，以及类型成员用Type。而标准库在命名规范上与此不同，原模板没有后缀，别名模板则加上`_t`的后缀，以及类型成员用type。

#### Variable Template and Traits

特性返回一个`::value`或者别的什么类似名字的时候，就表示需要生成这个特性的结果。对这种情况，使用constexpr变量模板提供了一种表示方式

```c++
template<typename T1,typename T2>
constexpr bool IsSame=IsSameT<T1,T2>::value;
template<typename From,typename To>
constexpr bool IsConvertible=IsConvertibleT<From,To>::value;
```

同样c++标准的命名规范则是原模板没有后缀，对应的变量模板则加上`_v`。

