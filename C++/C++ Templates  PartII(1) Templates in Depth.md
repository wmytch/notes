# C++ Templates  PartII(1):Templates in Depth

[TOC]



## Chapter 12 Fundamentals in Depth

### 12.1 Parameterized Declarations

目前C++支持4种基本模板：类模板，函数模板，变量模板，以及别名模板。它们可以出现在名字空间范围(全局或者名字空间)也可以出现在类范围内。在类范围内他们就变成嵌套类模板，成员函数模板，静态数据成员模板，以及成员别名模板。在C++17中还引入了推断指引。

```c++
template<typename T>   //名字空间范围类模板
class Data
{
  public:
  	static constexpr bool copyable=true; //新标准，注意constexpr,这样不需要在别的地方初始化了
};

template<typename T>  //名字空间范围函数模板
void log(T x){
  ...
}

template<typename T>  //名字空间变量模板，自c++14
T zero=0;

template<typename T>  //同上
bool dataCopyable=Data<T>::copyable;  

template<typename T>
using DataList=Data<T*>;  //名字空间范围别名模板
```

```c++
class Collection
{
  public:
  	template<typename T>   //在类中定义的成员类模板
  	class Node
    {
      
    };
  	template<typename T>   //类成员函数模板，隐含为inline函数
  	T* alloc(){
      
    }
  	template<typename T>   //成员模板变量。对比上面例子中的copyable的定义
  	static constexpr T zero=0;  //原书此处没有constexpr，这是错误的
  	template<typename T>   //成员别名模板
  	using NodePtr=Node<T>*
};
```

要注意的是在c++17

- 变量，包括静态数据成员，以及变量模板都***有可能***是`inline`的，这也就意味着这他们的定义可能重复出现在不同的翻译单元
- 但是，就变量而言，必须显式的声明为inline才会是inline的。
- 成员函数包括函数模板，在类定义内部定义时，已经隐含是inline的了

```c++
//file1.cpp
#include <iostream>
inline int i=1;
//int i=1;
int j=1;
inline int k=1;

void print();
int main()
{
    print();
    std::cout<<"i1:"<<i<<"  j1:"<<j<<"  k1:"<<k<<std::endl;
}
```

```c++
//file2.cpp
#include <iostream>
//int i=2;
inline int i=2;
inline int j=2;
int k=2;

void print()
{
    std::cout<<"i2:"<<i<<"  j2:"<<j<<"  k2:"<<k<<std::endl;
}
```

- 先看两个文件中的`i`，这里都加了`inline`，否则编译就会出错，当然用`extern/static`修饰也是可以通过编译的，运行结果很常规，就不讨论了，只说两个`inline`的情况，输出的`i`值与编译时两个文件的顺序有关，哪个在前面，输出的值就是哪个文件里的`i`值，这个应该与编译器有关，反正两个值肯定是一样的 。
- 而对于`k`和`j`，输出的都是非`inline`的那个。
- 这里比较有趣的是如果两个文件分别定义两个变量为`extern int i=1`和`inline int i=2`，那么编译器会对第一个声明发出警告，输出结果则是第一个`i`的值。

```c++
#include <iostream>
                              
template<typename T>          
class Data                    
{
    public:
        static constexpr bool copyable = true;  
        template<typename U>  
        static U zero;        //如果要在这里初始化zero，则static constexpr U zero,如前所述
};                            
template<typename T>          
template<typename U>          
U Data<T>::zero=1;            //特化
                       

template<>
template<>
float Data<int>::zero<float> = 3.3; //特化

Data<float>::zero<int> = 4; //ERROR：不能赋值
int main()
{
    Data<double>::zero<float> = 2.2; //OK，赋值
  	Data<double>::copyable=false;   //ERROR:constexpr不能赋值
  	std::cout<<Data<int>::copyable<<" "<<Data<double>::zero<int><<std::endl;
    std::cout<<Data<int>::copyable<<" "<<Data<double>::zero<float><<std::endl;
    std::cout<<Data<int>::copyable<<" "<<Data<int>::zero<float><<std::endl;
  	std::cout<<Data<int>::copyable<<" "<<Data<float>::zero<int><<std::endl;  //zero为1
}   

```

- 这里主要看的是怎样实例化`zero`这个变量，以及对其引用的方法。
- 需要注意的是在全局范围对`zero`只能初始化或者说特化，在函数里才可以赋值，当然前提是这个变量不能是const的以及constexpr的。
- 所以，如果你需要在程序中改变一个类静态数据成员，就在类定义外部对其初始化，或者，从c++17起，使用`inline`，比方说上面的例子：`inline static bool copyable = false`，这样在函数中就可以对其赋值了。



```c++
template<typename T>
class List
{
  public:
  	List()=default;
  	template<typename U>
  	List(List<U> const&);
};
```

- 注意模板构造函数取消了自动生成的缺省构造函数，所以这里需要手工生成一个缺省构造函数，以备构造`List<T>`的实例。
- 注意这里仅仅指的是取消缺省构造函数，而模板拷贝/移动构造函数和模板赋值运算符并不会取消相应的缺省函数，但是他们会取消缺省构造函数。
- 同样要注意的是如果不使用缺省构造函数，那么不去管它也是没问题的。

#### 12.1.1 Virtual Member Functions

成员函数模板不能是虚拟的 ，因为虚函数表的大小是固定的，而虚函数模板实例的数量只有到程序全部翻译之后才会知道，目前编译器和链接器尚不支持这种机制。

类模板里的普通成员函数是可以虚拟的。

```c++
template<typename T>
class Dynamic
{
  public:
  	virtual ~Dynamic(); //每个Dynamic<T>实例都有一个析构函数
  	template<typename  T2>
  	virtual void copy(T2 const&); //错误，每一个给定的Dynamic<T>，copy()的实例数量是未知的
};
```

尝试理解下这个问题，比方说创建一个基类`Dynamic<T>`的实例，要注意的是 ，这里每一个类是依据T来区分，而不会去考虑T2，但是创建虚函数表时，就得考虑T2，每一个T2就代表一个虚函数，如果支持虚成员函数模板，那么，这个时候其虚函数表应该有多少项？这个只能在全部程序翻译完成，知道有多少基类实例才能确定了，更糟糕的是，如果一个类继承了这个基类，同样的道理，这个子类的虚函数表又该有多少项？毕竟，基类的虚函数表有多少项尚且一时不得而知，子类的虚函数表也必然无法确定。

#### 12.1.2 Linkage of Templates

- 在同一个范围内类模板不能与不同实体同名

```c++
int C;
class C;   //OK,类名与非类名属于不同的空间

int X;
template<typename T>
class X;  //ERROR，冲突了

struct S;
template<typename T>
struct S;  //ERROR,冲突了
```

- 模板名字不能是C链接属性的

```C++ 
extern "C++" template<T>
void normal();  //缺省情况，链接指定可以省略
extern "C" template<typename T>
void invalid();  //错误，
extern "Java"  template<typename>
void javalink();  //非标准的，不确定什么编译器支持
```

至于原因，与在C++程序中调用C函数的要使用`"extern C”`的原因是一样的。

- 模板通常具有外部链接性，当然也有例外
  - 名字空间范围里static限定的函数模板
  - 直接或者间接属于匿名空间的模板，这时候具有内部链接属性
  - 匿名类的成员模板，这时候没有连接属性

```c++
template<typename T>
void external();   //指向另一个文件的属于同一名字空间的同名实体，这是个声明

template<typename T>
static void internal();  //与其他文件中的实体无关

template<typename T>
static void internal();  //上一个声明的再次声明

namespace{
  template<typename>
  void otherInternal();   //同样与其他文件无关，
  												//因为namespace{
  												//...
  												//}等同于
  												//namespaceXXX{
  												//....
  												//} 
  												//using namespaceXXX; 
  												//上面XXX是随机的字符组合
}

namespace{     //上面声明的再次声明
  template<typename>
  void otherInternal(); 
}

struct {
  template<typename T> void f(T){};//没有链接性，不能再次声明，必须在这里定义，
  																 //对一个匿名类，在类外部无法提供定义
  																	//因为没有名字，无从指向
} x;
```

- 当前模板不能在函数范围或者局部类范围内声明，某种形式的lambda表达式除外，将来会看到。
- 一个模板实例的链接属性就是这个模板的链接属性。

```c++
template<typename T> T zero=T{};//其所有实例都是外部连接的，即便对于zero<int const>也是如此
int const zero_int =int{};   //是内部链接，因为这是个cosnt类型，
template<typename T> int const max_volume = 11;//所有实例却都是外部链接的，
																							//即便其类型都是int const的。
```

#### 12.1.3 Primary Template

主模板的声明不需要加上`<T>`

```c++
template<typename T>class Box;
template<typename T>class Box<T>;  //错误，不做特化
template<typename T>void translate(T);
template<typename T>void translate<T>(T); //错误，这种场合函数不允许这种句法，
																					//只有在调用函数时才有可能出现，那是因为有多个模板参数，
																					//并且其中有无法推断或者必须指定的参数
template<typename T>constexpr T zero=T{};
template<typename T>constexpr T zero<T> = T{}; //错误，不做特化
```



### 12.2 Template Parameters

在不涉及模板参数名字的时候，可以不用给模板参数命名，比方说声明当中：

```c++
template<typename,int>
class X;   //一个声明，不是定义。
```

#### 12.2.1 Type Parameters

类型参数指用`typename`或者`class`引入的模板参数。

```c++
template<typename Allocator>
class List{
	class Allocator * allocptr; //错误，直接使用Allocator * allocptr;
	frient class Allocator; //错误，直接使用Allocator
};
```

因为实际上一个模板参数的名字已经是一个类型的别名，不能再加上class修饰了。

比较一下：

```c++
#include <iostream>
class X{
public:
    X(){
        std::cout<<"class X"<<std::endl;
    }
};

int main()
{
 	typedef class X XA;
  class X s; //OK,去掉class也是正确并且更合适的
  class XA xs;  编译错误
  XA xs; //OK
}

```

#### 12.2.2 Nontype Parameters

非类型参数指这些

- 整型类型或者枚举类型
- 指针类型，虽然标准只允许对象指针和函数指针，但是目前所有的编译器都支持`void *`
- 指向成员的指针类型
- 左值引用类型，包括对对象和函数的引用
- `std::nullptr_t`
- 包含`auto`或者`decltype(auto)`的类型

除此之外，浮点类型将来也许会加入。

非类型参数有时候也是可以或者必须用`typename`或者`class`引入的：

```c++
template<typename T,typename T::Allocator * Allocator>
class List;
template<class X*>
class Y;
```

也可以指定函数和数组类型，不过它们会被隐含的退化成指针。

```C++
template<int buf[5]>class Lexer;  //buf实际上是个int *
template<int * buf>class Lexer; //再次声明，不是定义

template<int fun()>struct FuncWarp; //函数指针
template<int (*)()>struct FuncWarp;  //再次声明
```

非类型指针的声明更像一个变量，但是不能用`static`、`mutable`等非类型限定符修饰，不过可以使用`const`和`volatile`，这个时候实际上是会被简单忽略掉的，如果放在最外层的话：

```c++
template<int const length>class Buffer;
template<int length>class Buffer;  
```

这两个声明是一样的。

最后，非引用的非类型参数在表达式当中总是个`prvalues`，不能获取地址，也不能被赋值。而一个左值引用类型的非类型参数，可以表示一个左值：

```c++
template<int & Counter>
struct LocalIncrement{
	LocalIncrement(){ Counter=Counter+1;}
	~LocalIncrement(){ Counter=Counter-1;}
};
int counter{};  //注意counter声明的范围，此外在函数范围内必须是static的，但是不能是const的
int main()
{
    typedef class X XA;
    class X s;
    XA xa;
    
    {
      	int &rc=counter;
        LocalIncrement<rc> li;
        std::cout<<counter<<std::endl;
    }
    std::cout<<counter<<std::endl;
}
```

#### 12.2.3 Template Template Parameters

TTP是只是占位符，指向类或者别名模板，声明跟类模板类似，然而不能使用`struct`和`union`：

```C++
template<typename Counter>
struct Local{
        Local(){std::cout<<"Local"<<std::endl;}
        void print()
        {
            std::cout<<"print"<<std::endl;
        }
};

template<template<typename X> class C>//OK,class用typename代替也是可以的
void f(C<int>* p)     //更要注意的是这里的C是一个带类型参数的模板类
{
  p->print();
}
template<int arg>   //比较一下
struct NonType{
        NonType(){std::cout<<"NonType"<<std::endl;}
        void print()
        {
            std::cout<<arg<<std::endl;
        }
};
template<template<int arg> typename C>  //注意这里的arg是可以省略的，因为并没有什么用处
void g(C<100> * p)   //g(C<arg>* p)是不对的，arg是未定义的，因为对其引用在上面的C之后就结束了
{
    p->print();
}

int main()
{
  	Local<int> l;
    f(&l);
  	NonType<100> n;
    g(&n);
}
```

上面注释虽然已经说明了，不过还是要强调一下 ：

```c++
template<template<typename T,T*>class Buf> //OK
class Lexer{
  static T* storage;  //编译错误，T的生存期到上面Buf之后的'>'为止，如前所述。
  										//所以实际上TTP中的类型名字有时候是可以省略的，当然这个例子里不能省略。
};
```

#### 12.2.4 Template Parameter Pack

```c++
template<typename... Types>
class Tuple;

using IntTuple=Tuple<int>;
using IntCharTuple=Tuple<int,char>;
using IntTriple=Tuple<int,int,int>;
using EmptyTuple=Tuple<>;

template<typename T,unsigned... Dimensions>
class MultiArray;
using TransformMatrix=MultiArray<double,3,3>;

template<typename T,template<typename,typename>... Containers>
void testCotainers();
```

主类模板，变量模板，以及别名模板最多只能带一个TPP(当然我是故意的)，并且只能作为最后一个模板参数。函数模板就允许多个TPP存在，只要一个TPP之后的模板参数有缺省值或者可以推断出来。不过，部分特化的类模板和变量模板的申明是可以有多个TPP的，道理一样的。

```c++
template<typename... Types,typename Last>
class LastType;     //错误，对一个申明来说这里无从推断

template<typename... TestTypes,typename T>
void runTests(T value);    //T可以推断，从而可以把前面tpp推断出来。

template<unsignd...> struct Tensor;
template<unsigned... Dims1,unsigned... Dims2>
auto compose(Tensor<Dims1...>,Tensor<Dims2...>); //从实际传入的Tensor可以分别推断出Dims1和Dims2

template<typename...> TypeList;
template<typename X,typename Y> struct Zip;
template<typename... Xs,typename... Ys>
struct Zip<TypeList<Xs...>,TypeList<Ys...>>;  //Xs...和Ys...是可以从实际传入的TypeList推断出的
```

强调一下，所谓推断，指的是用实际参数(argument)去推断模板参数(parameter)的类型以及，数量。

TPP本身在其参数子句中是不能再扩展的：

```c++
template<typename... Ts,Ts... vals>  //这是错误的。理解下这里的的意思
struct StaticValues{};  						 //Ts是类型参数，vals是非类型参数
																		 //应该说这里有两个错误，
																		//一个是不能再展开，另一个是只能有一个TPP
```

不过，可以这样：

```c++
template<typename... Ts> struct ArgList
{
  template<Ts... vals> struct Vals{};
};
ArgList<int,char,char>::Vals<3,'x','y'> tada;  //注意下ArgList的模板参数是类型，
																							 //而Vals的参数则是对应的值，或者说常量表达式
																							 //也就是说，Ts是类型参数，vals是非类型参数
```

#### 12.2.5 Default Template Arguments

- 一个缺省参数不能依赖于对应的模板参数。

- 类模板、变量模板、别名模板的缺省参数只有在其之后的模板参数有缺省参数时才能给出

```c++
template<typename T1,typename T2,typename T3,
				 typename T4=char,typename T5=char>
class Quintuple; //OK

template<typename T1,typename T2,typename T3=char,
				 typename T4,typename T5>
class Quintuple; //OK

template<typename T1=char,typename T2,typename T3,
				 typename T4,typename T5>
class Quintuple; //ERROR,T2没有缺省参数
```

- 函数模板参数的 缺省参数不受上条限制

```c++
template<typename R=void,typename T>
R* address(T& value);  //OK
```

- 缺省参数不能重复

```c++
template<typename T=void>
class Value;

template<typename T=void>
class Value;  //Error,重复了
```

- 部分特化不允许使用缺省参数

```c++
template<typename T>
class C;
...
template<typename T=int>
class C<T*>;   //错误
```

- TPP不允许使用缺省参数 

```c++
template<typename... Ts=int> struct X;  //ERROR
```

- 类成员函数在类外部定义时不允许使用缺省参数

```c++
template<typename T>struct X
{
  T f();
};
template<typename T=int> T X<T>::f()  //ERROR
{
  
}
```

- 友元类声明不允许使用缺省参数

```c++
struct S{
  template<typename =void> friend struct F;
};
```

- 对一个友元函数可以使用缺省函数的情况是，是个定义，并且在同一个翻译单元当中，其他地方没有再声明这个函数

```c++
struct S{
  template<typename =void> friend void f(); //error,不是定义
  template<typename =void> friend void g(){}  //OK,到目前为止
};

template<typename> void g();//出错了，同一个文件中已经给了带缺省参数的函数定义，别的地方就不能再声明了
```

### 12.3 Template Arguments

实例化一个模板的时候，模板参数的确定有这么些机制：

- 显式指定模板参数，模板名字跟着用尖括号括起来的模板参数。由此产生的名字叫*template-id*

- Injected class name，这个就不翻译了，总之就是在一个类的定义范围内对这个 类的名字的引用，比方说

```c++
template<typename p1,typename p2,...>  //...只是示意，不表示tpp
class X{
  X<p1,p2,...> v;   //这里的X<p1,p2,...>就是所谓的injected class name
};
```

- 缺省模板参数，这个也不多解释了，只是要住对于一个类模板或者别名模板，即使所有的参数都有缺省值，那个尖括号也是不可少的。

```c++
template<typename P1=p1,typename P2=p2,...>
class X;

X<> x;  //尖括号不可少，等同于X<p1,p2,...>
```

- 参数推断，主要适用于函数模板，类模板只是某些情况下使用。

#### 12.3.1 Function Template Arguments

函数模板参数可以显式指定，可以从函数参数推断，可以使用缺省参数。

```c++
template<typename T>
T max(T a,T b)
{
  return b<a? a:b;
}
int main()
{
  ::max<double>(1.0,-3.0);
  ::max(1.0,-3.0);
  ::max<int>(1.0,-3.0); //上面两个例子不用多说，就本条语句而言，虽然通过函数参数可以推断出float，
  											//但是显式指定的int阻止了这个推断，因此，返回值是int型
}
```

对于无法推断的模板参数，通常放在参数列表的前面，这样可以指定这些参数，而不需要再列出其他可以推断的参数

```c++
template<typename DstT,typename SrcT>
DstT implicit_cast(SrcT const& x)
{
  return x;
}
int main()
{
  double value=implicit_cast<double>(-1);
}
```

这样的参数是不能放在tpp之后的，因为无法去推断或者指定这样的参数，毕竟编译器对模板参数列表的处理是从头到尾，而不会跳到最后去看一看

```c++
template<typename... Ts,int N>
void f(double (&)[N], Ts... ps);

double a[11]={0};
f<int,int,10>(a,10,11);  //没有意义的，无法从参数a推断出N来，N也无法从<int,int,10>里找出来，
												 //因为尖括号里边的参数都被认为是Ts，但又因为存在一个字面量10而出错了
```

因为函数存在重载，所以即使指定所有模板参数也并不足以确定一个正确的函数，有可能会有很多个
```c++
template<typename Func,typename T>
void apply(Func funPtr,T x)
{
  funcPtr(X);
}
template<typename T>void single(T);
template<typename T>void multi(T);
template<typename T>void multi(T*);

int main()
{
  apply(&single<int>,3); //ok
  apply(&multi<int>,7); //不知道是哪个multi，也就无从在apply函数里边调用funcPtr的时候从T来推断
}
```

另外，依据SFINAE，并不需要所有的函数模板对所有的模板参数都有效。

#### 12.3.2 Type Arguments

替换模板类型参数的类型实参必须符合对模板参数的限定：

```c++
template<typename T>
void clear(T p)
{
  *p=0;  //T必须能够进行*运算,换句话说就是p必须是个指针
}
int main()
{
  int a;
  clear(a);  //int不是指针，int*就可以了，所以要clear(&a)才是正确的
}
```

### 12.3.3 Nontype Arguments

- 类型相符的另一个非类型模板参数。
- 编译时可以确定值的整型或者枚举型常量，包括可以不缩紧的隐式转换，如`char`转换成`int`，而500对于一个8位的`char`参数就不是一个合法的参数了。
- `&`(取地址运算符)+外部变量或者函数的名字，适用于指针类型的模板参数。
- 函数和数组名字，`&`(取地址运算符)可以加也可以不加。
- 对于引用类型的模板参数，可以直接使用变量名字或者函数名，不需要加上取地址运算符，加了就是另外一个意义了。
- 成员指针常量，即`&C::m`这样的形式，其中`C`为一个类类型，或者说一个类的名字，不是实例变量，`m`是这个类的非静态成员，包括数据和函数。
- null指针适配于非类型的指针参数或者成员指针参数。

```c++
template<typename T,T nontypeParam>
class C;
C<int,33>* C1;
int a;
C<int *,&a> c2;  //注意T是int *,如果是int，后面参数&a就是不对的，后面&a也不能是a

void f();
void f(int);
C<void (*)(int),f>* c3;  //f(int)

template<typename T> void templ_func();
C<void (),templ_func<double>> *c4;  //这里,下面几种句法都是合法的，templ_func前面可以也可以不加&
																		//C<void(),templ_func<double>> * c4;
    																//C<void (),templ_func<double>> * c5;
    																//C<void (*)(),templ_func<double>> * c6;
    																//C<void(*)(),templ_func<double>> * c7;
struct X
{
  static bool b;
  int n;
  constexpr operator int() const{return 42;}  //类型转换，将一个X转换为int
};

C<bool&, X::b> *c5;//静态成员也是可以接受的变量/函数名
C<int X::*,&X::n>* c6; //注意这两个参数，前一个参数是一个成员指针类型，替换的是类C定义中的T
											 //后一个参数是指向类中某个成员的指针，替换类C定义中的nontypeParam
											 
C<long,X{}> * c7;  //X{}先转换成int，再转换成long，

struct Base
{
  int i;
};
struct Derived:public Base
{

} derived;

C<Base *,&derived>* err1; //不考虑继承,C<Base,derived>同样也是不对的
C<int& ,base.i>* err2;  //变量的字段不被认为是变量
int a[10];
C<int*,&a[0]>* err3;  //数组元素的地址同样不被接受
```

#### 12.3.4 Template Template Arguments

```c++
#include <map>
// 	template<typename Key,typename T,
//					typename Compare=less<Key>,
//					typename Allocator = allocator<pair<Key const,T>>>
//	class map;
#include <array>
// 	template<typename T,size_t N>
//	class array;

template<template<typename...>class TT>  
class AlmostAnyTmpl{};

AlmostAnyTmpl<std::vector> withVector; //std::vector有两个模板参数
AlmostAnyTmpl<std::map> withMap; //std::map有4个模板参数，如上
AlmostAnyTmpl<std::array> withArray;//错误，tpp不匹配非类型模板参数
```

上面`AlmostAnyTmpl`使用TPP是为了避免在c++17之前参数替换时不考虑缺省值的问题。

#### 12.3.5 Equivalence

1. 成员函数模板生成的函数不覆盖虚函数。
2. 构造函数模板生成的构造函数不会被当成拷贝或者移动构造函数，但是可以是缺省构造函数。
3. 赋值函数模板生成的赋值运算符也不会被认为是拷贝或者移动赋值运算符。

### 12.4 Variadic Templates

```c++
template<typename... T>
class Tuple
{
  public:
  	static constexpr std::size_t length=sizeof...(T); //主要看sizeof...的用法
  //注意句法，另外对每一个不同的TPP参数都各自对一应一个不同的类，从而length也各不相同
};
int a1[Tuple<int>::length];
int a3[Tuple<short,int,long>::length];;
```

#### 12.4.1 Pack Expansions

```c++
template<typename... Types>
class MyTuple:public<Types...>{

};
MyTuple<int,float> t2;  //从Tuple<int,float>继承

template<typename ...Types>
class PtrTuple:public<Types*...>{

};
PtrTuple<int,float> t3; //从Tuple<int*,float*>继承
```

体会一下这里的句法。

#### 12.4.2 Where Can Pack Expansions Occur

`sizeof…`并没有真正生成一个list，折叠表达式也没有。

```c++
template<typename... Mixins>
class Point:public Mixins... {  //基类集展开
	double x,y,z;
public:
	Point():Mixins()...{}  //基类初始化集展开
	
	template<typename Visitor>
	void visitMixins(Visitor visitor){
		visitor(static_cast<Mixins&>(*this)...); //调用参数集展开
	}
};

struct Color{ char red,green,blue;};
struct Label{std::string name;};
Point<Color,Label> p; //继承自Color和Label
```

当然，通常不提倡多继承。

这里比较有趣的是`visitMixins`这个函数，我们知道这是所谓的visitor模式，参数展开并没有什么特别可说的，现在的问题是`visitor(static_cast<Mixins&>(*this)…)`的语义到底什么，显然visitor是个Visitor的实例，调用visitMixins的时候，作为一个实参，必然已经做了初始化，所以`visitor(…)`就是一个函数运算符的调用，也就是说Visitor类中定了函数运算符`operator()`。

```c++
template<typename... Ts>
struct Values{
	template<Ts... Vs>
	struct Holder{
	
	};
};

int i;
Values<char,int,int*>::Holder<'a',17,&i> valueHolder;  
```

同样，注意句法。Ts是个可变长模板参数，Vs看起来也是，但是一但Ts确定，Vs中参数的个数以及各自的类型必然也就确定了。

在Values的声明中，`Ts…`的`…`有双重含义，一方面表示这个参数是个模板参数集，另一个又是对这个参数集的展开。

#### 12.4.3 Function Parameter Packs

```c++
template<typename T> void c_style(int,T...);
template<typename T...>void pack(int,T...);
```

对于第一种，`T…`被认为是`T,…`，`c_style`是个变长参数的函数。

对于第二种，`T…`就是个参数集。

事实上第一种的写法可以改成`c_style(int,T,…)`，或者第二种写法也可以改成`pack(int,T… args)`，这样就不会混淆了。

#### 12.4.4 Multiple and Nested Pack Expansions

```c++
template<typename F,typename... Types>
void forwardCopy(F f,Types const& ...values){
	f(Types(values)...);
}
```

类似于

```c++
template<typename F,typename T1,typename T2,typename T3>
void forwardCopy(F f,T1 const &v1,T2 const& v2,T3 const& v3){
	f(T1(v1),T2(v2),T3(v3));
}
```

```c++
template<typename ... OuterTypes>
class Nested{
	template<typename... InnerTypes>
	void f(InnerTypes const&... innerValues){
		g(OuterTypes(InnerTypes(innerValues)...)...); //先用innerValue初始化InnerTypes,
																									//再用InnerTypes初始化OuterTypes
	}
};
```

类似于

```c++
template<typename O1,typename O2>
class Nested{
	template<typenae I1,typename I2,typename I3>
	void f(I1 const& iv1,I2 const& iv2,I3 const& iv3){
		g(O1(I1(iv1),I2(iv2),I3(iv3)),
		  O2(I1(iv1),I2(iv2),I3(iv3)));  //注意不是g(O1(I1(iv1),
		  															 //			    O2(I2(iv2)));
	}
}
```

#### 12.4.5 Zero-Length Pack Expansions

零长度的TPP，就等于不存在了。

```c++
template<typename T,typename... Types>
void g(Types... values){
	T v(values);   //如果Types的长度为0，则这里就是 T v;而不是T v();
}
```

另外

```c++
template<>
class Point:{
	Point():{}
};
```

编译器是不会这样处理的，而是会变成

```c++
class Point{
	Point(){}
};
```

#### 12.4.6 Fold Expression

`()`是必须的。

对于`(pack op ...)`和`(... op pack)`，如果`…`长度为0，那么

- 如果`op`是`&&`，则返回`true`
- 如果`op`是`||`，则返回`false`
- 如果`op`是`,`，则返回`void`

而对于`(pack op … op value)`和`(value op … op pack)`，pack为空也可以通过value获得值

### 12.5 Friends

一个友元声明，就相当于在一个实体上打了一个洞或者开了一扇窗，或者说，是个内鬼。

- 友元声明可以是一个实体的唯一声明
- 友元函数声明可以是一个定义。
- 友元类声明不能是定义。

#### 12.5.1 Friend Classes of Class Template

```c++
template<typename T>
class Tree{
  friend class Factory; //ok,到此处Factory可以是第一次出现
  											//也就是说可以不用在Tree之前出现Factory的定义或者声明
  friend class Node<T>; //如果到此处为止，之前没有定义或者声明Node，则错误
  template<typename> friend class Tree; //任意的Tree<T2>都是Tree<T>的友元类
};
```

这样也是可以的

```c++
template<typename T>
class Wrap
{
  friend T;  
};
```

不过如果T不是一个类类型，就会被简单忽略掉。

#### 12.5.2 Friend Functions of Class Templates

```c++
template<typename T1,typename T2>
void combine(T1,T2);

class Mixer{
  friend void combine<>(int&,int&); //OK
  friend void combine<int,int>(int,int); //OK
  friend void combine<char>(char,int);  //OK
  friend void combine<char>(char&,int); //ERROR
  freind void combine<>(long,long){...} //ERROR,此处不允许定义模板实例，注意是模板函数实例
  																			//实际上，这条目前编译是可以通过的，但是链接时出错了
};
```

- 如果一个名字不是限定的，也就是不包含`::`，那么它就不能指向一个模板实例。如果在友元声明处尚无可见的可匹配非模板函数，所谓可见就不解释了，那么这个友元声明就是这个函数的首次声明，这个声明也可以是定义。

- 如果一个名字是限定的，也就是包含`::`，这个名字就必须指向一个已经声明了的函数或者函数模板，并且函数优先。但是这样的声明不能是定义。

```c++
void multiply(void *);  //普通函数
template<typename T>    //函数模板
void multiply(T);

class Comrades{
  friend void multiply(int){}  //定义一个新函数::multiply(int),注意是全局范围的
  friend void ::multiply(void*); //指向普通函数，而不是一个模板实例
  friend void ::multiply(int); //指向一个函数模板实例
  friend void ::multiply<double*>(double*);  //模板实例，
  friend void ::error(){}  //Error，不能定义一个限定的友元
};
```

上面是在普通类中的友元声明，类模板也是可以的

```c++
template <typename T>
class Node{
  Node<T>* allocate();
};

template<typename T>
class List{
  friend Node<T>* Node<T>::allocate();
};
```

在类模板中定义的友元函数，只有在真正被调用时才会实例化，而这个实例函数实际上是名字空间范围的。

```c++
template<typename T>
class Creator{
  friend void feed(Creator<T>){} //每一个T定义一个不同的函数::feed()
};

int main()
{
  Creator<void> one;
  feed(one);  //::feed(Creator<void>)
  Creator<double> two;
  feed(two);  //::feed(Creator<two>)
}
```

要注意的是，尽管这些函数是模板实例化的一部分，但他们不是模板的一部分，而是普通函数。不过他们也被称为模板实体，并且只是在用到的时候才会生成。而且由于他们的定义在类定义的内部，因而是隐含inline的，因此，在两个不同的翻译单元当中生成同一个函数也不是错误。

#### 12.5.3 Friend Templates

```c++
class Manager{
  template<typename T>
  friend class Task;
  
  template<typename T>
  friend void Schdule<T>::dispatch(Task<T>*);
  
  template<typename T>  //同样可以是定义
  friend int ticket(){   
    return ++Manager::counter;
  }
  static int counter;  //必须在某个地方初始化，或者加上inline或者const或者constexpr
};
```

一个友元模板只能声明为主模板或者主模板的成员，与这个主模板关联的部分特化模板或者显式特化都被自动认为是友元。

## Chapter 13 Names in Templates

- 模板出现时的上下文
- 模板实例化时的上下文
- 模板实例化时与参数相关的上下文

### 13.1 Name Taxonomy

- 如果一个名字显式的被范围解析运算符(`::`)或者成员访问运算符(`.`或者`->`)指定，则称为限定名字。比如`this->count`就是个限定名字，但是其中的`count`不是
- 如果一个名字依赖于某种形式的模板参数，则称为依赖名字，比方说`std::vector<T>::iterator`，这里如果T是个模板参数。但是如果T是个已知类型的别名，比方说`using T=int`，则这个表达式就不是了。

### 13.2 Looking Up Names

#### 13.2.1 Argument-Dependent Lookup

ADL主要应用于非限定名字，在函数调用或者运算符调用时，这些名字(函数名、运算符)是个非成员函数的形式。所谓ADL，就是一个非限定的名字如果跟着括起来的一个参数表达式列表，则会去查找与这些参数类型相关的名字空间或者类。比如一个指向类`X`的指针，则类`X`以及`X`所属的名字空间或者类就被视为相关的。

- 对内置类型，为空集
- 对指针和数组类型，为其底层类型
- 对枚举类型，为其声明所在名字空间
- 对类成员，其所属的类
- 对类类型，包括联合类型，相关类包括这个类本身，其所属的类，直接或者间接的基类。相关名字空间指相关类声明所在名字空间。
- 如果一个类是个模板类实例，则包括其参数的类型，以及这些参数的声明所属的类以及名字空间
- 对函数类型，与参数和返回值相关的类及名字空间
- 类成员指针，包括与这个类相关的集合，以及与这个成员相关的集合

#### 13.2.2 Argument-Dependent Lookup of Friend Declarations

一个友元函数声明可能是备选函数的首次声明，这时候就认为这个函数所在的名字空间是包含其声明的类所在的最接近的名字空间。然而，在这个范围内，这个友元函数声明并不直接可见。

```c++
template<typename T>
class C{
  friend void f();
  friend void f(C<T> const&);
};

void g(C<int>* p)
{
  f();  //f()可见？
  f(*p); //f(C<int> const&)可见？
}
```

所以，除非之前实例化过类C，否则对`f()`的调用就会编译出错，注意，这是对首次声明而言。

- 因为`f()`没有参数，也就无从寻找其相关的类或者名字空间，从而也就找不到`f()`的定义或者声明。
- 而对于`f(*p)`，确实可以认为其关联的类是`C<int>`，其关联的名字空间是全局，因此只要此前实例化过`C<int>`，那么`f(*)`就是可见的。实际上，在这种情况下，关联类已经被编译器假定为实例化了。

#### 13.2.3 Injected Class Names

就是在一个类的定义中用引用自己的名字，或者说就是这个类的别名。

```c++
template<template<typename> class TT> class X {};
template<typename T> class C{
  C* a;//Ok,C<T>* a;
  C<void>& b; //ok
  X<C> c; //ok,没有模板参数列表就代表这个模板类本身，也就是等同于X<C<T>> c;
  X<::C> d;  //::C本身并不是注入类名，但仍然代表了这个模板
};
template<int I,typename... T>class V{
  V* a; //ok,等同于V<I,T...>* a;
  V<0,void> b; //ok
};
```

#### 13.2.4 Current Instantiations

```c++
template<typename T> class Node{
  using Type=T;
  Node* next;  //当前实例
  Node<Type>* previous; //Node<Type>当前实例
  Node<T*> * parent;  //未知实例，依赖于对T*的实例化，T*与T是不同的。
};
```

```c++
template<typename T>class C{
  using Type=T;
  
  struct I{
    C* c;
    C<Type>* c2;
    I *i;
  };
  struct J{
    C* c;
    C<Type>* c2;
    I* i;;
    J* j;
  };
};
```

对其中的`C::I`我们可以做另外一个显式特化

```c++
template<> struct C<int>::I{
  
};
```

这样，在`C::J`的定义中，`C::J::I`就是一个未知的实例了。

### 13.3 Parsing Templates

#### 13.3.2 Dependent Names of Types

```c++
template<typename T>
struct S: typename X<T>::Base{
  S():typename X<T>::Base(typename X<T>::Base(0)){
    
  }
  ...
};
```

其他的都好理解，第三行的第二个`typename`表明这里不是用0来初始化一个临时的`X<T>::Base`对象，而是把0转换成一个`X<T>::Base`对象，当然，也可以说因为加了`typename`，我们或者编译器才会认为这里是个类型转换，反过来，如果没有加`typename`，那么这里就是一个对象的初始化。

除非是禁止使用`typename`的地方，如果不清楚的时候，并且你确信需要一个类型的时候，加上`typename`总是没错的，至少阅读代码的时候能清楚你的目的。

#### 13.3.3 Dependent Name of Templates

```c++
template<typename T>
class Shell{
  public:
  T v1;
  template<int N>
  class In{
    public:
    int v2;
    template<int M>
    class Deep{
      public:
      int v3;
      virtual void f();
    };
  };
};
template<typename,int N>
class Weird{
  public:
  void case1(typename Shell<T>::template In<N>::template Deep<N>* p){
    p->template Deep<N>::f();
  }
  void case2(typename Shell<T>::template In<N>::template Deep<N>& p){
    p.template Deep<N>::f();
  }
};
```

这里template，`::`，`->`，以及`.`的用法都好理解，那么对于Shell这样一个类，我们怎样去访问其中的比方说`v1`，`v2`，以及`v3`？实际上这个问题并不是问题，因为类声明并不分配空间，只能去访问一个类实例，而不是一个类声明，在Shell中嵌入定义的两个类并没有实例，或者说并没有变量指向这两个类，所以是不能直接访问其中的`v2`，`v3`的，除非这两个变量是`static`的。当然，只要进行了初始化，比方说`Shell<int>::In<3> in`，这样就可以访问`in.v2`了，`v3`也类似。

当然，我们也可以改成这样：

```c++
#include <iostream>
template<typename T,int I,int D>
class Shell{
  public:
  T v1;
  template<int N>
  class In{
    public:
    int v2;
    template<int M>
    class Deep{
      public:
      int v3; 
      virtual void f(){ 
        std::cout<<v3<<std::endl;
      };
    };
    Deep<D> deep;
  };
  In<I> in;
};
template<typename T,int N>
class Weird{
  public:
  void case1(typename Shell<T,N,N>::template In<N>::template Deep<N>* p){
    p->template Deep<N>::f();
  }
  void case2(typename Shell<T,N,N>::template In<N>::template Deep<N>& p){
    p.template Deep<N>::f();
  }
};
int main()
{   
    Shell<int,3,3> shell;
    shell.v1=1;
    shell.in.v2=2;
    shell.in.deep.v3=3;
    Weird<int,3> weird;
    weird.case1(&shell.in.deep);
    shell.in.deep.v3=4;
    weird.case2(shell.in.deep);
}
```

#### 13.3.4 Dependent Names in Using Declarations

```c++
template<typename T>
class BXT{
  public:
  	using Mystery=T;
  	template<typename U>
  	struct Magic;
};

template<typename T>
class DXTT:private BXT<T>{
  public:
  	using typename BXT<T>::Mystery;  //引入一个类型
  	Mystery *p;  
};
//错误的例子
template<typename T>
class DXTM:private BXT<T>{
  public:
  	using BXT<T>::template Magic;  //错误，标准没有
  	Magic<T> * plink;   //所以这也是错的
};
//正确的例子
template<typename T>
class DXTM:private BXT<T>{
  public:
  	template<typename U>
  	using Magic=typename BXT<T>::template Magic<T>;
  	Magic<T>* plink;
};
```

上面例子只是对类模板有效，函数模板标准并未涉及。

另外，可以看到，前一节的例子用using也是可以解决的。

#### 13.3.5 ADL and EXPlicit Template Arguments

```c++
namespace N{
  class X{
    
  };
  template<int I> void select(X*);
}
void g(N::X* xp)
{
  select<3>(xp);  //ERROR:no ADL
}
```

出错的原因是，编译器只有知道`<3>`是个模板参数列表，才能决定`xp`是个函数调用参数，换句话说就是`select`是个函数调用，反过来，编译器只有知道`select()`是个模板，才能知道`<3>`是个模板参数列表。

这个问题的解决可以在`g()`的定义前加上一句`template<typename T>void select();`来解决，这样可以保证`select<3>`可以被当做模板id来处理。

当然，如果不是考虑ADL，在`g()`中调用这样`N::select<3>()`也是可以的。

#### 13.3.6 Dependent Expressions

```c++
template<typename T> void typeDependent1(T x)
{
  x;  //类型依赖的表达式，因为x的类型是不定的，指在这个模板里
}
template<typename T>void typeDependent2(T x)
{
  f(x); //类型依赖的表达式，因为x是类型依赖的
}
```

```C++
template<int N>void valueDependent1()
{
  N;  //值依赖的表达式，不是类型依赖的表达式，因为N的类型是确定，只是值不确定，指在这个模板里
}
```

```c++
template<typename T> void valueDependent2(T x)
{
  sizeof(x);  //值依赖的表达式，而不是类型依赖，因为sizeof的返回值类型是确定的，但是值可能会变
  						//像sizeof这样的运算，可以把一个类型依赖的操作数转换成一个值依赖的表达式
}
```

```c++
template<typename T> void maybeDependent(T const&x)
{
  sizeof(sizeof(x));  //实例化依赖的表达式，但既不是值依赖也不是类型依赖，
  										//因为外层sizeof总是返回size_t的大小
  										//另外，T必须是个完整类型，不然sizeof会报错
}
```

但是要注意的是，类型依赖是值依赖的子集，值依赖是实例化依赖的子集，实例化依赖是表达式的子集。这里所谓的依赖指的是对模板参数的依赖。

### 13.4 Inheritance and Class Template

#### 13.4.1 Nondependent Base Classes

~~~c++
template<typename X>  //注意，是X
class Base{
  public:
  	int basefield;
  	using T=int;
};

class D1:public Base<Base<void>>{  //实际上与模板无关
  publice:
  	void f(){ basefield=3; } //对继承成员的常规访问
};

template<typename T>
class D2:public Base<double>{  //无依赖基类
	public:
  	void f(){ basefield=3; } //对继承成员的常规访问
  	T strange;  //Base<double>::T，不是模板参数T
};
~~~

这里需要说明的是，一个无限定的名字，在查找时，基类中的名字优先于模板参数，当然这里说的是基类是无依赖的情况，也就是基类与继承类的模板参数无关的时候。

所以

```c++
void g(D2<int*>& d2,int *p)
{
  d2.strange=p;  
}
```

是不对的，`d2.strange`与`p`的类型是不匹配的。

所以，写继承类的时候必须注意基类的名字。

#### Dependent Base Classes

```c++
template<typename T>
class DD:public Base<T>{  //依赖性基类，虽然这个模板参数在Base的定义中并没有实际用处
  public:
  	void f(){ basefield=0;}  //#1
};
template<>
class Base<bool>{
	public:
		enum{ basefield=42;}  //#2
};

void g(DD<bool>& d)
{
  d.f(); //#3 oops?
}
```

正常的编译器将会提示:

>error: use of undeclared identifier 'basefield'
>        void f(){ basefield=0;}

原因在于c++标准说无依赖的名字不会去依赖性基类中查找，注意，是不去有依赖性的基类中查找，而不是说碰上了这个名字就不管了，还是要去找的，能不能找得到是另外一回事，就如上面例子。

这也是所谓两阶段查找的例子：第一阶段是首次看到模板定义时，第二阶段是在模板实例化时。

所以，可以这样解决这个问题

```c++
template<typename T>
class DD1:publice Base<T>{
  public:
  	void f(){this->basefield=0;}  //延迟查找
};
```

或者

```c++
template<typename T>
class DD2:public Base<T>{
  public:
  	void f(){Base<T>::basefield=0;}
};
```

或者

```c++
template<typename T>
class DD3:public Base<T>{
  using Base<T>::basefield;
};
```

但是，必须要说，这里只是解决了之前编译错误找不到变量的问题，但是就上面函数`g()`而言，还是错误的，因为这时候会变成向一个不可赋值表达式赋值

>error: expression is not assignable
       void f(){ this->basefield=0;}

或者

>error: expression is not assignable
>        void f(){ Base<T>::basefield=0;}

或者

>error: expression is not assignable
>        void f(){ basefield=0;}

当然，这是另外一个问题了，或者准确的说是`Base<bool>`的定义的问题。

```c++
template<typename T>
class B{
  public:
  	enum E {e1=6,e2=28,e3=496};
  	virtual void zero(E e=e1){}
  	virtual void one(E&){}
};
template<typename T>
class D:public B<T>{
  public:
  	f()
    {
      typename D<T>::E e; //this->E是不合法的，这里B<T>::E也是可以的，
      										//因为这里是单继承，在多继承的情况下就要看情况而定了
      this->zero();  //继承的虚函数
      one(e);  //这是个依赖性函数，因为其参数是依赖性的
    }
};
```

就上面程序而言其实还有个问题，就是对one的调用，要么加上`this->`或者`B<T>::`或者`D<T>::`之类的限定，这样才会在基类中寻找one的定义，要么在D中定义一个one函数，而此时的函数签名应该是`void one(typename D<T>::E&)`，当然，其中的`D<T>`改成`B<T>`也是可以的。

在当前实例中寻找一个限定名字时，c++标准指定首先在当前实例中查找这个名字，然后在所有非依赖性基类中查找。如果找到了，那么这个名字就指向当前实例的一个成员，并且是一个非依赖性的名字。如果找不到，并且这个类有依赖性的基类，那么这个限定名字就指向一个未知特化的成员。比如

```c++
class NonDep{
  public:
  	using Type=int;
};

template<typename T>
class Dep{
  public:
  	using OtherType=T;
};

template<typename T>
class DepBase:public NonDep,public Dep<T>{
  public:
  	void f(){
      typename DepBase<T>::Type t;//找到NonDep::Type;
      typename DepBase<T>::OtherType* ot;//找不到，DepBase<T>::OtherType是一个未知特化的成员
      																	 //因为不会去一个依赖性的基类中查找名字
    }
};
```

