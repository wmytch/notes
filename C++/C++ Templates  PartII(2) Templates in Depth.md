# C++ Templates  PartII(2):Templates in Depth

## Chapter 14 Instantiation

### 14.1 On-Demand Instantiation

```c++
template<typename T> class C;  //#1 只是声明
C<int>* p=0;  //#2 没问题，不需要知道C<int>的定义
template<typename T>
class C{
  public:
  	void f(); //#3 成员声明
};            //#4 类模板定义结束
void g(C<int>& c)  //#5 只需要类模板声明
{
  c.f();     //#6 使用类模板定义，在这个翻译单元，需要能找到C::f()的定义
}
template<typename T>
void C<T>::f()  //#6所需要的定义
{
  
}
```

- 对于#1，也称为前向声明
- 对于指针和引用，我们也不需要一个类型的完整定义，这对普通类型、模板都是一样的，如#2，#5
- 如果需要知道一个模板特化的大小或者需要访问其成员，那么这个模板的定义就必须可见，如#6

总之，就是需要啥再去做啥。

```c++
template<typename T>
class C{
  public:
  	C(int);
};
void candidate(C<double>); //#1
void candidate(int);  //#2
int main()
{
  candidate(42);
}
```

虽然编译器会选择#2这个函数，但是编译器很有可能在解决重载时为了确定#1函数是否合用，而实例化一个`C<double>`，因为在其定义中有一个隐含的类型转换，只不过在有确切匹配的情况下不会选择一个隐含类型转换。

### 14.2 Lazy Instantiation

#### 14.2.1 Partial and Full Instantiation

```c++
template<typename T> T f(T p){ return 2*p;}
decltype(f(2)) x=2;

template<typename T>class Q{
  using Type=typename T::Type;
};
Q<int>* p=0;

template<typename T> T v=T::default_value();
decltype(v<int>) s;
```

这三个都是部分实例化的例子，注意不是部分特化，两个不同的概念。

反正编译器就这么处理了，不需要的就不去管它。可以与前面说过的需要啥再去做啥联系起来看。

#### 14.2.2 Instantiated Components

当一个类模板实例化时，每一个成员的声明都会实例化，而其定义不会，匿名union和虚函数除外。

```C++
template<typename T>
class safe{
  
};

template<int N>
class Danger{
  int arr[N];
};

template<typename T, int N>
class Tricky{
  public:
  	void noBodyHere(Safe<T> =3); //OK,除非使用缺省参数引发错误，注意空格的使用
  															//这里的问题在于是否允许一个int来初始化一个Safe<T>
  															//而这需要将来在调用函数时看Safe<T>的具体定义
  															//到目前为止，并不需要考虑这个问题，所以是没问题的
  	void inclass(){
      Danger<N> noBoomYet;  //OK,除非使用时N<=0;
    }
  	struct Nested{
      Danger<N> pfew;  //OK,除非使用时N<=0;
    };
  	union {
      Danger<N> anonymous;  //OK,除非实例化Tricky时N<=0
      int align;
    };
  	void unsafe(T (*p)[N]);  //OK,除非实例化Tricky时N<=0
  	void error(){
      Danger<-1> boom;   //肯定是错的
    }
};
```

如果定义`Tricky<int,-1> inst;`，在目前版本gcc，会在`unsafe`的声明和`boom`的定义处引发编译错误，实际上，即使不对Tricky做实例化，`boom`也是通不过编译的，而匿名union定义的地方并没有引发错误，至于为什么大概还是编译器的处理方式吧，`inclass`和`Nested`没有引发错误，但是如果在什么地方使用或者调用了他们的时候，就会出错了。

```c++
template<typename T>
class VirtualClass{
  public:
  	virtual ~VirtualClass(){};
  	virtual T vmem();  //#1
};
int main()
{
  VirtualClass<int> inst; //2
}
```

对于虚函数vmem，这里编译是没有问题的，但是如果使用inst这个实例的话，比方说#2改成这样`{VirtualClass<int> inst;}`，那么就会出错了，除非加上vmem的定义。

### 14.3 The C++ Instantiation Model

#### 14.3.1 Two-Phase Lookup

1. 第一个阶段，解析模板阶段

   - 对于非依赖性名字，使用常规的查找规则，以及，如果需要的话，同时使用参数依赖的查找规则(ADL)。
   - 对于无限定的依赖性名字，指带有依赖性参数的函数调用，使用常规查找规则，但这时的查找结果是不完整的，还需要在第二个阶段，也就是模板实例化阶段再做额外的ADL查找
2. 第二个阶段，也就是在POI实例化一个模板阶段
   - 查找限定的依赖性名字，包括对模板参数的替换
   - 对无限定的依赖性名字，做额外的ADL查找

对于无限定的依赖性名字，初次常规查找用来决定其是否模板。

```c++
namespace N {
  template<typename> void g(){}
  enum E{ e };
}
template<typename> void f(){}
template<typename T> void h(T p)
{
  f<int>(p); //#1
  g<int>(p); //#2 错误
}
int main()
{
  h(N::e);
}
```

- 对于#1，当看到名字`f`跟着一个`<`，编译器就要决定这个`<`符号是一个尖括号还是一个小于符号。这种情况下，常规查找可以看到`f`的声明，这是个模板，于是可以决定这是个尖括号，解析继续。
- 对于#2，因为常规查找找不到`g`，于是`<`被认为是个小于号，于是出错。
- 但是，如果略过这个问题，接下来实例化`h`，在使用ADL来查找`T=N::e`时，又可以看到`N::g`，但是编译器并不能看得那么远，在#2找不到`g`就会报错。

#### 14.3.2 Points of Instantiation

POI是源码中替换模板或者说一个模板特化可以插入的点，在这个点，编译器必须能够访问模板实体的定义或者声明以生成这个特化。

```c++
class MyInt{
  public:
  	MyInt(int i);
};

MyInt operator - (MyInt const&);
bool operator > (MyInt const&,MyInt const&);
using Int=MyInt;
template<typename T>
void f(T i)
{
  if(i>0)
  {
    g(-i);
  }
}
//#1
void g(Int)
{
  //#2
  f<Int>(42);  //调用点
  //#3
}
//#4
```

当编译器看到`f<Int>(42)`时，知道模板`f`需要用`MyInt`来替换`T`，从而生成`f`的一个实例，这时候就需要创建一个POI，也就是要插入这么一个特化函数：
```c++
  template<>
  void f<Int>(Int i)
  {
    if(i>0)
    {
      g(-i);
    }
  }
```

- \#2和\#3不能是POI，因为就C++本身句法而言，在这里不能插入`::f<Int>(Int)`的定义
- 对于\#1，因为这里`g(Int)`不可见，无法解析`g(-i)`，所以不能是POI。在这里不论是常规查找，还是ADL都是如此，因为查找总是往前找的。
- 因此只有\#4才是合适的POI

那么，为什么要用MyInt而不是int。我们从头开始看这个问题，假如使用的是int
- 当碰到f的定义时，可以注意到无限定的名字g是有依赖性的，因为其参数依赖于模板参数T，于是对g进行了常规查找，但是在这个点，g是不可见的，也就是说找不到的。
- 之后继续编译，创建了POI之后，在POI又做了一次查找，但是这次是ADL，是依据相关联的类和名字空间进行的，然而int并没有关联的类和名字空间，所以找不到g，或者说就没有去找。如果用MyInt，则会引发ADL，于是从头开始查找，就能够找到g了。
- 因此，编译器就会提示找不到g的错误。
- 当然，错误发生点是在模板`f`的定义当中，而不是在POI里边，如果在f前面加个g的前向声明，是可以解决这个问题的。

对于类而言，POI的选择又是另外一种情况。


```c++
template<typename T>
class S{
  public:
  	T m;
};
//#1
unsigned long h()
{
  //#2
  return (unsigned long)sizeof(S<int>);
  //#3
}
//#4
```

- 同样，\#2和\#3不能是POI
- 如果POI在\#4,那么`sizeof(S<int>)`是非法的，因为在这里看不到`S<int>`的定义，而类是不做ADL的。
- 因此，只能是在\#1

当实例化一个模板时，有可能会引发额外的实例化。

```c++
template<typename T>
class S{
  public:
  	using I=int;
};

//#1
template<typename T>
void f()
{
  S<char>::I var1=41;  //注意，这里没有用typename
  typename S<T>::I var2=42; //注意，这里必须用typename
}
int main()
{
  f<double>();
}
//#2:#2a,#2b
```

- 如前面讨论的，`f<double>()`的POI在\#2
- 特化`S<char>`的POI在\#1
- `S<T>`在这一点还不能实例化
- 当在\#2实例化`f<double>()`时，编译器发现需要实例化`S<double>`
- 对于函数模板，次级POI与主POI是一样的，也就是说没有主次之分，只要能找得到
- 对于类模板，次级POI则需要放置在主POI的前面
- 就本例而言，`f<double>()`放在\#2b，而`S<double>`则要放在\#2a

一个翻译单元通常包含一个实例的多个POI。对于类模板实例，只保留第一个，其他的都会被忽略。对于函数和变量模板实例，所有的POI都会保留。这些同样的POI都必须是一致的，但是编译器并不做检查。

## Chapter 15 Template Argument Deduction

### 15.1 The Deduction Process

```c++
template<typename T> void f(T); //模板参数是T
template<typename T> void g(T&); //模板参数也是T

double arr[20];
int const seven=7;

f(arr);  //非引用参数：T是double*
g(arr);  //引用参数：T是double[20]
f(seven); //非引用参数：T是int，忽略const和volatile
g(seven); //引用参数：T是int const
f(7);  //非引用参数：T是int
g(7);  //引用参数：T是int，然而会引发错误：不能给引用传送字面量 
```

### 15.2 Deduced Contexts

~~~c++
template<typename T>
void f1(T*);

template<typename E,int N>
void f2(E(&)[N]);

template<typename T1,typename T2,typename T3>
void f3(T1 (T2::*)(T3*));

class S{
  public:
  	void f(double *);
};

void g(int*** ppp)
{
  bool b[42];
  f1(ppp);    //T=int**
  f2(b);      //E=bool N=42
  f3(&S::f);  //T1=void T2=S T3=double
}
~~~

复杂的类型声明是由基本类型构造而成，于是匹配过程从最高层递归的解析组成元素。这就叫推断上下文。

然而有一些结构不是推断上下文，例如

- 有限定的类型名称，比如`Q<T>::X`就不会被用来推断T
- 非类型表达式，比方说`S<I+1>`就不会被用来推断`I`，同样，`int(&)[sizeof(S<T>)]`也不会用来推断T

上面的意思是，不会通过解析这些结构来推断模板参数。但是，这并不表明非推断上下文会阻止对参数的推断过程

```c++
template<int N>
class X{
  public:
  	using I=int;
  	void f(int){
      
    }
};

template<int N>
void fppm(void (X<N>::*p)(typename X<N>::I));

int main()
{
  fppm(&X<33>::f);  //没问题，N推断为33
}
```

在函数模板fppm中，`X<N>::I`不是推断上下文，然而`X<N>::*p`中的`X<N>`是，可以用来推断N。

### 15.3 Special Deduction Situations

```c++
template<typename T>
void f(T,T);

void (*pf)(char,char)=&f; //A void(char,char)  P void(T,T)
```
```C++
class S{
  public:
  	template<typename T>operator T&();  //类型转换函数，把S转换成T
};
void f(int (&)[20]);
void g(S s)
{
  f(s);  //把s转换成int (&)[20],因此对于S::T&来说，A是int[20],P是T
}
```

### 15.4 Initializer Lists

当函数参数是个初始化列表时，这个参数没有特定的类型，所以通常不会做推断

```c++
#include <initializer_list>
template<typename T> void f(T p);
int main()
{
  f({1,2,3});  //错误，无法推断T
}
```

所以

```c++
template <typename T> void f(std::initializer_list<T>);

int main()
{
  f({2,3,5,7,9});  //OK,T推断为int
  f({'a','e','i','o','u',42});  //错误，T是char还是int？
}
```

换句话说就是，对于参数类型P，移除引用和顶层的const和volatile，如果等价于`std::initializer_list<P’>`，而类型`P’`具有可推断模式，那么推断是可以进行的，只要初始化列表中的元素都是同样的类型。

同样的，如果P是一个指向数组的引用，而这个数组元素类型P’是可推断的，则也是可以通过与初始化列表中的元素逐个比较来进行推断。更进一步说，如果其边界可推断，比方说提供了一个非类型参数来表示这个边界，那么也可以从列表元素的个数推断出这个边界来，也就是可以推断出这个非类型参数的值来。

### 15.5 Parameter Packs

```c++
template<typename First,typename... Rest>
void f(First first,Rest... rest);
void g(int i,double j,int* k)
{
  f(i,j,k); //First=int, Rest={double,int*}
}
```

```c++
template<typename T,typename U> class pair{};
template<typename T,typename... Rest>
void h1(pair<T,Rest> const&...);
template<typename... Ts,typename... Rest>
void h2(pair<Ts,Rest> const&...);

void foo(pair<int,float>pif,pair<int,double>pid,pair<double,double>pdd)
{
  h1(pif,pid);  //OK,T=int,Rest={float,double}
  h2(pif,pid);  //OK,Ts={int,int} Rest={float,double}
  h1(pif,pdd);  //错误，T是int?还是double？
  h2(pif,pdd);  //OK,Ts={int,double} Rest={float,double}
}
```

注意，上面的`{}`表示一个集合，而不是列表，不可能存在把两个不同参数集中的类型合并成一个列表的事情。就可变参数函数h1和h2来说，其每一个参数都是一个pair模板的实例，每一个模板实例都有各自的T/Ts和Rest，但是对于h1，每一个pair实例的T都必须是一样的，而对h2来说，其Ts/Rest是可变的，两个参数集不需要相同，模板参数只是个占位符。

```c++
template<typename... Types> class Tuple{};

template<typename... Types>
bool f1(Tuple<Types...>,Tuple<Types...>);

template<typename... Types1,typename... Types2>
bool f2(Tuple<Types1...>,Tuple<Types2...>);

void bar(Tuple<short,int,long>sv，
					Tuple<unsigned short,unsigned,unsigned long>uv)
{
  f1(sv,sv); //OK, Types={short,int,long}
  f2(sv,sv); //OK,Types1和Types2都是{short,int,long}
  f1(sv,uv); //ERROR,无法推断Types，因为两个参数不同
  f2(sv,uv); //OK,
}
```

这里的`{}`可以当成列表，也可以看成集合，在这里没有区别。

#### 15.5.1 Literal Operator Templates

```c++
template<char... cs>
int operator"" _B7()  //注意句法
{
    std::array<char,sizeof...(cs)> chars{cs...};
    for (char c:chars){
        std::cout<<"'"<<c<<"'";

    }
    std::cout<<std::endl;
    return sizeof...(cs);
}
int main()
{
    int a=121.5_B7;
    std::cout<<a<<std::endl;
    auto b=01.3_B7;  //ok,<'0','1','.','3'>
    auto c=0XFF00_B7; //ok,<'0','X','F','F','0','0'>
  	auto d=0815_B7;   //error,8不是合法的8进制字面量
  	auto e=hello_B7;  //error，hello_B7未定义
  	auto f="hello"_B7; //error，没有可匹配的_B7运算符
}
```

当然，这里只是简单返回了参数的个数，字面量运算符本意是返回经过处理的字面量，其参数必须是某种形式的数值字面量。

### 15.6 Rvalue References

#### 15.6.1 Reference Collapsing Rules

直接声明引用的引用是不允许的

```c++
int const& r=42;
int const& & ref2ref=i; //error
```

然而通过模板参数替换，类型别名，或者decltype这些手段，引用的引用则是可以实现的

```c++
using RI=int&;
int i=42;
RI r=i;
RI const& rr=r;  //ok,然而rr的类型是int&,不是int&&
```

这种情况称之为引用坍缩规则：首先，去除紧接内层引用之外的所有const和volatile，然后应用下表

| 内层引用 |  +   | 外层引用 |  ->  | 结果 |
| :------: | :--: | :------: | :--: | :--: |
|    &     |      |    &     |      |  &   |
|&||&&||&|
|&&||&||&|
|&&||&&||&&|

```C++
using RCI=int const&;
RCI volatile&& r=42;  //ok,r是int const&
using RRI=int&&;
RRI const&& rr=42; //ok,rr是int&&
```

#### 15.6.2 Forwarding References

```c++
template<typename T> void f(T&& p);  //p是转发引用
void g()
{
  int i;
  int constj=0;
  f(i);  //调用参数是左值，T推断为int&，p的类型是int&
  f(j);  //调用参数是左值，T推断为int const&，p的类型是int const&
  f(2);  //右值参数，T推断为int，p为int&&
}
```

如果一个局部变量声明为类型T，那么可能会出现一些问题

```c++
template<typename T> void f(T&&)  //p是转发引用
{
  T x;  //如果传入一个左值，x会是一个引用，但是没有初始化，会引发错误
  			//所以，通常需要std::remove_reference_t<T> x;
}
```

#### 15.6.3 Perfect Forwarding

```c++
class C{};
void g(C&);
void g(C const&);
void g(C&&);

template<typename T>
void forwardToG(T&& x)
{
  g(static_cast<T&&>(x));  //或者g(std::forward<T>(x));
}

void foo()
{
  C v;
  C const c;
  forwardToG(v);  //调用g(C&)
  forwardToG(c);  //调用g(C const&)
  forwardToG(C()); //调用g(C&&)
  forwardToG(std::move(v)); //调用g(C&&)
}
```

在forwardToG函数中，***参数x***要么是个左值引用，要么是个右值引用，然而***表达式x***必然是个左值，这里可以仔细理解下，换句话说就是进入函数体之前的x是个引用，进入函数体之后对x的使用就成为表达式，这时候就是个左值了。而使用`static_cast<>`则把这个左值转换成其原先的类型，也就是或者左值引用，或者右值引用，这里涉及到引用坍缩规则，所以用的T&&。当然，最好还是使用`std::forward<>()`。

##### Perfect Forwarding for Variadic Templates

```c++
template<typename... Ts> 
void forwardToG(Ts&&... xs)
{
  g(std::forward<Ts>(xs)...);
}
```

在某些情况下，完美转发并不完美，比方说位域左值或者特定的常量，如0指针

```c++
void g(int*);
void g(...);

template<typename T>
void forwardToG(T&& x)
{
  g(std::forward<T>(x));
}
void foo()
{
  g(0);  //g(int *)
  forwardToG(0); //g(...),完美转发并不能捕获0的类型
  
  g(nullptr);  //g(int*)
  forwardToG(nullptr); //g(int*)，可以捕获nullptr的类型
}
```

总之完美转发的目的在于传递一个参数的类型及其引用类型，返回值也是可以进行完美转发的

```c++
template<typename... Ts>
auto forwardToG(Ts&&... xs) -> decltype(g(std::forward<Ts>(xs)...))
{
  return g(std::forward<Ts>(xs)...))
}
```

或者，c++14

```c++
template<typename... Ts>
decltype(auto) forwardToG(Ts&&... xs)
{
  return g(std::forward<Ts>(xs)...))
}
```

#### 15.6.4 Deduction Surprises

```c++
void int_lvalues(int&); //接受int类型的左值
template<typename T>void lvalues(T&); //接受任意类型的左值

void int_rvalues(int&&); //接受int类型的右值
template<typename T>void anything(T&&); //任意类型的左值或者右值
```

不过，这种推断行为只适用于这种情况：首先是个函数模板，其参数必须写成`T&&`这样的形式，并且这个模板参数`T`必须是这个模板声明的参数。

所以，下面的例子都不适用`anything(T&&)`相同的规则

```c++
template<typename T>
class X
{
  public:
  	X(X&&);   //X不是模板参数，这是个右值拷贝构造函数
  	X(T&&);   //这个构造函数不是函数模板
  
  	template<typename U>X(X<U>&&); //X<U>不是模板参数
  	template<typename U>X(U,T&&);  //T是外层模板的参数
};
```

这种推断规则并不常见，并且可以这样解决

```c++
template<typename T>
	typename std::enable_if<!std::is_lvalue_reference<T>::value>::type
	rvalues(T&&);
```

### 15.7 SFINAE(Substitution Failure Is Not An Error)

```c++
template<typename T, unsigned N>
T* begin(T (&array)[N])
{
  return array;
}

template<typename Container>
typename Container::iterator begin(Container& c)
{
  return c.begin();
}

int main()
{
  std::vector<int> v;
  int a[10];
  
  ::begin(v);  //ok,Container=std::vector<int>
  ::begin(a);  //ok,T=int N=10
}
```

对于容器板模板，虽然int[10]可以匹配Container，但是int[10]作为一个数组，没有iterator这个子类型，所以这个替换被忽略，但并不引发错误，这就是SFINAE。

#### 15.7.1 Immediate Context

可以通过定义什么不是即时上下文来定义什么事即时上下文，在实例化过程中

- 类模板的定义
- 函数模板的定义
- 变量模板的初始化
- 缺省参数
- 缺省成员初始化
- 异常指令
- 在替换过程中触发的对特殊成员函数的隐含定义(缺省构造函数、拷贝构造函数、赋值运算符等)

以上这些都不属于函数模板替换的即时上下文，其他的都是。

不在即时上下文中发生的错误都是真实的错误，而不是SFINAE。

```c++
template<typename T>
class Array{
  public:
  	using iterator=T*;
};
template<typename T>
void f(Array<T>::iterator first,Array<T>::iterator last);

template<typename T>
void f(T*,T*);

int main()
{
  f<int&>(0,0); //Error:第一个函数模板中用int&来替换T，
  							//而在把0隐含生成一个临时Array<int&>对象时会失败
  							//因为指向引用的指针是非法的
  							//当然，同样的错误也发生在另外一个函数模板中，不过这并没有引发其他对象的实例化，
  							//只是强制转换成int& *,
  							//所以是个SFINAE
}
```

这两个例子不同的地方在于，第一个例子错误发生在构造`typename Container::iterator`时，这是发生在对模板函数begin进行替换时的即时上下文中。而第二个例子发生在实例化一个或者说两个Array<int&>`临时变量时，这个错误发生在类模板Array的上下文，所以编译器会报错。

详细的说，就是，第一个例子，Container就是int[10]，这个类型的构造本身没有错误，只是在替换`Container::iterator`时出错，因为int[10]不存在这么个成员，此时并没有触发int[10]的构造或者说实例化的过程，错误就发生在实例化begin的即时上下文中。

而第二个例子，首先触发了对`Array<int&>`的一个隐含的实例化过程，在这个过程中出错了，而Array并不属于f函数的即时上下文。

```c++
template<typename T>auto f(T p){
  return p->m
}
int f(...);
template<typename T>auto g(T p) -> decltype(f(p));

int main()
{
  g(42);
}
```

从g(42)可以推断T是int，而在g的声明当中需要确定f(p)的类型，也就是需要知道f()的返回类型。非模板f匹配，但不是一个很好地匹配，因为其参数是可变参数。而模板版本返回类型需要推断，，然而p->m出错了，因为int没有这个成员，也就是说错误发生在非即时上下文。**因此，如果在返回类型容易判断的时候避免使用推断。**

总的来说，就是在函数模板实例化时，如果对函数模板的参数进行直接替换不成功，比方说类型不匹配，如上面的int[10]的iterator成员不存在这样的错误，就是SFINAE，如果在这个替换过程中触发了别的对象的初始化，并且在这个别的对象的初始化过程中出错，编译器就会报错。

### 15.8 Limitations of Deduction

#### 15.8.1 Allowable Argument Conversions

通常来说，模板推断试图找到一个替换，使得模板参数P与调用参数A等价。然而，当达不到这一点的时候，一些差异也是允许的：

- 如果原始的模板参数申明为引用，那么P的类型可以有const/volatile修饰，而A可以没有。

- 如果A是个指针或者指向成员的指针，而P有const/volatile修饰，则A可以转换成由const/volatile修饰的同类型指针。

- 只要不是对一个转换运算符模板进行推断，P可以是A的一个基类，或者，如果A是一个指向类的指针，P可以是一个指向其基类的指针。

  ```c++
  template<typename T>
  class B{
    
  };
  template<typename T>
  class D:public B<T>{
    
  };
  template<typename T>void f(B<T>*);
  void g(D<long> dl)
  {
    f(&dl);  //T=long P=B<T>* A=D<T>*  
  }
  ```

如果在推断上下文中P不包含模板参数，则所有的隐含转换都是允许的

```c++
template<typename T>int f(T,typename T:X);
struct V{
  v();
  struct X{
    X(double);
  };
} v;
int r=f(v,7.0); //ok,T为V，则第二个参数为V::X，X是可以用double构造的
```

然而，

```c++
template<typename T>
T max (T a,T b)
{
  return b<a?a:b;
}

std::string maxWithHello(std::string s)
{
  return ::max(s,"hello");  //Error 
}
```

上面的问题在于通过s推断T为std::string，然而”hello”却被推断为char[6]。然而的然而，如果改成`max<std::string>max(s,“hello”)`就没问题了。

另外，如果两个参数是继承自同一个基类的不同的类，自动推断并不会将这两个参数推断为其共同的基类。

#### 15.8.2 Class Template Arguments

c++17之前，参数推断只限于函数和成员函数。

```c++
template<typename T>
class S{
  public:
  	S(T b):a(b){}
  private:
  	T a;
};

S x(12);  //c++17之前并不会依据参数12来对推断T，这里会出错
```

#### 15.8.3 Default Call Arguments

缺省参数只是在需要的时候才会去实例化

```c++
template<tempname T>
void init(T *loc,T const& val=T())
{
  *loc=val;
}

class S{
  public:
  	S(int,int);
};

S s(0,0);

int main()
{
  init(&s,S(7,43));  //对于S来说，T()是非法的，然而，在显式提供了参数的情况下，并不会去实例化一个T()
  									 //所以这里是没有问题的
}
```

缺省参数不会用来进行推断

```c++
template<typename T>
void f(T x=42)
{
  
}

int main()
{
  f<int>(); //ok,T=int
  f();  //error,不能从缺省参数42来推断T
}
```

#### 15.8.4 Exception Specifications

异常指令与缺省参数一样都是在需要时才实例化，因此同样也不会用来做参数推断。

```c++
template<typename T>
void f(T,int) noexcept(nonexistent(T))); //#1
template<typename T>
void f(T,...);  //#2,c风格的可变参数函数
void test(int i)
{
  f(i,i);  //error：选择#1，然而表达式nonexistent(T)是不合法的，因为nonexistent()这个函数不存在
}
```

本来，函数不存在这样的错误会触发SFINAE，从而会去选择#2这个函数，然而，由于在推断是异常指令被忽略了，所以选择了一个更合适的#1，因为可变参数函数被认为是最糟糕的匹配，于是在之后实例化noexcept时就出错了。

### 15.9 Explicit Function Template Arguments

```c++
void f();
template<typename> void f();
namespace N{
  class C{
    friend int f();  //声明了一个新的的函数
    friend int f<>();  //错误，与函数模板的返回类型冲突
  };
}
```

对于一个普通函数，只会在包含它的最小范围内寻找，如这里的名字空间N，如果找不到，则被认为引入了一个新的实体，然而要注意的是，这时候的N::f是不可见的，也就是将来只有通过ADL才能找到。

然而对于一个模板函数，必须有可见的模板，在这里便是如此，从而引发返回类型冲突错误。

### 15.10 Deduction from Initializers and Expressions

#### 15.10.1 The `auto` Type Specifier

`auto`和`decltype(auto)`被称为占位符类型。

```c++
int x;
auto&& rr=42; //auto=int，右值引用绑定到一个右值
auto&& lr=x;  //auto=int&，由于引用坍缩，lr成为左值引用
```

```c++
template<typename Container> void g(Container c){
  for(auto&& x:c){
    
  }
}
```

这里我们不需要知道容器的迭代接口，使用`auto&&`可以保证在迭代时不需要额外的拷贝，并且仍然可以调用`std::forward<T>()`。

```c++
template<typename T>struct X {T const m;};
auto const N=400u;  //ok，常量类型unsigned int
auto* gp=(void*)nullptr; //ok,gp类型是void*
auto const S::*pm=&X<int>::m; //ok,pm类型是int const X<int>::*
X<auto> xa=X<int>(); //error，auto不能是模板参数
int const auto::*pm2=&X<int>::m; //error，auto不能是变量声明的一部分
```

为什么上面最后两个例子不支持，并没有特别的技术上的理由，只是委员会认为那样不好。

可以理解下`auto const S::*pm=&X<int>::m;`，在这里，首先确认pm是个指向成员的指针，它的值是`&X<int>::m`,而m的类型是`int const`，所以pm的类型就是`int const X<int>::*`，前面的`const S::`对这个推断并没有影响，因为`const`修饰的是`auto`，对于一个已经是`const`的变量，多加一个`const`并不会产生额外的`const`性，而`S::`只是修饰`pm`，表示其归属性。

##### Deduced Return Types

```c++
auto f();
auto f(){ return 42;} //ok

int known();
auto known(){ return 42;}  //这样是错误的。
```

##### Deducible Nontype Parameters

```c++
template<auto V> struct S;  //注意与前面X<auto> xa=X<int>();的区别，那里是声明一个变量。
S<42>* ps;  //ok
S<3.14>* pd; //error，非类型参数不能是浮点数
```

使用非类型参数推断时，通常可以配合使用decltype

```C++
template<auto V> struct Value{
  using ArgType=decltype(V);
};
```

auto还可以用于可变参数，注意这里针对的是非类型参数，auto当然不能代替typename

```c++
template<auto... VS> struct Values{
  
};
Values<1,2,3> beginning;
Values<1,'x',nullptr> triplet;
```

如果需要单一类型，则可以

```c++
template<auto V1,decltype<V1>... VRest> struct HomogeneousValues{
  
};
```

但这时候实例化参数列表不能为空。

最后看一个auto用于类成员模板参数的例子

```c++
template<typename> struct PMClassT;  //主模板
template<typename C,typename M> struct PMClassT<M C::*>{  //部分特化模板，只是为了获取C
  using Type=C;
};
//注意这种用法，PMClass就是上面模板参数C的别名
template<typename PM> using PMClass=typename PMClassT<PM>::Type; 

template<auto PMD> struct CounterHandle{
  PMClass<decltype(PMD)>& c;
  CounterHandle(PMClass<decltype(PMD)>& c):c(c){}
  void incr(){
    ++(c.*PMD);
  }
};

struct S{
  int i;
};

int main()
{
  S s{41};
  CounterHandle<&S::i> h(s);  
  h.incr();
}
```

对于`CounterHandle<&S::i> h(s)`，我们可以这么来理解：

首先，必须了解类模板目前是不存在对类型参数进行推断的，但是对非类型参数本身其类型还是可以进行推断的，这也是auto存在的意义。

其次看PMClass的实例化，PMClass的实例化又涉及到PMClassT的实例化，也就是说，首先实例化了一个PMClassT实例，其参数是一个非类型参数，一个类成员指针`&S::i`，从而使用了`template<typename C,typename M> struct PMClassT<M C::*>`这个模板，并推断出M就是int，C就是S，从而PMD的类型就是`int S::*`，其值已经知道是`&S::i`，当然，这种表示方法只是说明了i在S中的偏移，而不是实际的地址。

因此，对h这个CounterHandle的实例，其成员c实际上就是个S的实例，从而h.incr()就是增加了c.i的值

总的来说，就是在类模板中一个成员的类型由非类型的模板参数确定，为了确定这个非类型参数的类型，使用了一个辅助类模板，这个辅助类模板的唯一作用就是确定这个类型。

而对于本例而言，其目的就是说明怎样利用auto通过一个类成员指针推断出这个类，并从而操作这个类的成员，如果只是为了操作一个类的成员，当然不需要绕这个弯。

#### 15.10.2 Expressing the Type of an Expression with `decltype`

传入decltype的参数可以是一个被声明的实体或者一个表达式，其结果可能会不同

- 如果`e`是一个实体的名字(比如变量，函数，枚举或者数据成员)，或者是对类成员的访问，`decltype(e)`返回的结果是那个实体或者类成员所声明的类型，因此，decltype可以用来检查变量的类型。

  ```c++
  auto x=...;
  auto y1=x+1;   //y1的类型与x和+运算都有关系，比方说如果x是char，则y1是int，因为1被推断为int
  decltype(x) y2=x+1  //y2的类型必定与x相同
  ```

- 如果`e`是任意的其他类型的表达式，毕竟c中都是表达式，则`decltype(e)`反射了该表达式的类型和值的范畴

  - 如果`e`是一个类型T的lvalue，`decltype(e)`产生`T&`
  - 如果`e`是一个类型T的xvalue，`decltype(e)`产生`T&&`
  - 如果`e`是一个类型T的prvalue，`decltype(e)`产生`T`

```c++
void g(std::string&& s)
{
  std::is_lvalue_reference<decltype(s)>::value;  //false
  std::is_rvalue_reference<decltype(s)>::value;  //true
  std::is_same<decltype(s),std::string&>::value; //false
  std::is_same<decltype(s),std::string&&>::value; //ture
  
  std::is_lvalue_reference<decltype((s))>::value;  //true
  std::is_rvalue_reference<decltype((s))>::value;  //false
  std::is_same<decltype((s)),std::string&>::value; //true
  std::is_same<decltype((s)),std::string&&>::value; //false
}
```

前面四条检查的是变量s的类型，而后四条检查的则是表达式(s)的值的范畴进行检查。

decltype可以完整地保留表达式的类型和值的信息，所以可以用来传递一个函数的返回值的类型。

```c++
??? f();
decltype(f()) g()
{
  return f();
}
```

如果`f()`返回`int&`，`decltype`首先会决定表达式`f()`的值是int，这个表达式是个左值，所以`decltype`判定的结果就是`g()`返回一个左值引用。

类似的，如果`f()`返回一个右值引用，调用`f()`会产生一个xvalue，于是`decltype(f())`产生一个右值引用型。

当auto不足以产生足够的信息时，可以配合decltype，比方说`auto element=*pos;`这个语句，通常带来某种拷贝，于是我们使用引用`auto& elemen=*pos`，然而，这时候如果`*pos`返回的是一个值，这样做就会出错，所以我们可以`decltype(*pos) element=*pos`。

#### 15.10.3 `decltype(auto)`

```c++
int i=42;
int const& ref=i; //ref类型是int const&,并且指向i
auto x=ref;  //x是int，并且是一个新的独立的对象
decltype(auto) y=ref; //y的类型是int const&并且同样指向i

std::vector<int> v={42};
auto x=v[0];  //x是一个新的int对象
decltype(auto) y=v[0];  //y是一个引用int&
```

所以前一节的例子`decltype(*pos) element=*pos`我们可以写成`decltype(auto) element=*pos`。

```c++
template<typename C> class Adapt{
	C container;
  ...
  decltype(auto) operator[](std::size_t idx){
    return container[idx];
  }
};
```

如果`container[idx]`返回一个左值，则`decltype(auto)`会返回一个左值引用。如果返回一个右值，`decltype(auto)`则会返回一个对象而不是引用。

与auto不同，decltype(auto)不允许使用修饰符或者在声明时通过运算来改变其类型。

```c++
decltype(auto)* p=(void*)nullptr;  //invalid
int const N=100;
decltype(auto) const NN=N*N; //invalid
```

也就是说在使用`decltype(auto)`时，不能使用类型转换来改变一个对象的类型，也不能对对象进行运算

如前面看到，变量和表达式是有区别的

```c++
int x;
decltype(auto) z=x;  //z是个int型的对象
decltype(auto) r=(x); //r是一个int&
```

这也说明了括号的作用

```c++
int g();
...
decltype(auto) f(){
  int r=g();
  return (r);  //运行时错误:返回一个临时对象的引用。
}
```

注意上面错误是在使用`decltype(auto)`时的情况。

```c++
template<decltype(auto) Val> class S
{
  
};
constexpr int c=42;
extern int v=42;
S<c> sc;  //#1 S<42>
S<(v)> sv;  //#2 S<(int&)v>
```

上面如果用`S<v>`是错误的，因为`decltype(v)`的结果是`int`，因此这里模板参数需要一个整型常量值，而`v`并不是。

```c++
template<auto N> struct S{};
template<auto N> int f(S<N> p);
S<42> x;
int r = f(x);
```

这里模板f的参数N可以从函数调用函数推断出来，因为这时候S<42>属于推断上下文，或者说，在这里不需要再对S做初始化，已经初始化好了。

```c++
template<auto V> int f(decltype(V) p);
int r1=f<42>(42);  //ok
int r2=f(42);  //error:decltype<V>不是推断上下文。
```

因为对一个非类型模板，匹配decltype<V>的V不是唯一的，比方说decltype<7>和decltype<42>的结果是一样的，但是对于f这个模板，V必须是确定的。

#### 15.10.4 Special Situations for `auto` Deduction

```c++
 #include <initializer_list>

template<typename T>
void deduceT(T t){}
template<typename T>
void deduceInitList(std::initializer_list<T>){}

int main()
{
   	deduceT({2,3,4});  	//#1 Error 
   	deduceT({1});				//#2 Error

    deduceInitList({2,3,4,5}); //#3 ok

    auto primes={2,3,5,7};  //primes是std::initializer_list<int>
    deduceT(primes);	//#4 OK,T=std::initializer_list<int>
}
```

不能够从一个初始化列表来推断一个模板参数，如上面的 #1、#2。然而如果函数的参数有更多的限定或者修饰，如上面的#3，明确说明了参数是个初始化列表类型，初始化列表也是可以用来推断T的。而对于#4，primes首先被初始化为一个`std::initializer_list<int>`，于是T被推断为`std::initializer_list<int>`，注意不是T。实际上这里也说明了模板函数推断上下文与非推断上下文的关系，primes是一个已经初始化的对象，并不需要在deduceT函数的调用中再去初始化，而#1、#2则需要离开函数调用去初始化一个初始化列表。

对于初始化列表，要注意一些问题

```c++
auto oops{0,8,15};  //这是错误的，只能是oops={0,8,15}，这时候oops的类型就是一个std::initializer_list<int>
auto val{2};  //val是个值为2的int

auto subtleError()
{
  return {1,2,3}; //ERROR,因为一个初始化列表实际上是一个指针，指向一个底层的数组，函数结束之后，这个数组就悬空了
}
```
如果一个auto带了几个变量声明，那么这几个变量会独立进行推断，只有当这些推断都合法并且是同一类型，才是合法的
```c++
auto first=container.begin(),last=container.end(); //两个变量分别做推断，各自都是同一个容器的迭代器类型，所以正确的
char c;
auto *cp=&c,d=c; //ok,*cp=&c可以推断出auto是char，d=c也推断出auto是char，所以是正确的
auto e=c,f=c+1; //e=c推断auto是char，f=c+1推断auto是int，两者不符，所以是错误的
```

对于函数返回值的推断，同样是单独对每一条return做推断。

```c++
auto f(bool b){
	if(b){
    return 42.0;  //推断类型为double
  }else{
    return 0;   //错误，推断冲突
  }
}
```

而对于递归函数，则有个先后的问题

```c++
auto f(int n)
{
  if(n>1){
    return n*f(n-1);  //error,f(n-1)的类型未知
  }else{
    return 1;
  }
}
```

但是

```c++
auto f(int n)
{
  if(n<=1){
    return 1;  //返回类型推断为int
  }else{
    return n*f(n-1);  //ok,f(n-1)是int
  }
}
```

返回类型推断还有一个比较特殊的与void返回值相关的情况

```c++
auto f1(){}  //ok,void
auto f2(){return;} //ok,void
auto* f3(){} //error,auto*不能推断为void
```

对于函数模板而言，返回类型推断需要能够对参数做即时的实例化以确定其类型。但如果涉及到SFINAE，则有些复杂

```c++
template<typename T,typename U>
auto addA(T t,U u)->decltype(t+u)
{
  return t+u;
}
void addA(...);

template<typename T,typename U>
auto addB(T t,U u)->decltype(auto)
{
  return t+u;
}
void addB(...);

struct X{};

using AddResultA=decltype(addA(X(),X()));  //ok,AddResultA是void
using AddResultB=decltype(addB(X(),X()));  //error
```

上面的错误是发生在函数重载解析的时候，使用decltype(auto)引发的。因为addB模板的函数体必须全实例化已决定其返回值，也就是说要实例化X对象，这个实例化不发生在对addB调用的即时上下文中，因而不会引发SFINAE，而是在实例化X时，X没有定义`+`运算符引发的错误，也就是在addB模板中的`return t+u`处。而对于addA模板，并不需要完全实例化出来一个X临时对象，因为已经知道返回值必然是个X，当发现X没有`+`运算符时，就会引发SFINAE，从而选择了非模板的函数。

总之，使用返回值类型推断时一定要小心，它不仅仅是个捷径，有可能会引发一些非即时上下文中初始化的错误。

#### 15.10.5 Structured Bindings

有三种不同的实体可以通过结构化绑定来初始化

- 简单类类型，其所有非静态成员都是public的，不论是这个类本身的还是继承来的，也没有匿名联合，括起来的标识符的数量必须与类成员的数量相等。

  ```c++
  struct MaybeInt{ bool valid; int value;};
  MaybeInt g();
  auto const&& [b,N]=g();  
  ```

- 数组

  ```c++
  int main()
  {
    double pt[3];
    auto &[x,y,z]=pt;
    x=3.0;y=4.0;z=0.0;
    plot(pt);
  }
  ```

  注意这里把数组pt跟[x,y,z]绑定之后，对pt的操作会影响到[x,y,z]，反之亦然。

  ```c++
  auto f()->int(&)[2];  //f()返回一个对int数组的引用
  
  auto [x,y]=f();  //#1
  auto& [r,s]=f(); //#2
  ```

  对于#1，x，y的值分别是从f()所返回的数组当中逐个拷贝过来的，而不是对其的引用，则是引用了。

- std::tuple_like类
  
  ```c++
  #include <tuple>
  
  std::tuple<bool,int> bi{true,42};
  auto [b,i]=bi;
  int r=i;  //r=42
  ```
  下面这个例子说明了怎样使用std::tuple_like类来处理一个数据结构，除了pair，tuple，array之外任意的类和枚举都是可以的
  ```c++
  #include <utility>
  
  enum M{};  //定义一个空枚举，
  template<> class std::tuple_size<M>{
    public:
    	static unsigned const value=2;  //设定M为只有两个元素的枚举
  };
  template<> class std::tuple_element<0,M>{
    public:
    	using type=int;  //M的第一个元素的类型是int
  };
  template<> class std::tuple_element<1,M>{
    public:
    	using type=double;  //M的第二个元素的类型是double
  };
  
  template<int> auto get(M);
  template<> auto get<0>(M){return 42;}  //M的第一个元素的值是42
  template<> auto get<1>(M){return 7.0;} //M的第二个元素的值是7.0
  
  auto [i,d]=M(); //等价于int&& i=42;double&& d=7.0  
  ```
  
  注意上面i，d都是右值引用，也就是说，接下来对i，d的操作并不影响M的返回值，实际上M当中也并没有存储值的实体，换句话说，M是个计算型变量。另外就是不能再次次使用[i,d]来绑定M()。

#### 15.10.6 Generic Lambdas

```c++
template<typename Iter>
Iter findNegative(Iter first,Iter last)
{
  return std::find_if(first,last,
                     [](auto value){
                       return value<0;
                     })
}
```

上面的 lambda就是个所谓的泛型lambda。

实际上，c++对lambda表达式的处理是，首先创建一个通常被称为闭包类型的类，然后把这个lambda表达式作为这个类的一个实例，通常被称为闭包或者闭包对象。这个闭包类型包含一个函数运算符。

```c++
[](int i){
  return i<0;
}
```

大体上会处理成

```c++
class SomeCompilerSpecificNameX
{
  public:
  	SomeCompilerSpecificName();  //只被编译器调用
  	bool operator()(int i) const{
      return i<0;
    }
};
```

因此，对于

```c++
foo(...,
    [](int i){
      return i<0;
    });
```

实际上会处理成为

```c++
foo(...,
   SomeCompilerSpecificNameX{});  //传递一个闭包类型的对象
```

如果lambda捕获局部变量

```c++
int x,y;
...
[x,y](int i){
  return i>x&&i<y;
}
```

则

```c++
class SomeCompilerSpecificNameY{
  private:
  	int _x,_y;
  public:
  	SomeCompilerSpecificNameY(int x,int y):_x(x),_y(y){}
  	bool operator()(int i){
      return i>_x&&i<_y;
    }
};
```

对于泛型lambda

```c++
[](auto i){
  return i<0
}
```

则会

```c++
class SomeCompilerSpecificNameZ
{
  public:
  	SomeCompilerSpecificName();  //只被编译器调用
  	template<typename T>
  	bool operator()(T i) const{
      return i<0;
    }
};
```

这个成员函数模板只有在闭包被调用的时候才会实例化，而不是在lambda表达式出现的地方。

```c++
#include <iostream>
template<typename F,Typename... Ts> void invoke(F f,Ts... ps)
{
  f(ps...);
}

int main()
{
  invoke([](auto x,auto y){
    				std::cout<<x+y<<std::endl;
  				},
        21,21);
}
```

### 15.11 Alias Templates

```c++
template<typename T,typename Cot>
class Stack;

template<typename T>
using DequeStack=Stack<T,std::deque<T>>;

template<typename T,typename Cont>
void f1(Stack<T,Cont>);

template<typename T>
void f2(DequeStack<T>);

template<typename T>
void f3(Stack<t,std::deque<T>);  //等价于f2

void test(DequeStack<int> intStack)
{
  f1(intStack);  //OK:T=int,Cont=std::deque<int>
  f2(intStack);  //OK:T=int
  f3(intStack);  //OK:T=int
}
```

上面的例子只是说明别名模板就是个模板别名，不是一个新的类型，换句话说，别名模板在推断时是透明的。

注意别名模板是不能特化的

```c++
template<typename T> using A=T;
template<> using A<int>=void;  //ERROR,但是假如是可行的话。。。
```

那么对于void类型将无法匹配`A<T>`并断定T是void，因为`A<int>`和`A<void>`都将等于void。当然这里只是针对推断而言，因为出现了多对一的情况，因此无法从一个一推断到底对应哪个多。

### 15.12 Class Template Argument Deduction

```c++
template<typename T1,typename T2,typename T3=T2>
class C
{
  public:
  	C(T1 x=T1{},T2 y=T2{},T3 z=T3{});
};

C c1(22,44.3,"hi");  //<int,double,char const*> 全部模板参数由调用参数推断
C c2(22,44.3);  //<int,double,double>  T1,T2由调用参数推断，进而推断出T3
C c3("hi","guy"); //<const char*,const char*,const charT3
C c4;  //Error,T1,T2未定义，缺省调用参数不参与推断，实际上，T未确定时也无从构造一个T的对象
C c5("hi"); //Error,T2未定义

C<string> c10("hi","my",42); //Error,只有T1显式指定，T2未做推断
C<> c11(22,44.3,42);  //Error,T1和T2都未显式指定
C<string,string> c12("hi","my");  //ok,
```

可以看到，就类模板而言，

- 要么全部通过构造函数调用参数进行推断，这时候如果有缺省模板参数，可以不用提供其对应的调用参数。
- 要么全部显示指定模板参数，这时候如果有缺省模板参数，也可以不指定这个缺省参数。

#### 15.12.1 Deduction Guides

```c++
template<typename T>
class S{
  private:
  	T a;
  public:
  	S(T b):a(b){}
};
template<typename T> S(T)->S<T>;  //推断指南
S x{12};  //ok,等同于S<int> x{12};
S y(12);  //ok,等同于S<int> y(12);
auto z=S{12};  //ok,等同于 auto z=S<int>{12};
```

注意推断指南与函数模板句法上的区别，另外推断指南也可以用explicit修饰。

`S* p=&x`是不合句法的，错误提示无法从指针推断模板参数的类型，因为这里S只是一个占位符，其后必须紧接变量的名字以及对这个变量的初始化。虽然我们可以用auto p=&x`或者`auto p{&x}`来声明p，但注意，我们这里自动推断的是p的类型，而不是推断模板参数T是什么。

同样道理，`S s1(1),s2(2.0)`也是不合法的，因为这里会推断出`S<int>`和`S<double>`两个类，一个占位符有两个实体符合，自然就是不合法的了，这与`auto`的情况是一样的。

#### 15.12.2 Implicit Deduction Guides

实际上，前面的例子推断指引是可以去掉的，这也就是所谓的隐式推断指引，当然只是针对构造函数。

通常来说，一个类模板的每一个构造都会有一个推断指引。因此对于主模板的每一个构造函数和构造函数模板，都会提供一个隐含进行推断的机制，也就是自动生成隐式的推断指引，以前面的`template<typename T> S(T) -> S<T>`为例，这是一个完整地推断指引：

- 隐式指引的模板参数列表由类模板的参数组成，如果构造函数是个模板，则其后跟着这个构造模板函数的模板参数列表。构造函数模板的缺省模板参数保留。例如前面的`template<typename T>`，就是一个隐式指引的参数列表。注意，并不需要与类模板参数列表完全一致。
- 接下来是那个像函数签名的表达式，是从构造函数或者构造函数模板直接复制过来的。例如前面的 `S(T)`。
- 最后那个像函数返回值的表达式，则是`->`跟着模板名加上从类模板拿来的模板参数，例如前面的`-> S<T>`。

另外要注意的是用花括号初始化时，单个元素与多个元素处理是不同的

```c++
S x{12};  //x:S<int>
S y{x};   //y:S<int>，而不是S<S<int>>
S z(x);  //z:S<int>
```

类似的vector的例子

```C++
std::vector v{1,2,3};  //vector<int>
std::vector w2{v,v};   //vector<vector<int>>
std::vector w3{v};     //vector<int>!!!
```

所以要注意

```c++
template<typename T,typename... Ts>
auto f(T p,Ts... ps){
	std::vector v{p,ps...};
}
```

在这里，如果T是个vector，那么v的类型就与ps是否为空相关了，或者是个vector<T>,或者是个vector<vector<T>>，当然，我们这里假定T与Ts的类型都是一样的。

另外，

```c++
template<typename T>
struct ValueArg{
  usint Type=T;
};

template<typename T>
class S{
  private:
  	T a;
  public:
  	using ArgType=typename ValueArg<T>::Type;
  	S(ArgType b):a(b){}
};
```

在C++17之前，上面这段代码没有任何问题，但是在c++17，会屏蔽掉隐式推断指南，我们可以写一条推断指南，

`template<typename> S(typename ValueArg<T>::Type)->S<T>;`

这一条实际上也是所生成的隐式推断指南。因为`ValueArg<T>::`不属于推断上下文，所以像`S x(12)`这样的声明推断指南是无效的。

#### 15.12.3 Other Subtleties

##### Injected Class Names

```c++
template<typename T> struct X{
  template<typename Iter> X(Iter b,Iter e);
  template<typename Iter> auto f(Iter b,Iter e){
    return X(b,e);  //这是啥
  }
};
```

在C++14，`X(b,e)`中的`X`就是所谓的注入类名，等同于`X<T>`。然而如果要是依据类模板参数推断，应该是`X<Iter>`。但是为了保持后向兼容，不对注入类名做模板参数推断。

##### Forwarding References

```C++
template<typename T> struct Y{
  Y(T const&);
  Y(T&&);
};
void g(std::string s){
  Y y=s;
}
```

这里的意图是通过与拷贝构造函数关联的隐式推断指南来推断T是std::string。然而，我们可以把隐式推断指南写下来：

```c++
template<typename T> Y(T const&) -> Y<T>;  //#1
template<typename T> Y(T&&) -> Y<T>; //#2
```

我们知道，在推断时，T&&实际上是作为一个转发引用，也就是说由于引用坍缩它最终有可能会是一个左值引用。

就上面这个例子而言，首先`s`是个左值。

- 由#1，推断T是一个`std::string`，而调用参数会调整为`std::string const`。
- 由#2，通常会推断T是一个`std::string&`，由于没有调整为const的需要，这会被认为是一个较好的匹配。
- 因此，这时候可能会引发一些错误，比如在模板参数不能为引用的场合，又比如悬空引用。
- 所以，委员会决定，对于T&&，在隐式推断指引时，如果T是类模板参数，禁用对其的特别推断规则，也就是说不坍缩了，T&&还是T&&，而如果T是构造函数模板的参数，则常规规则还是应用的。

##### The `explicit` Keyword

在推断指引中使用`explicit`表示只在直接初始化时使用，而不考虑复制初始化。

```c++
template<typename T,typename U> struct Z{
  Z(T const&);
  Z(T&&);
};

template<typename T> Z(T const&) -> Z<T,T&>; //#1
template<typename T> explicit Z(T&&) -> Z<T,T>;  //#2

Z z1=1;  //只有#1，等同于Z<int,int&> z1=1;
Z z2{2}; //#2更佳，等同于Z<int,int> z2{2};
```

也就是说，直接初始化时优先考虑有explicit修饰的指引，复制初始化时不考虑有explicit修饰的指引。

##### Copy Construction and Initializer Lists

```C++
template<typename... Ts> struct Tuple{
  Tuple(Ts...);
  Tuple(Tuple<Ts...> const&);
};
```

我们把其隐式推断指引写一下

```c++
template<typename... Ts> Tuple(Ts...) -> Tuple<Ts...>;  //#1
template<typename... Ts> Tuple(Tuple<Ts...> const&) -> Tuple<Ts...>;  //#2
```

- 对于`auto x=Tuple{1,2};`，会选择#1，因此x是一个`Tuple<int,int>`。实际上与`auto x{Tuple{1,2}}`以及`Tuple x{1,2}`是等价的。

- 对于

  ```c++
  Tuple a=x;
  Tuple b(x);
  ```

  两个指引都是匹配的，但是#2更匹配，因此a、b都是由x复制构造而来。

- 对于

  ```c++
  Tuple c{x,x};
  Tuple d{x};
  ```

  第一个里的两个x调用了两次#2，也就是复制构造了两个临时对象，c则调用了#1，最后c是`Tuple<Tuple<int,int>,Tuple<int,int>>`，
  
  第二个则是调用了#2，因此d是`Tuple<int,int>`而不是`Tuple<Tuple<int>>`，类似的`auto e=Tuple{x}`，也是`Tuple<int,int>`。这里都是调用了复制构造函数。

##### Guides Are for Deduction Only

指引不是函数模板，它只是用来推断参数，而不是被用来被调用。因此，对于指引来说，传值和传引用的区别是不重要的

```c++
template<typename T> struct X{};
template<typename T> struct Y{
  Y(X<T> const&);
  Y(X<T>&&)
};
template<typename T> Y(X<T>) ->Y<T>;
```

假定类型`X<TT>`的一个值xtt，不论是左值还是右值，都会推断出类型`Y<TT>`，至于哪一个构造函数会被选择，则要看xtt的情况。




