# C++ Templates  PartII:Templates in Depth

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
- 所以，如果你需要在程序中改变一个类静态数据成员，就这类定义外部对其初始化，或者，从c++17起，使用`inline`，比方说上面的例子：`inline static bool copyable = false`，这样在函数中就可以对其赋值了。



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



