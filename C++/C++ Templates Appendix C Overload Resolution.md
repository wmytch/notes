# Appendix C Overload Resolution

[TOC]

## C.1 When Does Overload Resolution Kick In

首先，通过函数指针或者成员函数指针调用函数不会进行重载解析。其次，像函数的宏也不会进行重载。

从非常高的层面说，调用一个具名函数的过程大体如下：

- 查找名字，已形成一个初始重载集
- 如果有必要，用不同的方式来调整这个集合，比方说，模板函数推断和替换发生后，有一些函数模板候选就可以丢弃了
- 根本就不匹配的候选，即使考虑了隐含转换或者缺省参数之后仍然不匹配的，会从重载集中移除。结果就是一个称为可行函数候选的集合。
- 进行重载解析以寻找一个最佳候选，如果有，则中选，如果没有，则说明这次调用是不明确的。
- 检查中选函数。例如，如果是一个已删除函数(定义的时候使用了`=delete`)，或者是一个私有成员函数，发出错误提示。

上面每一步都有其微妙之处，而重载解析是最复杂的。不过有一些简单的原则可以澄清大多数情况。

## C.2 Simplified Overload Resolution

通过对调用的每一个实际参数与候选函数的对应的形式参数的匹配程度进行比较，重载解析会对可行候选函数进行排序。对于一个被认为比另一个函数更好的候选，其每一个参数都必须比另一个函数的对应参数更匹配。比方说

```c++
void combine(int,double); 
void combine(long,int);  
int main()
{
    combine(1,2); //不明确
}
```

因为对于`combine(1,2)`这次调用，其第一个参数更匹配第一个函数，而第二个参数更匹配第二个函数。也许会有人说long比double更接近int，所以第二个函数可以选择，但是C++并不会定义接近度。

有了这么一个原则，剩下的就是怎样判断一个实际参数与可行候选的形式参数的匹配程度。我们可以模拟对候选函数的排序：

1. 完美匹配。形式参数的类型是表达式(指实际参数)的类型，或者是指向表达式的类型的引用类型(可能有const或者volatile修饰)。
2. 需要微小调整的匹配。比如数组变量退化为指向其首元素的指针，或者给实际参数`int**`加上const以匹配形式参数`int const* const*`。
3. 需要提升的匹配。除了整型类型外，也包括从float到double的提升。
4. 只需要标准转换的匹配。包括任意的标准转换，比如从int到float，或者一个派生类向其public继承并且没有歧义的基类之一进行的转换，但是不包括对转换操作符或者转换构造函数的隐式调用。
5. 需要用户定义转换的匹配。允许任何形式的隐含转换。
6. 与省略号(…)匹配的转换。一个省略号形式参数可以匹配几乎所有类型。不过，有一个例外，具有非平凡(nontrivial)复制构造函数的类类型，可能有效，也可能无效，取决于编译器实现。

看例子

```c++
int f1(int);  //#1
int f1(double); //#2
f1(4);  //#1中选

int f2(int);  //#3
int f2(char);  //#4
f2(true);  //#3中选，需要提升，而#4需要强标准转换

class X
{
    public:
    	X(int);
};

int f3(X);  //#5
int f3(...); //#6
f3(7);   //#5中选，用户定义转换的匹配，#6是属于没有其他匹配时候的选择
```

注意重载解析发生在模板参数推断之后，而这种推断并不会考虑所有上述所有类型的转换。比如

```c++
template<typename T>
class MyString
{
    public:
    	MyString(T const*);  //转换构造函数
};
template<typename T>
MyString<T> truncate(MyStirng<T> const&,int);

int main()
{
    MyString<char> str1,str2;
    str1=truncate<char>("hello world",5); //OK
    str2=truncate("hello world",5); //Error    
}
```

模板参数推断时不会考虑转换构造函数提供的隐式转换，所以对于str2，找不到可行的truncate函数，根本就不会进行重载解析。

在模板参数推断的上下文中，对模板形参的右值引用，如果其对应的实际参数是个左值，那么这个右值引用会被推断为左值引用(引用坍缩)，如果其对应的实际参数是个右值，才会被推断为右值引用。比如

```c++
template<typename T> void strange(T&&,T&&);
template<typename T> void bizarre(T&&,double&&);

int main()
{
    strange(1.2,3.4);  //OK,T推断为double
    double val=1.2;
    strange(val,val); //OK,T推断为T&
    strange(val,3.4); //Error,推断冲突
    bizarre(val,val); //Error,左值val不匹配double&&
}
```

以上是一些简单原则，接下来进行一些细化说明。

### C.2.1 The Implied Argument for Member Functions

对非静态成员函数的调用有一个隐含的参数，可以通过`*this`来访问。对于一个类MyClass，这个隐含参数的类型通常是`MyClass&`或者`MyClass const&`，各自对应非const和const的成员函数，`MyClass volatile&`或者`MyClass const  volatile&`也是有的，如果成员函数是volatile的话，但这种情况很少见。但是我们知道this是个指针类型，这个特性远远早于C++引入引用，所以把this更改成与`*this`是不可行的，就这么将就吧。

在重载解析中这个隐含参数`*this` 的作用与其他显式参数的作用是一样。大多数情况下这很自然，但有时候可能也会出人意料。

```c++
class BadString
{
    public:
    	BadString(char const*);
    
    	char& operator[](std::size_t);  //#1
    	char const& operator[](std::size_t) const;
    	
    	//隐含转换成null结束的字节串
    	operator char*();  //#2
    	operator char const*();
};

int main()
{
    BadString str("correkt");
    str[5]='c';  //可能会引发重载解析不明确
}
```

看一下上面这段程序，我觉得这里并不能完全说明问题。原书上解释是如果对作为隐含参数的str也就是`*this`做了隐含转换，就会变成一个c类型的字符串，于是`str[5]`就会调用内置的下标操作符，显然书上这里说的不明确并不是有多个同级的候选引发的不明确，而是指隐含的调用了非意图的函数。然而，就这段程序而言，这里并不会发生这样的转换。另外，如果只是把下标操作符的参数改成ptrdiff_t，那么在这种情况下自定义的和内建的下标操作符应该是是同一级，按道理也是会引发不明确的。不过，书上说把转换操作符定义成显式的在任何情况下都是有道理的。

一个可行候选集中可以有静态和非静态的成员。当比较静态成员和非静态成员是，隐含参数(`*this`)就被忽略掉了，因为只有非静态成员才有这个参数。

缺省情况下，`*this`作为非静态成员函数的隐含参数是个左值引用类型，不过C++11引入了使其变成右值引用的句法

```c++
struct S
{
    void f1();		//*this是左值引用
    void f2() &&;  //*this是右值引用
    void f3() &;  //*this是左值引用
};
```

对于`f3()`的后缀，显式的指定`*this`是个左值引用，当然不用这个后缀也是，但是这两种情况并不等同。

```c++
int main()
{
    S().f1();  //OK:一条旧规则允许一个右值S()匹配隐含的非const的左值引用类型*this
    S().f2();  //OK:右值S()匹配右值引用类型*this
    S().f3();  //Error:右值S()不能匹配显式的左值引用类型
}
```

### C.2.2 Refining the Perfect Match

对于一个类型X的实际参数，可以完美匹配四种形式参数：X，X&，X  const&，X&&(X const&&当然也是，但是很罕见)。不过比较常见的是重载两种引用的函数，在C++11之前，就是指下面的情况

```c++
void report(int&); //#1
void report(int const&); //#2

for(int k=0;k<10;++k)
{
    report(k);  //#1
}

report(42); //#2
```

在这里，没有const的版本倾向于左值，而只有带const的版本才能匹配右值。

C++11引入右值引用之后，下面例子说明了另外一个常见的区分两个完美匹配的情形

```c++
struct Value
{
    ...
};

void pass(Value const&);	//#1
void pass(Value&&);		//#2

void g(X&& x)
{
    pass(x);  //#1,x是个左值
    pass(X()); //#2,X()是个右值，prvalue
    pass(std::move(x)); //#2,std::move(x)是个右值，xvalue
}
```

这一次带右值引用的版本被认为更加匹配右值。

注意对于成员函数调用的隐含参数`*this`，上面规则也是适用的

```c++
class Wonder
{
    public:
    	void tick();		//#1
    	void tick() const;  //#2
    	void tack() const; 	//#3
};

void run(Wonder& device)
{
    device.tick();  //#1
    device.tack();  //#3,不存在非const的tack版本
}
```

最后,下面这个例子说明了两个完美匹配也会引发不明确，如果只是通过有没有const来重载的话。

```c++
void report(int);		  //#1
void report(int&); 		  //#2
void report(int const&);  //#3

for(int k=0;k<10;++k)
{
    report(k);  //不明确，#1和#2同等匹配
}

report(42); //不明确，#1和#3同等匹配
```

## C.3 Overloading Details

这里是关于重载解析的一些经常碰到的规则的讨论。

### C.3.1 Prefer Nontemplates or More Specialized Templates

在其它条件相同的情况下，优先选择非模板函数而不是模板实例，不论这个实例是通用模板还是显式特化产生的。

```c++
template<typename T> int f(T);  //#1
void f(int); //#2

int main()
{
    return f(7); //错误，选择了#2，但是#2没有返回值。
}
```

这个例子也说明了重载解析不会考虑返回值。

但是，如果重载解析的其他方面有些许不同，比方说有不同的const和引用修饰符，就会首先使用通用的重载解析。这种情况可能会出乎意料，比方说在16.2.4讨论过的，复制或者移动构造函数的模板版本和非模板版本接受同样的参数。

如果是在两个模板中选择，则优先选择更特别的那个模板。可以参见16.2.2。这条有一个特别的情况，就是如果两个模板的唯一区别就是其中之一有一个尾部参数包，那么没有参数包的就会被认为更特别。可以参见4.1.2。

### C.3.2 Conversion Sequences

通常，一个隐含转换可能是一系列的基本转换，比如

```c++
class Base
{
    public:
    	operator short() const;
};

class Derived:public Base
{};

void count(int);

void process(Derived const& object)
{
    count(object);  
}
```

`count(object)`有效是因为object可以隐式的转换成int，不过这需要好几步：

1. object从Derived const转换成Base const，这是一个glvalue转换，保留了对象的成分或者说身份。
2. 用户定义的从Base const到short的转换。
3. 从short到int的提升。

重载解析有一条重要的原则，如果一个转换序列是另一个转换序列的子序列，那么更短的这条转换序列优先。比如，如果我们有另一个函数`void count(short)`，那么对于`count(object)`的调用就会优先选择这一个，因为不需要上面第三步的从short到int的提升。

### C.3.3 Pointer Conversions

指针和成员指针会有各种特别的标准转换，包括

- 转换成bool类型
- 从任意的指针类型转换成`void*`
- 从派生类指针到基类指针的转换
- 从基类的成员指针到派生类成员指针的转换

尽管这些转换都会导致“只需要标准转换的匹配”，但它们的层级不同，上面的列表就是按从低到高的顺序排列的。

首先，不论是普通指针还是成员指针，向bool类型的转换被认为是最低层级的。比如

```c++
void check(void*);  //#1
void check(bool);   //#2

void rearrange(Matrix* m)
{
    check(m);  //#1
}
```

这里需要说明一下为什么会有指针向bool类型的转换，要记得在C/C++中，非空的对象在bool表达式中是被认为是true的。

其次，向`void*`的转换是被认为低于从派生类指针到基类指针的转换的，并且，如果转换的多个目标类之间存在继承关系，那么向最近的父类的转换优先，比如

```c++
class Interface {...};
class CommonProcesses:public Interface {...};

class Machine:public CommonProcess {...};

char* serialize(Interface*);	  //#1	
char* serialize(CommonProcess*);  //#2

void dump(Machine* machine)
{
    char* buff=serialize(machine); //#2
}
```

这里选择#2也是符合直觉的。

对于成员指针也有一条类似的规则：对于向两个相关的成员指针类型的转换，更接近的那个父类优先。

### C.3.4 Initializer Lists

初始化列表实际参数(用`{}`括起来的初始化器)，可以转换成许多不同类型的形式参数：initializer_list，有用initializer_list构造的类类型，初始化列表的元素可以独立的作为构造函数的参数的类类型，或者其组成可以用初始化列表初始化的聚合类类型。下面是一些例子

```c++
#include <initializer_list>
#include <string>
#include <vector>
#include <complex>
#include <iostream>

void f(std::initializer_list<int>)
{
    std::cout<<"#1\n";
}
void f(std::initializer_list<std::string>)
{
    std::cout<<"#2\n";
}
void g(std::vector<int> const& vec)
{
    std::cout<<"#3\n";
}
void h(std::complex<double> const& cmplx)
{
    std::cout<<"#4\n";
}
struct Point
{
    int x,y;
};
void i(Point const& pt)
{
    std::cout<<"#5\n";
}
int main()
{
    f({1,2,3});  //#1
    f({"hello","initializer","list"});  //#2
    g({1,1,2,3,5});  //#3
    h({1.5,2.5});  //#4
    i({1,2});  //#5
}
```



