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