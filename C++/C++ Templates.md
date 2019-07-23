# C++ Templates

[TOC]

## Chapter 1 Function Templates

### 1.1 A First Look at Functiion Templates

- a function template represents a family of functions
- 模板函数与普通函数的区别在于一些元素是未定义的，也就是参数化的

#### 1.1.1 Defining the Template

- 关于“even if the two values are equivalent but not equal”，意思就是两个值是`<=`而不是`==`。因此

  ~~~c++
  template<typename T>
  T max(T a,T b)
  {
    return b<a? a:b;
  }
  ~~~

  或者

  ~~~c++
  template<typename T>
  T min(T a,T b)
  {
    return a<b? a:b;
  }
  ~~~

  采用这样的比较顺序，对max函数来说，如果`a==b`,那么返回的必定是第二个参数`b`，同样的对于min函数，如果`a==b`,返回的也是第二个参数`b`。总的来说这是一种很微妙的处理，只有到了一定的深度才会需要去考虑这个问题。

- 模板定义时不能用struct代替typename，struct没有作为模板参数类型限定符的语义。

####　1.1.2 Using the Template

- 使用`::max()`确保调用的是在全局名字空间所定义的这个max函数，而不是`std::max()`。

- 函数模板只是一个编译器生成函数的模板，不同的参数会生成不同的函数。

- 虽然“one-entity-fits-all”也是可接受的，但实际上并没有编译器这么做，因为这会影响运行时的效率。

- instance和instantiate的语义对于类和模板来说是不一样的，类实例指的是一个具体的对象，而模板实例则指的是一个函数或者类的定义。

- `void`也是合法的模板参数，比如

  ~~~c++
  template <typename T>
  T foo(T* )
  {
  }
  void *vp=nullptr;
  foo(vp);
  
  ~~~

  当然，`void*`也是合法的:

  ~~~c++
  template <typename T>
  T foo(T a)
  {
  }
  void *vp=nullptr;
  void *re=nullptr;
  re=foo(vp);
  ~~~

  #### 1.1.3 Two-Phase Translation

  1. 忽略模板参数，检查模板定义本身，包括句法错误比如缺少分号，未知的名字，以及不依赖于模板参数的断言。
  2. 加上模板参数，再次检查定义，也就意味着依赖模板参数的部分会被检查两次。
  3. 两阶段翻译会引发一个问题，假如在使用一个函数模板的过程中又触发了对这个模板的实例化操作，这个时候编译器就需要查看这个模板的定义，这就打破了编译和链接的界限，因为对于通常的函数而言，一个函数的声明已经提供足够的信息用来编译了。这也是个比较微妙的问题，暂时存疑。

### 1.2 Template Argument Deduction

#### Type Conversions During Type Deduction

1. 所谓在用引用声明函数参数时，对同一个参数T，传入的参数必须完全匹配。

~~~c++
template <typename T>
T max(T &a,T &b)
~~~

那么在调用时这个函数时，有这么几种情况：

~~~c++
int i,j;
i=1;j=2;
int &a=i;
int &b=j;
const int k=3;
const int l=4;
const int &s=i;
const int &t=j;
const int &x=k;
const int &y=l;
~~~

| 调用函数 | 结果 |
| :------: | :--: |
| max(1,2) | ❌ |
|max(1,j)|❌|
|max(1,b)|❌|
|max(1,l)|❌|
|max(1,t)|❌|
|max(1,y)|❌|
|max(i,j)|✅|
|max(a,b)|✅|
|max(k,l)|✅|
|max(s,t)|✅|
|max(x,y)|✅|
|max(i,b)|✅|
|max(i,l)|❌|
|max(i,t)|❌|
|mas(i,y)|❌|
|max(a,l)|❌|
|max(a,t)|❌|
|max(a,y)|❌|
|max(k,t)|✅|
|max(k,y)|✅|
|max(s,y)|✅|

可以总结一下：

* 不能传入字面量做参数，虽然错误各有不同，但是这个错误提示不要当真，总之就是参数不匹配。

* 同样类型的参数，不论本身是否引用，都是匹配的。

* const与非const是不匹配的，不论是否引用。

* 以上同样适用于volatile。

基于以上几条，还可以再做一个总结，就是所谓匹配，只是指字面量，普通变量，const变量，volatile变量之间的匹配，与传入参数是否为引用无关，当然，这里讨论是基于模板参数为引用的情况。

2. 对于模板参数声明为值的情况，只做所谓的退化，也就是忽略const和volatile，引用转换成被引用类型，裸数组和函数转换成相应的指针，然后退化之后的参数必须匹配。但是并不做常规的自动类型转换，比方说int和float，string和char[]，因为编译器并不知道应该向哪个方向转换。

#### Type Deduction for Default Arguments

对于函数的缺省参数，是不能做办法类型推断的，因为不能对不存在的东西做推断，要记住，类型推断是在编译时做的。

~~~C++
template<typename T>
void f(T = "")

f(1); //没问题，T是int，调用f<int>(1)
f();  //无从推断T是什么类型,可以这样理解，这时候T是一个string呢还是一个char *？或者const char *?
~~~

当然这个问题可以这样解决：

~~~C++
template<typename T = std::string>
void f(T = "")

f(); //没问题，没有推断，只是取了缺省的类型
~~~

### 1.3 Multiple Template Parameters

- template<typename T> //T是template parameter

- T max(T a,T b)  //a和b是call parameters

- 考虑这种情况
~~~C++
template <typename T1,typename T2>
T1 max(T1 a,T2 b)  //注意返回值类型是T1
{
    return b<a?a:b;
}

auto m=::max(42,66.6); //m=66
auto n=::max(66.6,42); //n=66.6
~~~

#### 1.3.1 Template Parameters for Return Types

比较

```C++
template <typename T1,typename T2,typename RT>
RT max(T1 a,T2 b)
...
::max<int,double,double>(4,7.2);
```

与

~~~C++
template <typename RT，typename T1,typename T2>
RT max(T1 a,T2 b)
...
::max<double>(4,7.2);
~~~

都是合法的，但是显然第二种方式更简洁。

#### 1.3.2 Deducing the Return Type

对于C++14，这样就可以了

~~~C++
template <typename T1,typename T2>
auto max(T1 a,T2 b)
{
  return b<a?a:b;
}
~~~

其他的方式就忘了他们吧，毕竟C++14现在至少gcc是支持的。

有一个比较精妙的问题是auto作为一个返回类型时总是会退化的，跟下面代码一样：

~~~C++
int i=42；
int const & ir=i;
auto a=ir;  //a的类型是int，而不是int &；
~~~

#### 1.3.3 Return Types as Common Type

~~~C++
#include <type_traits>

template<typename T1,typename T2>
std::common_type_t<T1,T2> max(T1,T2)
{
  return b<a?a:b;
}
~~~

这是C++14之后的版本支持，C++11则需要

`typename std::common_type<T1,T2>::type`

### 1.4 Default Template Arguments

这个没什么可说的，记得模板参数是个类型，不是变量，另外后面出现的参数可以引用前面出现的参数。比如

~~~C++
#include <type_traits>

template<typename T1,typename T2,
				typename RT=std::decay_t<decltype(true?T1():T2())>>
RT max (T1,a,T2 b)
{
       return b<a?a:b;   
}
~~~

### 1.5 Overloading Function Template

~~~C++
int max(int a,int b)
{
  ...
}
template <typename T>
T max(T a,T b)
{
  ...
}

int main()
{
  ...
  max<>(7,42);  //限定必须调用模板函数，T通过推断得出为int
  max<double>(7,42); //不需要推断，已经为double
  max('a',42.7); //普通版函数，类型自动转换为int
}
~~~



~~~C++
template <typename T1,typename T2>
auto max(T1 a,T2 b)
{
  ...
}

template <typename RT,typename T1,typename T2>
RT max(T1 a,T2 b)
{
  ...
}

...
auto a=::max(4,7.2);  //T1 int ,T2 double，不存在第三个模板参数.第一个版本最匹配
auto a=::max<long double>(7.2,4);  //RT=long double,T1 double，T2 int
auto c=::max<int>(4,7.2); //T1 int T2 double,模板参数可以匹配RT，也可以匹配T1,两个版本都能匹配
auto d=::max<int>(7.2,4);  //这样就没问题了，RT=int T1=double T2=int
auto e=::max<double>(7.2,4); //T1=double,T2=int 模板参数可以匹配RT，也可以匹配T1
auto f=::max<double>(4,7.2); //RT=double T1=int T2=double
~~~

## Chapter2 Class Templates
### 2.1 Implementation of Class Template Stack
#### 2.1.1 Declaration of Class Templates

- 在一个类模板内，可以直接使用这个类的名字而不带上模板参数，与带上模板参数的类名是等价的，也就是说都可以用：

  ~~~C++
  template<typename T>
  class Stack{
  	Stack (Stack const&);
    Stack& operator=(Stack const&);
  };
  ~~~

  与

  ~~~C++
  template<typename T>
  class Stack{
  	Stack<T> (Stack<T> const&);
    Stack<T>& operator=(Stack<T> const&);
  };
  ~~~

  是等价的，但是<T>这样的表达通常表示一些对特殊参数的特殊处理，所以还是用第一种方式比较好。

- 模板声明必须是全局的。

### 2.2 Use of Class Template Stack

- 只有被调用的模板(成员)函数才会生成代码
- 另外要记住的是，类声明或者说定义本身并不占用内存，只有类实例才会分配内存。

- 因此，同样的，模板定义本身并不分配内存，模板只是编译器在生成类时的参照
- 另外，如果类模板有静态成员，对于不同的类，也就是不同模板参数所分别生成的类，这些成员分别只会生成一次，不会多也不会少。
- 实例化的类模板类型，跟通常的类型是一样的。

### 2.3 Partial Usage of Class Template

- 类模板定义的操作并不依赖于模板参数
- 反过来，模板参数也并不需要都实现类模板里定义的操作
- 意思就是，定义类模板时，只需要考虑这个类模板要做什么，并不需要考虑模板参数能提供什么或者有什么操作
- 反过来，实例化一个类模板时，只要不去调用模板参数不支持的操作，就是没问题的
- 因为，正如上面所说，只有在调用一个函数时，才会生成代码，没有调用就不会涉及。

### 2.4 Friends

~~~C++
template<typename T>
class Stack{
  ...
  void printOn(std::ostream& strm) const{
    ...
  }
  friend std::ostream& operator<<(std::ostream& strm,Stack<T> const& s){
    s.printOn(strm);
    return strm;
  }
};
~~~

~~~C++
template<typename T>
class Stack{
  ...
  template<typename U>
  friend std::ostream& operator<<(std::ostream&,Stack<U> const&);  
/* 或者 
	template<typename T>
	friend std::ostream& operator<<(std::ostream&,Stack<T> const&);
	或者
	friend std::ostream& operator<<(std::ostream&,Stack<T> const&);
	都是可以的
	*/
};
~~~

~~~C++
template<typename T>
class Stack;
template<typename T>
std::ostream& operator<<(std::ostream&,Stack<T> const&);
...
template<typename T>
class Stack{
  ...
  friend std::ostram& operator<< <T>(std::ostream&,Stack<T> const&);//注意这里的<T>,如果没有这个，那么我们就是在这里又声明了一个新的非模板函数，有了这个，指的就是上面声明的那个函数。
};
~~~

### 2.5 Specializations of Class Templates

~~~C++
template<>
class Stack<std::string>{
private:
	std::deque<std::string> elems;
	...
public:
	void push(std::strig const&);
	...
};

void Stack<std::string>::push(std::string const &elem)
{
  elems.push_back(elem);
}
~~~

没有模板参数，原类模板中的T由具体类型代替，因此也不需要`template<typename T>`表达式。

当然，重新写一个类也不是不可以，但是这就与引入模板的初衷相违了，也就是不那么泛型了。

### 2.6 Partial Specialization

~~~C++
template<typename T1,typename T2>
clas MyClass{
...
};
~~~
~~~C++
template<typename T>
clas MyClass<T,T>{
...
};
~~~
~~~C++
template<typename T>
clas MyClass<T,int>{
...
};
~~~
~~~C++
template<typename T1,typename T2>
clas MyClass<T1*,T2*>{
...
};
~~~

还是那句话，如果重新写一个类，就违反引入模板的初衷了。

这些是正确的特化：

~~~C++
MyClass<int,float> mif;//MyClass<T1,T2>
MyClass<float,float> mff;//MyClass<T,T>
MyClass<float,int> mfi;//MyClass<T,int>
MyClass<int*,float*> mp;//MyClass<T1*,T2*>
~~~

这些是引发不明确的特化：

~~~C++
MyClass<int,int> m; //MyClass<T,T>?MyClass<T,int>?
MyClass<int*,int*> m; //MyClass<T,T>?MyClass<T1*,T2*>?。不过可以通过提供这么一个声明：
											//template<T>
											//class MyClass<T*,T*>{...};
											//来解决这个问题
~~~

事实上，我们还可以看到，特化并不以templalte<...>表达式中的参数个数为依据。

### 2.8 Type Alias

#### Typedefs and Declarations

- typedef-name

  ~~~C++
  typedef Stack<int> IntStack;
  ~~~

- alias declaration

  ~~~C++
  using IntStack=Stack<int>
  ~~~

#### Alias Templates
~~~C++
template<typename T>
using DequeStack=Stack<T,std::deque<T>>;
~~~

这里不能用typedef，因为不能用template<typename T>修饰typedef。所以只能这样

~~~C++
template<typename T>
using DequeStack=Stack<T,std::deque<T>>;
typedef Stack<int,std::deque<int>> DS;
int main()
{
    DequeStack<int> ds1;
    DS ds2;  //这里不能用DS<int>或者类似的其他表达式
    return 0;
}
~~~

换句话说，using除了可以在模板内引入一个别名之外，还可以在模板外使用，这种用法生成的名字被称为**别名模板**，注意，只是一个别名，并没有生成新的模板定义，所以`DequeStack<int>`和`Stack<int,std::deque<int>>`指的是同样的一个东西。

#### Alias Templates for Member Types

对于

~~~C++
template<typename T>struct MyType{
  typedef ... iterator;
};
~~~

或者

~~~C++
template<typename T>struct MyType{
	using iterator=...
};
~~~

之后我们就可以这样：

~~~C++
template<typename T>
using MyTypeIterator=typename MyType<T>::iterator;

MyTypeIterator<int> pos;  //与typename MyType<T>::iterator pos;等价的
~~~

注意上面的typename是必须的，因为我们定义的确实是类型而不是变量。跟上面一样，在这里是不能使用typedef来引入一个别名的。

#### Type Traits Suffix _t

c++14之后，标准库定义了这么一个类似的玩意：

~~~C++
namespace std{
  template<typename T> using add_const_t = typename add_const<T>::type;
}
~~~

所以，从C++14开始，我们就可以用`std::add_const_t<T>`这样的表达式来代替`typename std::add_const<T>::type`。

### 2.9 Class Template Argument Deduction
对于C++17，我们可以这样：
~~~C++
Stack<int> intStack1;
Stack<int> intStack2=intStack1;  
Stack intStack3=intStack1;  //C++17,注意这里都是调用拷贝构造函数。
~~~

我们甚至可以通过给构造函数传递一些初值，这样编译器就可以推断参数类型：

~~~C++
template<typename T>
class Stack{
  private:
  	std::vector<T> elems;
  public:
  	Stack ()=default;  //从编程角度而言，Stack(){}也是一样的。这里之所以会出现一个缺省构造函数，没有特别原因，如果代码不需要调用到缺省构造函数，比方说Stack<int> is；这样的，那么不要这个构造函数也是可以的。
  	Stack (T const& elem):elems({elem}){ //注意{elem}的句法，如果没有{}，如果elem是整型，那么就会初始化一个大小为elem的vector,而不是在vector中放进去一个elem元素，如果是其他类型，就会报错，这是容器类的构造函数决定的。
      ...
    }
};
~~~

#### Class Template Arguments Deduction with String Literals

对于上面的代码，如果我们这样声明：

`Stack stringStack=“bottom”;`，那么T就会被推断为char const[7]。为了避免这种情况，我们就需要提供一个传值的构造函数：

```C++
Stack (T elem):elems({std::move(elem)}){
  ...
}
```

因为调用时传值参数会退化，于是就可以推断成`char const*`。这里使用move以避免不必要的拷贝，毕竟elem只是一个临时对象。

#### Deduction Guides

`Stack(char const*) ->Stack<std::string>`称之为推断指引，通常放在类定义之后。

于是`Stack stringStack(“bottom”); `可以推断出`Stack<std::string>`。

然而`Stack stringStack=“bottom”;`虽然可以推断出`Stack<std::string>`,但是这个语句是通不过编译的，因为，这里的“buttom”是个字符串常量，而不是一个std::string，这就与构造函数的参数不符了，这里并不会发生自动类型转换。所以要么强制转换成std::string,要么把推断指引去掉，但是这时候就会推断成Stack<char const [7]>了。

### 2.10 Templatized Aggregates

所谓聚合类，是这样的类或者结构：

- 没有用户自定义的、显式的或者继承来的构造函数
- 没有private或者protected的非静态数据成员
- 没有虚函数
- 没有virtual的，private的或者protected的基类。

这样的类，也是可以模板化的：

~~~C++
template<typename T>
struct ValueWithComment{
  T value;
  std::string comment;
};

...
  
ValueWithComment<int> vc;
vc.value=42;
vc.comment="initial value";
~~~

对于C++17，我们可以通过推断指引来实现下面的语句：

~~~C++
ValueWithComment(char const*,char const*)->ValueWithComment<std::string>;
ValueWithComment vc2={"hello","initial value"};
~~~

如果没有推断指引，这个功能是不能实现的，因为ValueWithComment没有可以用来推断的构造函数。

## Chapter 3 Nontype Template Parameters

### 3.1 Nontype Class Template Parameters

对于非类型模板参数，最好不要提供缺省值。

### 3.2 Nontype Function Template Parameters

~~~C++
template<int Val,typename T>
T addValue(T x)
{
  return x+val;
}
...
std::transform(source.begin(),source.end(),
              dest.begin(),
              addValue<5,int>) ；
~~~

注意一下上面的句法。可以认为，在调用transform函数时，addValue<5,int>做了一个特化，然后把这个临时函数的指针传入了transform中，而不会在transform中进行特化，也就意味着不会在transform函数中对这个函数模板做参数推断，所以这里必须在参数列表中指定这个函数模板的模板参数。另外，这种方式用lamda表达式代替也是可以的。

体会一下下面两种函数模板声明，共同点是都可以从前面出现参数类型推断当前的参数类型，不同点可以参看注释：

~~~C++
template<auto Val,typename T=decltype(Val)>  //保证函数的返回值继承自非类型参数
T foo();  
~~~

~~~C++
template<typename T,T Val=T{}>  //保证传入的值是模板参数的类型
T foo();
~~~

### 3.3 Restrictions for Nontype Template Parameters

- 整型常量,包括枚举

- 指针（对象、函数、成员）

- 左值引用（对象或者函数）

- std::nullptr_t

- 当传递指针或者引用型的模板参数时，所指向的对象不能是字符串文本、临时对象、成员数据或者其他子对象。

- 在C++11，对象应该是外部链接的

- 在C++14，对象可以是内部链接的也可以是外部链接的

- 在C++17，没有链接限制了

~~~C++
template<char const *name>
class Myclass{

};
...
MyClass<"hello"> x; //错误。字符串文本不能做非类型模板参数

extern char const s03[]="hi"; //外部链接.用const就可以在这里对s03做初始化。
char const s11[]="hi";  //内部链接。const可选

int main()
{
  MyClass<s03> m03; //正确(所有版本)
  MyClass<s11> m11; //C++11
  static char const s17[]="hi";  //static 是必须的，const是可选的。同样不要在意编译提示，或者换句话说编译提示里的constant expression指的实际上是静态数据，而不是一个const变量或者说对象
  MyClass<s17> m17;  //C++17
  
}
~~~

#### Avoiding Invalid Expressions

非类型参数可以是任意的编译时表达式：

~~~C++
template<int I,bool B>
class C;
...
C<sizeof(int)+4,sizeof(int)==4> c;
C<42,(sizeof(int)>4)>  //注意第二个参数的括号，没有括号，碰到第一个>就会认为模板参数列表结束，从而会出现句法错误。
~~~

### 3.4 Template Parameter Type auto

~~~C++
template<typename T,auto Maxsize>
class Stack{
	using size_type=deltype(Maxsize);
	std::array<T,Maxsize> elems;   //使用Maxsize的值
	size_type numElems;  //使用Maxsize的类型
	
public：
	Stack():numElems{0}{}
	size_type size() const{   //c++14之后可以直接使用auto
		return numElems;
	};
	void push(T const& elem); //注意这里是引用
	...
};

template<typename T,auto Maxsize>
void Stack<T,Maxsize>::push(T const& elem)
{
  assert(numElems < Maxsize);
  elems[numElems]=elem;  //注意，这里发生了一次拷贝，所以传入一个引用参数是没有问题的
  ++numElems;
}
~~~

注意一下:

~~~C++
Stack<std::string,40> stringStack;

stringStack.push("hello"); //这里hello会被转换为std::string,或者说生成了一个临时的std::string对象
														//另外要注意的是，这里跟前面所说的模板参数不能从字符串文本推断出std::string或者构造函数没有把字符串文本参数自动转换成std::string对象不是一回事。
~~~

另外，前面所说的对非类型参数的限制也同样适用于auto，比方说不能传个浮点数进去。

## Chapter 4 Variadic Templates

### 4.1 Variadic Template

#### 4.1.1 Variadic Templates by Example

~~~C++
#include <iostream>
void print()
{
  
}

template<typename T,typename... Types>
void print(T firstArg,Types... args)
{
  std::cout<<firstArgs<<'\n';
  print(args...);
}
~~~

这段代码要注意的是这是个递归函数，而上面的那个空的print函数就是递归的终止情况。另外要注意的就是调用的句法。

#### 4.1.2 Overloading Variadic and Nonvariadic Templates

~~~C++
#include <iostream>
template<typename T>
void print(T arg)
{
  std::cout<<arg<<'\n';
}

template<typename T,typename... Types>
void print(T firstArg,Types... args)
{
  print(firstArg)；
  print(args...);
}
~~~

这样也是可以的。

#### 4.1.3 Operator sizeof...

~~~C++
template<typename T,typename... Types>
void print(T firstArg,Types... args)
{
  std::cout<<sizeof...(Types)<<'\n';   //打印还剩余多少个类型
  std::cout<<sizeof...(args)<<'\n';    //打印还剩余多少个参数
}
~~~

然而，我们并不能通过判断剩余参数来决定递归终止：

~~~C++
template<typename T,typename... Types>
void print(T firstArg,Types... args)
{
  std::cout<<firstArg<<'\n';
  if(sizeof...(args)>0){
    print(args...);
  }
}
~~~

在编译期，递归到了sizeof…(args)==0时，并不会终止，而是会继续实例化分支里的print(args…)，因此，如果不存在`print()`的定义，就会编译出错。**需要区分编译时行为和运行时行为。**

### 4.2 Fold Expressions

| Fold Expression | Evaluation|
| :--------------: | :----------------------------------------: |
| (… op pack)     | (((pack1 op pack2) op pack3) … op packN) |
|(pack op ...)|(pack1 op (... (packN-1 op packN)))|
|(init op ... op pack)|(((init op pack1)op pack2) ... op packN)|
|(pack op ... op init)|(pack1 op (... (packN op init)))|

注意句法，包括外围的`()`，里边的`…`，以及前后一致的op(运算符)，pack指不定长的参数列表，init指一个确定的初值。

~~~C++
template<typename T>
class AddSpace
{
  private:
  	T const& ref;
 	public：
    AddSpace(T const& r):ref(r){
    }
  	friend std::ostream& operator<<(std::ostream &os,AddSpace<T> s){
      return os<<s.ref<<' ';
    }
};
template<typename... Args>
void print(Args... args)
{
  (std::cout<<...<<AddSpace(args))<<'\n'; //注意这里AddSpace(args)做了类型推断，还有就是这个表达式在编译时就展开了，所以并不存在给AddSpace传入一串参数的问题，而是每次只传一个
}
~~~

#### 4.3 Application of Variadic templates

~~~C++
namespace std{
  template<typename F,typename... Args> 
  shared_ptr<T> make_shared(Args&&... args);
  
  class thread{
    public:
    	template<typename F,typename... Args>
    	explicit thread(F&& f,Args&&... args);
    ...
  };
  template<typename T,typename Allocator=allocator<T>>
  class vector{
  	public:
  		template<typename... Args> reference emplace_back(Args&&... args);
  };
}
~~~

可以看到，这里的参数都是通过移动语义实现了完美转发的。

注意对于可变长函数参数，如果传值，那么就会进行拷贝和退化，如果传引用，则严格保持原样。这个跟普通参数是一样的。

### 4.3 Variadic Class Templates and Variadic Expressions

#### 4.4.1 Variadic Expressions

~~~C++
template<typename... T>
void printDoubled(T const&... args)
{
  print(args+args...);  //注意，两个args，另外，不要忘记参数列表总是在展开之后才传入调用函数
}
~~~

所以

~~~C++
printDoubled(7.5,std::string("hello"),std::complex<float>(4.2));
~~~

之后，print函数的实际调用是：

~~~C++
print(7.5+7.5,std::string("hello")+std::string("hello"),std::complex<float>(4.2)+std::complex<float>(4.2))
~~~

这是一种新的句法，而不是什么语法糖。

还可以看这么个例子

~~~C++
template<typename ... T>
void addOne(T const&... args)
{
  print(args+1...); //ERROR：1...is a literal with too many decimal points
  print(args+1 ...);//OK
  print((args+1)...);//ok
}
~~~

可以这么看这两个例子的区别，上面是两个向量加法，下面是一个向量加一个常量。

~~~C++

template<typename T1,typename... TN>
constexpr bool isHomogeneous(T1,TN...)
{
    return (std::is_same<T1,TN>::value && ...);
}

int main()
{
     std::cout<<isHomogeneous(43,-1,"hello")<<std::endl;  //0
}

~~~

这里的问题是`(std::is_same<T1,TN>::value && …);`会展开成什么样，到底是`<T1,T2>,<T1,T3>...`呢还是`<T1,T2>,<T2,T3>...`,有这个 疑问在于前面$4.2提到过的`(pack op …)`会展开成`(pack1 op (... (packN-1 op packN)))`，而按照本节说明，会展开成`func(pack1,pack2) op func(pack1,pack3) …`。不过我们可以这样验证这里会展开成什么,增加一个函数

~~~C++
template<typename T1,typename... TN>
constexpr bool isHomogeneous1(T1,TN...)
{
    return (std::is_same<T1,TN>::value || ...);
}

int main()
{
    std::cout<<isHomogeneous(43,-1L,-1L)<<std::endl;  //1
  	std::cout<<isHomogeneous1(43,-1L,-1L)<<std::endl; //2
  	std::cout<<isHomogeneous1(43,-1L,-1)<<std::endl;  //3
}
~~~

显然不论展开成那种情况，语句1都会输出0。

如果展开成`<T1,T2>,<T1,T3>...`,那么语句2会输出0，语句3会输出1；

如果展开成`<T1,T2>,<T2,T3>...`,那么语句2会输出1，语句3会输出0；

而我们运行的结果是`0 0 1`，说明`(std::is_same<T1,TN>::value && …)`展开成了`(std::is_same<T1,T2>&&std::is_same<T1,T3>)`。

#### 4.4.2 Variadic Indices

~~~C++
template<typename C,typename... Idx>
void printElems(C const&coll,Idx... idx)
{
  print(coll[idx]...)
}

std::vector<std::string> coll={"good","times","say","bye"};
printElems(coll,2,0,3);  //=>print(coll[2],coll[0],coll[3]);
~~~

~~~C++
template<std::size_t... Idx,typename C> //非类型模板参数
void printIdx(C const& coll)
{
  print(coll[Idx]...);
}

printIdx<2,0,3>(coll);  //与printElems等价
~~~

#### 4.4.3 Variadic Class Templates

~~~C++
template<std::size_t...>
struct Indices{

};

template<typename T,std::size_t... Idx>
void printByIdx(T t,Indices<Idx...>)
{
  print(std::get<Idx>(t)...);  //get<>()编译时访问指定下标的元素
}

std::array<std::string,5> arr={"hello","my","new","!","world"};
printByIdx(arr,Indices<0,4,3>());

auto t=std::make_tuple(12,"monkeys",2.0);
printByIdx(t,Indices<0,1,2>());
~~~

## Chapter 5 Tricky Basics

### 5.1 Keyword typename

### 5.2 Zero Initialization

~~~C++
template<typename T>
void foo()
{
  T x;  //如果T是内置类型，t的初值未定义，如果是自定义类型，并且有缺省构造函数，其值取决于构造函数
  T y{}; //0,false,nullptr等等，取决于是什么内置类型，对于自定义类型，同样取决于构造函数，并且即便这个构造函数时explicit的也是有效的
  T z=T();  //c++11，对于自定义类型，c++17之前必须保证拷贝构造函数不是explicit的
}
~~~

~~~c++
template<typename T>
class MyClass{
  T x;   // 直接T x{}；也是可以的，这样就不需要在构造函数初始化x了
  public:
  MyClass():x{} {}  //MyClass():x(){} 也是有效的
};
~~~

但是缺省参数是不能用这个句法的

~~~c++
template<typename T>
void foo(T p{}){}  //这是不对的，只能用foo(T p=T{})
~~~

### 5.3 Using this->

对于一个类模板，如果其基类也依赖于模板参数，使用一个名字，比方说x，与this->x是不一样的，即使这个x是继承来的。

~~~C++
template<typename T>
class Base{
  public:
  	void bar();
};
template<typename T>
class Derived:Base<T>{
  public:
  	void foo(){
      bar();  //调用一个外部的bar()或者出错，反正不会是Base<T>::bar()
      				//如果确定要调用Base的bar()，就需要用Base<T>::()
      				//如果子类没有覆盖基类的同名函数，用this->也是可以调用基类的这个函数的
      				//但是如果子类覆盖了基类的函数，用this->调用的就是子类的函数了
      				//另外要注意这只是针对一个模板基类而言，
      				//如果Base不是个模板类，而且子类没有覆盖这个函数，那么就不需要加上this->之类的来修饰了
      				//总之，对一个模板基类，解析名字的时候是不会考虑基类里的名字的，除非指定。
      				//因为模板类是编译时生成的，如之前所述，没有调用的代码是不会生成的。
    }
};
~~~

### 5.4 Templates for Raw arrays and String Literals

前面已经说过，当用引用传递裸数组或者字符串文本时，参数是不会退化的，也就是说假如传入一个字符串“hello”，这个参数就会被认为是const char[6]，而当传值的时候这个参数就会退化成const char*。

也可以提供专门处理裸数组或者字符串文本的模板

~~~C++
template<typename T,int N,int M>
bool less(T (&a)[N],T (&b)[M])
{
  for(int i=0;i<N && i<M;++i){
    if(a[i]<b[i]) return true;
    if(b[i]<a[i]) return false;
  }
  return N<M;
}
~~~

也可以提供只处理一个字符串文本或者其他字符数组的模板：

~~~C++
template<int N,int M>
bool less(char const (&a)[N],char const (&b)[M])
{
  for(int i=0;i<N && i<M;++i){
    if(a[i]<b[i]) return true;
    if(b[i]<a[i]) return false;
  }
  return N<M;
}
~~~

其实到目前为止，我们可以发现，使用模板就可以解决C/C++函数传递数组做参数时不知道数组的大小的问题了，当然，记得用引用。

~~~C++
template<typename T>
class Stack{
  private:
  	std::deque<T> elems;
  public:
  	void push(T const&);
  	void pop();
  	T const& top() const{
      return elems.empty();
    }
  	
  	template<typename T2>
  	Stack& operator=(Stack<T2> const&);
  	template<typename> friend class Stack;//任意的Stack都是这个Stack<T>的友元，当然也是Stack<T2>的友元。注意前面的模板申明是必需的。
};

template<typename T>
	template<typename T2>  //这里的缩进不是必须的,但是也不能合成一句。因为Stack的定义就只有一个T
Stack<T>& Stack<T>::operator=(Stack<T2> const& op2)
{
  elems.clear();
  elems.insert(elems.begin(),
              op2.elems.begin(),
              op2.elems.end());   //注意，容器类在调用push_front()时会做类型检查
  return *this;
}
~~~

于是我们就可以给不同T的Stack赋值啦 。

#### Specialization of Member Function Templates

~~~C++
class BoolString{
  private:
  	std::string value;
  public:
  	BoolString(std::string const&):value(s){}
  	template<typename T=std::string>
  	T get()const{
      return value;
    }
};
template<>
inline bool BoolString::get<bool>()const{
	return value=="true"||value=="1"||value=="on";
}
~~~

注意，这个特化版本的函数不需要也不能声明，只能定义。另外这个函数通常在头文件里边，所以inline是必须的。

### 5.6 Variable Templates

~~~C++
template<typename T= long double>
constexpr T pi{3.1415926535897932385};

std::cout<<pi<double><<'\n';
std::cout<<pi<float><<'\n';   //注意这是两个不同的变量了。
~~~

这种生命不能在函数或者块里边，也就是必须是全局的。

#### Variable Templates for Data Members

~~~C++
namespace{
  template<typename T>
  class numeric_limits{
    public:
    	static constexpr bool is_signed=false;
  };
}

templage<typename T>
constexpr bool isSigned=std::numeric_limits<T>::is_signed;
~~~

这样我们就可以用`isSigned<char>`来代替`std::numeric_limits<char>::is_signed`了。

#### Type Traits Sffix_v

~~~C++
namespace{
  template<typename T>constexpr bool is_const_v=is_const<T>::value;
}
~~~

因此我们就可以使用 `is_const_v<T>`了。

### 5.7 Template Template Parameters

#### Template Template Argument Matching

~~~C++
#include <deque>
#include <cassert>
#include <memory>

template <typename T,
					template<typename Elem,
									typename = std::allocator<Elem>> 
								class Cont = std::deque>  //对于c++17，这里的class可以用typename代替
                  												
class Stack{
  private:
    Cont<T> elems;
  public:
  	void push(T const&);
  	void pop();
  	T const& top() const;
  	bool empty() const{
      return elems.empty();
    }
  
  	template<typename T2,template<typename Elem2,
  																typename = std::allocator<Elem2>
                                 >class Cont2>  //参见上面注释
    Stack& operator=(Stack<T2,Cont2> const&);
  	template<typename,template<typename,typename>class>
  	friend class Stack; //虽然Stack本身不用带上模板参数，但是上面的模板申明还是必须要的,并且别漏了那个class
}; 

template<typename T,template<typename,typename>class Cont>
void Stack<T,cont>::push(T const& elem)
{
  elems.push_back(elem);
}
template<typename T,template<typename,typename>class Cont>
void Stack<T,Cont>::pop()
{
  assert(!elems.empty());
  elems.pop_back();
}
template<typename T,template<typename,typename>class Cont>
T const& Stack<T,Cont>::top() const
{
  assert(!elems.empty());
  return elems.back();
}
template<typename T,template<typename,typename>class Cont>
template<typename T2,template<typename,typename>class Cont2>
Stack<T,Cont>&
Stack<T,Cont>::operator=(Stack<T2,Cont2> const& op2)
{
  elems.clea();
  elems.insert(elems.begin(),
              op2.elems.begin(),
              op2.elems.end());
  return *this;
}
~~~

## Chapter 6 Move Semantics and `enable_if<>`

### 6.1 Perfect Forwarding

所谓移动语义不透传，首先要了解一件事，就是变量必定是个 左值，所以

~~~C++
int &&rr=24;  //ok
int &&rr1=rr; //是不对的。这时候的rr已经是个左值，
~~~

因而，

~~~C++
class X{};
void g(X&){}
void g(X const &)
void g(X&&)
void f(X& val)
{
  g(val);
}
void f(X const& val)
{
  g(val);
}
void f(X&&val)
{
  g(std::move(val))
}
int main()
{
  X v;
  X const c;
  f(v); //f(X&)=>g(X&)
  f(c); //f(X const&)=>g(X const&)
  f(X()); //f(X&&)=>g(X&&)
  f(std::move(v))//f(X&&)=>g(X&&)
}
~~~

作为一个老程序员，反倒更应该注意`f(X())`,因为在引入移动语义之前，这里会创建一个临时变量，然后再复制到给参变量，引入移动语义之后，这里就不再复制一遍了。

使用模板，可以合并成一个 

~~~C++
template<typename T>
void f(T&& val){
  g(std::forward<T>(val)); 
}
~~~

`move()`没有模板参数，并且总是触发移动语义，而`forward`则依赖于传入的模板参数。

- X&& 声明的参数是一个右值引用，只能绑定到一个可移动的对象：

  - prvalue 比如临时对象
  - xrvalue 比如用`std::move()`传送的对象

    不管怎么说，这个参数是可以修改的，从而是可以偷取其值的。

  但是，X const&&这样的声明也是合法的，但是只是用来强制使用`std::move()`来传送一个临时的对象，这种情况下，必然是无法修改这个临时对象的。

- T&& 声明一个转发引用的模板参数，可以绑定到可变的，不可变的，移动的对象上，在函数里边是可以偷取其值的。

### 6.2 Special  Member Function Templates

~~~C++
#include <utility>
#include <string>
#include <iostream>

class Person
{
    private:
        std::string name;
    public:
        template<typename STR>
        explicit Person(STR&& n):name(std::forward<STR>(n)){
            std::cout<<"templ-const for '"<<name<<"'\n";
        }   

        Person(Person const& p):name(p.name){
            std::cout<<"copy-const for '"<<name<<"'\n";
        }
        Person(Person && p):name(std::move(p.name)){
            std::cout<<"move-const for '"<<name<<"'\n";
        }
};
int main()
{
    std::string s="sname";
    Person p1(s);
    Person p2{"tmp"};
   // Person p3{p1};
    Person const p2c{"ctmp"};
    Person p3c(p2c);
    Person p4{std::move(p1)};
    return 0;
}                      
~~~

上面`Person p3(p1)`出错的原因在于，根据重载规则，`Person(STR&& n)`是比`Person(Person const& p)`更合适的匹配，但是在对`name`初始化时，一个Person对象并不能自动转换成string。

### 6.3 Disable Templates with enable_if

~~~C++
template<typename T>
typename std::enable_if<(sizeof(T)>4)>::type  //如果表达式为真，则type为void，否则忽略这个函数
foo()
{
  
}

template<typename T>
typename std::enable_f<(sizeof(T)>4),T>::type  //如果表达式为真，则type为T，否则忽略这个函数
foo()
{
  
}
~~~

或者使用`enable_if_t`

```C++
template<typename T>
std::enable_if_t<(sizeof(T)>4)>  //如果表达式为真，则为void，否则忽略这个函数
foo()
{
  
}

template<typename T>
std::enable_f_t<(sizeof(T)>4),T>  //如果表达式为真，则为T，否则忽略这个函数
foo()
{
  
}
```

这种方式不仅仅用于确定返回值，所以更常见的是，

```C++
template<typename T,
	typename=std::enable_if_t<(sizeof(T)>4)>>  //如果sizeof(T)>4,则展开成typename=void
void foo()
{
  
}
```

也可以使用`using`

~~~C++
template<typename T>
using EnableIfSizeGreater4=std::enable_if_t(sizeof(T)>4)>;
template<typename T,
	typename=EnableIfSizeGreater4<T>>  //如果sizeof(T)>4,则展开成typename=void
void foo()
{
  
}
~~~

### 6.4 Using `enable_if<>`

于是我们可以用`enable_if<>`来解决前面的问题。

```C++
template<typename STR,
					typename=std::enable_if_t<
            					std::is_convertible_v<STR,std::string>>>
explicit Person(STR&& n);
```

或者

```c++
template<typename T>
using EnableIfString=std::enable_if_t<std::is_convertible_v<T,std::string>>;

template<typename STR,typename=EnableIfString<STR>>
explicit Person(STR&& n);
```

#### Disabling Special Member Function

所谓特殊成员函数，指的是缺省构造、缺省析构、缺省复制构造、缺省赋值这四个编译器会自动生成的函数。而成员函数模板并不会被认为是特殊成员函数，所以

```c++
class C{
  public:
  	template<typename T>
  	C (T const&){
      ...
    }
};

C x;
C y{x};  //使用缺省的拷贝构造函数，而不是成员模板函数，因为无从指定或者推断T的类型
```

不过我们可以这样，同时加上一些额外的限制

```c++
template<typename T>
class C
{
	C(C const volatile&)=delete;  //注意句法，只有这样我们才能使用成员模板
  
  template<typename U,typename=std::enable_if_t<!std::is_integral<U>::value>>
  C(C<U> const&);
};
```

## Chapter 7 By Value or by Reference?

### 7.2 Passing by Reference

#### 7.2.1 Passing by Const Reference

```c++
template<typename T>
void printR(T const& arg){}

std::string returnString();
std::string s="hi";
printR(s);  //OK:T是std::string，arg是std::string const&
printR(std::string("hi")); //同上
printR(returnString());//同上
printR(std::move(s)); //同上
printR("hi");  //OK: T是char [3] arg是char const(&)[3]
```

函数签名处的const，表示在函数体内不会修改这个引用参数，所以可以传入临时变量。

#### 7.2.2 Passing by Nonconstant Referenc

```c++
template <typename T>
void outR(T& arg){}

std::string returnString();
std::string s="hi";

outR(s);   //OK: T是std::string，arg是std::string&
outR(std::string("hi")); //ERROR:临时变量(prvalue)
outR(returnString()); //ERROR:临时变量(prvalue)
outR(std::move(s));  //ERROR:xvalue

std::string const c="hi";
outR(c);   //OK:T是std::string const
outR(returnConstString()); //OK:returnConstString()返回一个const string
outR(std::move(c));//OK:T是std::string const，move()把c转换成std::string const&&，
										//于是T被推断为std::string const&
outR("hi");//OK: T是char const[3]
```

函数签名处的参数没有const修饰，表明不保证在函数体内不做修改，所以不能传入临时变量，因为临时变量调用完毕就没有了，函数体内无从修改。

但是如果传入一个const的临时变量或者右值，于是T本身就被推断成const的，表明不会在函数体内做修改，这样也是可以的。

总结起来就是，要区分传送的对象本身以及所声明的参数

- 如果函数参数声明为const的，那么传入的对象本身是没有什么限制的，可以是const的，也可以是非const的，甚至可以是临时的，传入的对象必然会被一个const的引用所指向，这时候T总是会被推断为非const的。

- 如果函数参数声明没有const，那么传入的对象就必须是个左值，这时候的T就被推断为arg的类型，或者，也可以是const的临时变量和const的右值，这时候T本身会被推断为const的，从而要确保函数体内不会对参数做任何修改。

如果不希望传送一个const的对象到一个非const的引用，毕竟声明非const的引用本身就是为了修改参数，可以这样

```c++
template<typename T>
void outR(T & arg)
{
  static_assert(!std::is_const<T>::value,"out parameter of out<T>(T&) is const");
}
```

或者

```c++
template<typename T,typename= std::enable_if_t<!std::is_const<T>::value>
void outR(T &arg)
{
  
}
```

或者

```c++
template<typename T>
requires !std::is_const_v<T>
void outR(T &arg)
{
  
}
```

#### 7.2.3 Passing by Forwarding Reference

```c++
template<typename T>
void passR(T&& arg){  //arg声明为转发引用
  
}

std::string s="hi";
std::string const c="hi";
int arr[4];
passR(s);  //T和arg都是std::string&
passR(std::string("hi")); //arg是个prvalue。推断结果T是std::string ,arg是std::string&&
passR(returnString());  //同上
passR(std::move(s));    //arg是个xrvalue,推断结果同上

passR(c);  //T和arg都是std::string const&
passR("hi");  //T和arg都是char const (&)[3]
passR(arr);  //T和arg都是int (&)[4]
```

也就是说，对于`void passR(T&& arg)`，arg只有在prvalue或者xrvalue的情况下才会被推断为一个右值引用，这个时候T会被推断为非引用类型。其他情况T和arg类型一致，且都是引用类型。

因此，使用转发引用可以对应右值、const和非const这三种情况。

然而，这是模板参数T会被推断为引用的唯一场合，所以，如果

```c++
template<typename T>
void passR(T &&arg){
  T x;
};

int i;
passR(i); //T会被推断为int&，于是int &x；这样的声明就会出错。
```

## Chapter 8 Compile-Time Programming

#### 8.1 Template Metaprogramming

```c++
#include <iostream>

template <unsigned p,unsigned d>
struct DoIsPrime{
    static constexpr bool value=((p%d)!=0)&&DoIsPrime<p,d-1>::value;
};

template <unsigned p>
struct DoIsPrime<p,2>{
    static constexpr bool value=(p%2)!=0;
};

template<unsigned p>
struct IsPrime{
    static constexpr bool value=DoIsPrime<p,p/2>::value;
};

template<> struct IsPrime<0>{static constexpr bool value=false;};
template<> struct IsPrime<1>{static constexpr bool value=false;};
template<> struct IsPrime<2>{static constexpr bool value=false;};
template<> struct IsPrime<3>{static constexpr bool value=false;};

int main()
{
    std::cout<<IsPrime<3>::value<<std::endl;
}
~                            
```

需要注意的是:

- 句法，模板标识、类名、类成员变量定义及使用，`static constexpr`倒在其次了
- 模板是编译期展开的，所以不可能像这样使用命令行参数：`std::cout<<IsPrime<atoi<argv[1]>>::value<<std::endl;`

- 最后，`template<> struct IsPrime<3>{static constexpr bool value=false;};`是必须的。

### 8.2 Computing with `constexpr`



```c++
constexpr bool
doIsPrime(unsigned p,unsigned d)
{
  return d!2?=(p%d!=0)&&doIsPrime(p,d-1): (p%2!=0);
}
constexpr bool isPrime(unsigned p)
{
  return p<4?!(p<2) : doIsPrime(p,p/2);
}
```

或者

```c++
constexpr bool isPrime(unsigned int p)
{
  for(unsigned int d=2;d<=p/2;++d){
    if(p%2 ==0) return false;
  }
  return p>1;
}
```

对于constexpr，是否在编译期计算由编译器决定

```c++
constexpr bool b1=isPrime(9);  //编译时计算，因为这个值必须在编译时得到
const bool b2=isPrime(9);   //如果b2是个全局的或者是在名字空间定义的，就会在编译时计算
bool fiftyeSevenIsPrime(){
  return isPrime(57);   //编译器决定在什么时候计算
}

int x
...
std::cout<<isPrime(x);  //必然在运行时计算
```



### 8.3 Execution Path Selection with Partial Specialization

~~~c++
template<int SZ,bool=isPrime(SZ)>
struct Helper;
template<int SZ>
struct Helper<SZ,false>
{};
template<int SZ>
struct Helper<SZ,true>
{};

template<typename T,std::size_t SZ>
long foo(std::array<T,SZ> const&coll)
{
 	Helper<SZ> h; 
 ....
}
~~~

或者

```c++
template<int SZ,bool=isPrime>
struct Helper
{};
template<int SZ>
struct Helper<SZ,true>
{};
```

### 8.4 SFINAE(Substitution Failure Is Not An Error)

```c++
template<typename T,unsigned N>  //1
std::size_t len(T(&)[N])
{
  return N;
}

template<typename T>  //2
typename T::size_type len(T const& t)
{
  return t.size();
}
```
```c++
int a[10];
std::cout<<len(a);  //OK,第一个模板匹配
std::cout<<len("tmp"); //OK,第一个模板匹配
```

实际上，如果只看函数签名，第二个模板也是匹配的，但是返回值不合适，所以这个替换被忽略了，并不会引发错误。

```c
int *p;
std::cout<<len(p);  //ERROR:no matching len() function found

std::allocator<int> x;
std::cout<<len(x);  //ERROR:len() function found,but can't size()
```

这两个是不同的错误，都会引发编译错误。

#### 8.4.1 Expression SFINAE with `decltype`

```c++
template<typename T>
auto len(T const& t) -> decltype((void)(t.size()),T::size_type())
{
  
}
```

`decltype()`会评估其参数列表中用逗号运算符分隔的每一个表达式的值，如果在最后一个参数，这里是`T::size_type()`，之前无效或者为假或者非法，则这个函数模板会被SFINAE，也就是忽略掉，如果都是合法的，那么最后一个参数就是这个函数的返回值。

### 8.5 Compile-Time `if`

部分特化、SFINAE以及`std::enable_if`是针对整个模板的。使用`if constexpr(…)`则可以在编译时针对语句。

```c++
template<typename T,typename... Types>
void print(T const& firstArg,Types const&... args)
{
  std::cout<<firstArg<<'\n';
  if constexpr(sizeof...(args)>0){
    print(args...);
  }
}
```

注意`if constexpr(…)`不仅仅可以用在模板中，普通函数中也是可以用的，只要在编译时就可以生成一个布尔值。

```c++
int main()
{
  if constexpr(std::numeric_limits<char>::is_signed){
    foo(42);
  }else{
    undeclared(42);
    static_assert(false,"unsigned");
    static_assert(!std::numeric_limits<char>::is_signed,"char is unsigned");
  }
}
```

## Chapter 9 Using Templates in Practice

## Chapter 10 Basic Template Terminology

### 10.1 “Class Template” or “Template Class”

- *class type* 包括*`union`*，*class*则只是指*`class`*和*`struct`* 

### 10.3 Declarations versus Definitions

#### 10.3.1 Complete versus Incomplete Types

- void是*incomplete*的类型

- 元素是*imcomplete*类型的数组

- 未指定边界的数组

- 一个枚举，如果其底层类型或者枚举值没有定义，也是

- 类声明

- 以上所有加了const或者volatile修饰的仍然是

```c++
class C;  //incomplete
C const* cp; //cp是一个指向incomplete类型的指针
extern C elems[10];  //elems是incomplete的
extern int arr[];  //incomplete

class C{};  //C现在是complete的了，所以cp和elems也是了
int arr[10];  //complete
```

 ## Chapter 11 Generic Libraries

### 11.1 Callables

函数对象：

- 函数指针
- 重载`operator()`的类类型，以及lambda表达式
- 可以通过转换函数生成函数指针或者函数引用的类类型

#### 11.1.1 Supporting Function Objects

```c++
template<typename Iter,typename Callable>
void foreach(Iter current,Iter end,Callable op)
{
  while(current !=end)
  {
    op(*current);
    ++current;
  }
}

void func(int i)
{
  ...
}
class FuncObj
{
	public:
  	void operator()(int i) const{  //通常一个operator()需要定义成const的，否则在某些场合会出错
      ...
    }
};

std::vector<int> primes={2,3,5,7,11,13,17,19};
foreach(primes.begin(),primes.end(),func);  //退化成指针
foreach(primes.begin(),primes.end(),&func);  //指针
foreach(primes.begin(),primes.end(),FuncObj()); //函数对象
foreach(primes.begin(),primes.end(),[](int i){...});
```

#### 11.1.2 Dealing with Member Function and Additional Arguments

在C++17，还可以这样

```c++
template<typename Iter,typename Callable,typename... args>
void foreach(Iter current,Iter end,Callable op,Args const&... args)
{
  while(current!=end){
    std::invoke(op,args...,*current);
    ++current;
  }
}
class MyClass
{
  public:
  	void memfunc(int i) const{...}
};
foreach(primes.begin(),primes.end(),[](std::string const& prefix,int i){//两个参数了
  			std::cout<<prefix<<i<<'\n';}
       	"- value: ");
MyClass obj;
foreach(primes.begin(),primes.end(),&MyClass::memfunc,obj);//注意这里成员函数作为op参数的句法
```

就`std::invoke()`而言

- 如果Callable是个指向成员的指针，那么附加参数列表中的的第一个参数被认为是`this`对象，也就是指向一个类实例的指针，其余参数则被当做这个callable对象的参数
- 否则，所有参数都当做callable对象的参数

注意这里对callable和args都不能使用完美转发，因为第一次调用的时候其值已经被转移走了，接下来的调用就会没有目标了。

#### 11.1.3 Wrapping Function Calls

`std::invoke()`通常用来封装一个函数，比如记录调用情况，测试运行时间，以及准备一些上下文比方说启动一个新线程等等。当然我们是可以使用完美转发的 。

```c++
#include <utility> //std::invoke()
#include <functional> //std::forward()

template<typename Callable,typename... Args>
decltype(auto) call(Callable &&op,Args&&... args)
{
  return std::invoke(std::forward<Callable>(op),std::forward<Args>(args)...);
}
```

这里有趣的事情是怎么样把一个被调函数的返回值完美转发给调用者。为了能够返回引用，这里用了`decltype(auto)`，而不是仅仅使用`auto`。

### 11.4 References as Template Parameters

~~~C++
template<typename T>
void tmplParameterIsReference(T)  //因为实际上没有用到参数本身，所以就省略参数名了
{
  std::cout<<"T is reference:"<<std::is_reference_v<T><<'\n';
}

int main()
{
  std::cout<<std::boolalpha;  //注意这里的输出格式控制
  int i;
  int& r=i;
  tmplParameterIsReference(i);  //false
  tmplParameterIsReference(r);  //false
  tmplParameterIsReference<int&>(i);  //true
  tmplParameterIsReference<int&>(r);  //true
}
~~~

- 对于一个引用变量，`r`，尽管***表达式`r`***具有引用类型，然而一个***表达式***的类型并不会是个引用

- 然而可以强制指定一个类型

### 11.5 Defer Evaluations
```c++
template<typename T>
class Cont
{
  private:
  	T* elems;  //不完全类型
  public:
  	typename std::conditional<std::is_move_constructible<T>::value,T&&,T&>::type foo();
};

struct Node
{
  std::string value;
  Cont<Node> next; //这里Node是个不完全类型，而std::is_move_constructible<T>要求一个完全类型
};
```

可以这样来解决：

```c++
template<typename T>
class Cont
{
  private:
  	T* elems;  //不完全类型
  public:
  	template<typename D=T>
  	typename std::conditional<std::is_move_constructible<D>::value,T&&,T&>::type foo();
};
```

这样当一个具体的类型，比方说Node，的变量初始化的时候编译器才会去计算`std::is_move_constructible<D>`，这个时候的Node已经是完全的了，它只是在定义的时候是不完全的。

### 11.6 Things to Consider When Writing Generic Libraries

- 在模板中使用转发引用(`&&`)来转发值

```c++
template<typename T>
void f(T&& val){
  g(std::forward<T>(val));
}
```

- 如果这个值不依赖于模板参数，则使用`auto&&`

```c++
template<typename T>
void foo(T x)
{
  auto && val=get(x);
  ...
  set(std::forward<decltype(val)>(val)); 
}
```

- 当一个参数被声明为转发引用(&&)时，一个左值参数会变成一个引用类型(&)

```c++
templage<typename T> void f(T&& p);
int i;
int const j=0;
f(i);  //参数是左值，T为int&，p为int&
f(j);  //参数是左值，T为int const&,p为int const&
f(2);  //参数是右值，T为int,p为int&&
```

- 使用`std::addressof()`来获取一个依赖于模板参数的对象的地址，如果这个对象类型重载了`operator&`
- 对于成员函数模板，确保他们不会比缺省的拷贝/移动构造函数和赋值运算符更匹配
- 当模板参数有可能是字符串文本并且不是传值时，可以考虑使用`std::decay`
- 当输出或者输入输出参数(引用或者指针)依赖于模板参数时，必须考虑模板参数是const的情况
- 准备好处理模板参数是引用时引发的问题，特别的，可能需要确保返回值不是个引用
- 声明递归的数据结构时，要注意不完整类型的支持
- 重载所有的数组类型，`T []`，`T (&)[]`，`T (&)[SZ]`，`T *`而不仅仅是`T [SZ]`

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
  * 名字空间范围里static限定的函数模板
  * 直接或者间接属于匿名空间的模板，这时候具有内部链接属性
  * 匿名类的成员模板，这时候没有连接属性

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

#### Primary Template

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

