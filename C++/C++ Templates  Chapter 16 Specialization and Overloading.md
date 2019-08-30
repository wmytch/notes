# C++ Templates : Chapter 16 Specialization and Overloading

[TOC]



## 16.1 When “Generic Code” Doesn’t Quite Cut It

```C++
template<typename T>
class Array
{
  private:
  	T* data;
  public:
  	Array(Array<T> const&);
  	Array<T>& operator=(Array<T> const&);
  
  	void exchangeWith(Array<T>* b)
    {
      T *tmp=data;
      data=b->data;
      b->data=tmp;
    }
  
  	T& operator[](std::size_t k)
    {
      return data[k];
    }
};
template<typename T>
inline void exchange(T *a,T *b)
{
  T tmp(*a);
  *a=*b;
  *b=tmp;
}
```

对于简单类型，exchange运行得很好，但是对于复杂类型，就可能会有效率上的问题，主要是复制产生的一系列内存操作，当然，说到底效率问题最终大都会归结到内存访问上。

### 16.1.1 Transparent Customization

上面例子中的exchangeWith提供了一种高效的方式。此外，我们还可以通过重载函数模板来解决

```c++
template<typename T>
void quickExchangge(T *a,T *b)  //#1
{
  T tmp(*a);
  *a=*b;
  *b=tmp;
}

template<typename T>
void quickExchange(Array<T>* a,Array<T>* b)  //#2
{
  a->exangeWith(b);
}

void demo(Array<int>* p1,Array<int>* p2)
{
  int x=42,y=-7;
  quickExchange(&x,&y);  //使用#1
  quickExchange(p1,p2);  //使用#2
}
```

实际上，`quickExchange(p1,p2)`可以匹配两个模板，但是编译器会认为#2是更好的匹配，这也是我们想要的。

### 16.1.2 Semantic Transparency

```c++
struct S{
  int x;
} s1,s2;

void distinguish(Array<int> a1,Array<int> a2)
{
  int* p=&a1[0];
  int* q=&s1.x;
  a1[0]=s1.x=1;
  a2[0]=s2.x=2;
  quickExchange(&a1,&a2);  //之后*p仍然是1
  quickExchange(&s1,&s2);  //之后*q变成2
}
```

这是很显然的事情，第一个交换的是`Array<int>`内部的指针，但是并没有对其所指向的地址的内容做任何改变，而第二个交换则是改变了struct的成员变量的值。或者说p，q的值没有发生变化，它们所指向的仍然是交换前的地址，只不过p所指向的地址本来属于a1，交换后属于a2，而q所指向的地址的内容发生了变化。

实际上，对于最初的exchange函数我们可以添加一个重载函数

```c++
template<typename T>
void exchange(Array<T>* a,Array<T>* b)
{
  T *p=&(*a)[0];  //注意是取首元素的地址
  T *q=&(*b)[0];
  
  for(std::size_t k=a->size();k--!=0;)
  {
    exchange(p++,q++);
  }
}
```

这个函数会递归调用，所以像`Array<Array<char>>`这样的参数类型也是可以正确执行的。另外注意这里没有用inline，因为这里已经假定有许多操作。还有，p、q都取的是首地址，这样执行++操作时才是正确的。

## 16.2 Overloading Function Templates

```c++
template<typename T>
int f(T)
{
  return 1;
}
template<typename T>
int f(T*)
{
  return 2;
}
int main()
{
  std::cout<<f<int*>((int*)nullptr); //f<T>(T);
  std::cout<<f<int>((int*)nullptr); //f<T>(T*);
}
```

- 对于`f<int*>`，两个函数模板分别生成`f<int *>(int*)`和`f<int *>(int **)`这两个函数，而从`(int *)nullptr`我们得知参数只能是`(int *)`，所以选择了`f<T>(T)`。
- 对于`f<int>`，两个函数模板分别生成`f<int>(int)`和`f<int>(int *)`，而从`(int *)nullptr`我们可知参数必须是`(int *)`，所以选择了`f<int>(int *)`。

### 16.2.1 Signagures

函数签名包括下面一些内容

- 函数的名字，或者函数模板的名字，以及与这个名字相关的以下内容
- 所在的类或者名字空间，如果这个名字是内部链接，则包括这个名字声明所在的翻译单元
- const/volatile，如果有的话
- &或者&&，如果有的话
- 函数参数
- 返回类型
- 模板参数

只要签名不一样，那么就可以共存，但是也有可能造成重载的不明确错误

### 16.2.2 Partial Ordering of Overloaded Function Templates

```c++
template<typename T>
int f(T)
{
  return 1;
}
template<typename T>
int f(T*)
{
  return 2;
}
int main()
{
  f(0);  //f<T>(T)
  f(nullptr); //f<T>(T)
  f((int*)nullptr); //f<T>(T*)
}
```

对于`f((int*)nullptr);`，实际上两个模板都是匹配的：`f<int>(int *)`和`f<int*>(int *)`，之所以不会产生不明确错误，在于有一条规则就是选择那个更加特别的模板，于是选择了`f<int>(int *)`

### 16.2.3 Formal Ordering Rules

对于重载函数，按照以下顺序进行解析

- 忽略使用了缺省值的参数，也就是在调用函数时未提供对应的参数而是在函数中使用了该参数的缺省值。
- 忽略在函数中未使用的可变参数，也就是在调用函数时未提供可变参数。
- 接下来开始用生造类型替换模板参数形成函数调用参数列表并进行比较，对于类型转换函数模板，还包括其返回类型。**所谓生造类型，指的是实际上并不存在，只是为了替换而创造出来的一个用于占位的类型符号**。
- 如果由一个函数模板替换生成的调用参数列表可以精确匹配另一个函数模板，而反过来不行，则这一个函数模板被认为是比另一个函数模板更加特殊的。
- 如果一个函数并不比另一个函数特殊，反过来也是如此，那么就称之为不明确的

以前面的例子为例，首先替换生成两个参数列表：`(A1)`和`(A2*)`，显然，用`A2*`替换第一个模板的T是可行的，然而，用A1替换第二个模板的`T*`是不可行的，于是第二个模板被认为是更特殊的。

```c++
template<typename T>
void t(T*,T const* = nullptr,...);
tempalte<typename T>
void t(T const*,T*,T* = nullptr);
void example(int *p)
{
  t(p,p);
}
```

- 对于第一个模板，没有使用可变参数，对于第二个模板，使用了缺省参数，于是都被忽略了。
- 替换生成的参数列表分别是`(A1*,A1 const*)`和`(A2 const*,A2*)`
- 用`(A1*,A1 const*)`去替换第二个模板的参数，即`(T const*,T*,T* = nullptr);`，显然，`A1 const`替换`T`是可行的，然而这个结果并不是精确匹配的，因为此时`t<A1 const>(A1 const*,A1 const*,A1 const*=0)`是需要对参数`(A1 *,A1 const*)`做调整的，也就是说对第二个模板的第一个参数`T const*`替换之后变成`A1 const const *`，调整之后也就是去掉一个`const`才是个完全的匹配。
- 用`(A2 const*,A2 *)`去替换第一个模板的参数，即`(T*,T const*)`，显然，用`A2 const`替换T也是可行的，但这同样也是不精确的匹配，因为这时候替换后会变成`t<A2 const>(A2 const*,A2 const const*)`，其第二个参数同样也要调整，因而也是不精确的。
- 也就是说这两个模板不存在偏序，所以，这里对t的调用就会引发不明确错误。

### 16.2.4 Templates and Nontemplates

在解析重载时，同等匹配的情况下，优先选择非模板函数。

在使用const和引用的时候，情况可能会有变化

```c++
template<typename T>
std::string f(T&)
{
  return "Template";
}

std::string f(int const&)
{
  return "Nontemplate";
}

int main()
{
  int x=7;
  std::cout<<f(x)<<std::endl;  //Template
  int const c=7;
  std::cout<<f(x)<<std::endl; //Nontemplate
}
```

因为对于一个int，`f<>(int&)`是比`f(int const&)`更好的匹配。只有在用int const调用时，才会产生同样的签名int const&，这个时候才会优先选择非模板函数。

```c++
class C{
  public:
  	C()=default;
  	C(C const&)
    {
      std::cout<<"copy constructor\n";
    }
  	C(C&&)
    {
      std::cout<<"Move constructor\n";
    }
  	template<typename T>
  	C(T&&)
  	{
      std::cout<<"template constructor\n";
    }
};

int main()
{
  C x;
  C x2{x};  //template constructor
  C x3{std::move(x)};  //move construct;
  C const c;
  C x4{c};  //copy constructor
  C x5(std::move(c)); //template constructor
}
```

可见，成员函数模板比复制构造函数和移动构造函数有更高的优先级，所以，有时候需要屏蔽掉成员函数模板。如6.4解释的那样。

### 16.2.5 Variadic Function Templates

```c++
template<typename T>
int f(T*);
template<typename... Ts>
int f(Ts...);
template<typename... Ts>
int f(Ts*...)
int main()
{
  f(0,0.0);  //f<>(Ts...)
  f((int*)nullptr,(double*)nullptr); //f<>(Ts*...)
  f((int*)nullptr);  //f<>(T*)
}
```

- 对于`f(0,0.0)`
  - 对于`f(T*)`，无法推断T，并且参数的个数不符合
  - 对于`f(Ts…)`，可以推断出Ts是(int,double)
  - 对于`f(Ts*…)`，无法推断Ts
- 对于`f((int*)nullptr,(double*)nullptr)`
  - 对于`f(T*)`，同样的参数个数不符合
  - 对于`f(Ts…)`，可以推断出`f<int*,double*>((int*)nullptr,(double*)nullptr)`
  - 对于`f(Ts*…)`，可以推断出`f<int,double>((int*)nullptr,(double*)nullptr)`
  - 因此，需要考虑上面两种推断的偏序
  - 按照前面所述，生造出两个类型A1，A2\*
  - 显然，用A2\*替换第二个模板的模板参数是可行的
  - 然而，用A1替换第三个模板的参数时，是无法匹配的，因为函数调用函数都是指针
  - 因此，第三个模板被认为是更加特殊的
- 对于`f((int*)nullptr)`
  - 所有的三个模板都是匹配的
  - 然而，非可变参数模板被认为比可变参数模板更加特殊

基于同样的理由，下面的结果也是类似的

```C++
template<typename... Ts> class Tuple{};
template<typename T>
int f(Tuple<T*>);
template<typename... Ts>
int f(Tuple<Ts...>);
template<typename... Ts>
int f(Tuple<Ts*...>);

int main()
{
  f(Tuple<int,double>());  //f<>(Tuple<Ts...>)
  f(Tuple<int*,double*>()); //f<>(Tuple<Ts*...>)
  f(Tuple<int*>());  //f<>(Tuple<T*>)
}
```

## 16.3 Explicit Specialization

别名模板是不能特化的，不论是完全特化还是部分特化。如15.11所述。

不论是完全特化还是部分特化都没有定义一个新的模板或者模板实例，而只是针对未特化模板或者说泛型模板提供了可选的方案，这与重载模板概念上是不一样的。

### 16.3.1 Full Class Template Specialization

```c++
template<typename T>
class S
{
  public:
  	void info()
    {
      //generic (S<T>::info())
    }
};

template<>
class S<void>
{
	public:
		void msg()
		{
			//fully specialized (S<void>::msg())
		}
};
```

这里完全特化的类`S<void>`与泛型类模板`S<T>`的联系仅仅只是这个类名S，而其实现与泛型类可以毫无关系。

模板实参列表必须与模板形参列表对应，比方说不能用非类型的值来替换一个类型参数，不过对应有缺省值的模板参数，可以不用指定对应的实际参数。

```c++
template<typename T>
class Types
{
  public:
  	using I=int;
};

template<typename T,typename U=typename Types<T>::I>
class S;  //#1

template<>
class S<void>
{
	public:
		void f();  //#2
};

template<> class S<char,char>;  //#3

template<> class S<char,0>;  //Error,0不能替换U

int main()
{
  S<int>* pi;  //OK,#1,不需要定义S,因为只是个指针
  S<int> e1;   //Error,#1,但是缺少S的定义
  S<void>* pv; //OK,#2,
  S<void,int> sv; //ok,#2
  S<void,char> e2; //error,#1,但是缺少定义
  S<char,char> e3; //error,#3,但是缺少定义
}
template<>
class S<char,char>  //#3的定义
{};
```

这里需要注意的是

1. \#2等同于S<void,int>，所以sv选择了#2，这时候是不会再去考虑泛型版本的
2. e2并不会选择#2，因为#2的缺省参数是int
3. 如果声明了一个完全特化模板，则对于对应的参数列表，不会再去考虑泛型版本，如e3，但是在声明e3的地方，#3的定义并不可见

一个完全特化的类的成员函数在类外定义时，不需要也不能加上template<>的前缀

```c++
template<typename T>
class S;

template<> class S<char**>
{
	public:
		void print() const;
};

void S<char**>::print() const  //注意，没有模板前缀template<>
{
  
}
```

```c++
template<typename T>
class Outside
{
  public:
  	template<typename U>
  	class Inside
    {
      
    };
};

template<>
class Outside<void>
{
	template<typename U>  //注意，如前所述，这个Inside跟上面的Inside实际上没有关系
	class Inside
	{
		private:
			static int count;
	};
};

//这里不能也不需要加上template<>的前缀，因为Outside<void>已经是个完全特化的类
template<typename U>
int Outside<void>::Inside<U>::count=1;  
```

完全特化与泛型模板的实例不能混用，如果是在一个文件里，这种错误会被编译器捕捉到

```C++
template<typename T>
class Invalid{};
Invalid<double> x1;  //实例化一个Invalid<double>
template<>
class Invalid<double>; //Error,Invalid已经实例化了
```

实际上，上面的错误可以类比于重复定义。

然而，如果完全特化与泛型模板的实例分布在不同的文件中，编译器是捕获不到这个到错误的，甚至链接也不会出问题，但是运行的时候就有可能出现不可预知的错误

```c++
//file1
template<typename T>
class Danger
{
    public:
        enum { max=10 };
};

char buffer[Danger<void>::max];

extern void clear(char*);

int main()
{
    clear(buffer);
}


//file2
#include <iostream>
template<typename T>
class Danger;

template<>
class Danger<void>
{
    public:
        enum {max=100};
};
void clear(char * buf)
{
    std::cout<<Danger<void>::max<<std::endl;
}
```

上面程序编译运行输出100，因为模板替换是在编译期决定的，所以链接的时候两个max实际上已经各自确定了。

### 16.3.2 Full Function Template Specialization

对于函数模板特化而言，与类模板相比，还要考虑函数重载和参数推断。

说到参数推断，对于函数模板和类模板各自是有些特点的，如果不说差异的话。函数模板参数推断发生在函数调用的时候，不提供模板参数而是通过函数调用参数对模板参数进行推断。而类模板的参数推断则是发生在声明一个模板实例的时候，不指定模板参数而是通过构造函数的调用参数对模板参数进行推断。实际上，从这里看到，类模板的模板参数推断其实也是通过对函数参数进行推断而来的，只不过看起来似乎时机不一样，但本质上是一样的。

一个函数模板的完全特化不能包含缺省参数的值，然而，在主模板中指定的缺省参数在显式特化中仍然有效。

```c++
template<typename T>
int f(T,T x=42)
{
  
}

template<> 
int f(int, int=35) //error
{

}
```

因为，一个完全特化提供了一个可选的定义，而不是一个可选的声明。在函数模板被调用的地方，总是以主模板为基础进行解析的。换句话说就是，一个完全特化的函数名字以及参数列表都是以主模板的函数名字和参数列表为基准的。

一般而言，我们可以把主模板、部分特化模板的定义以及完全特化的声明放在一个文件里，而把完全特化的实现放在另外一个文件里，这样可以避免重复定义函数的问题，当然，完全特化也可以作为inline，从而可以放在头文件里边。

### 16.3.3 Full Variable Template Specialization

```C++
template<typename T> constexpr std::size_t SZ=sizeof(T);
template<> constexpr std::size_t SZ<void>=0;
```

显然，一个特化的初始化不需要与主模板相同。

实际上，一个变量模板特化生成的类型也不需要与主模板指定的类型匹配

```c++
template<typename T> typename T::Iterator null_iterator;
template<> BitIterator null_iterator<std::bitset<100>>;
```

注意，BitIterator与T::Iterator是不匹配的，也就是说`null_iterator<std::bitset<100>>`的类型与主模板的`null_iterator<T>`必然是不一样的，或者说并没有一个T存在一个Iterator会是个BitIterator，但是没有关系，我们可以通过完全特化出来这么一个类型。

### 16.3.4 Full Member Specialization

```c++
template<typename T>
class Outer{      //#1
  public:
  	template<typename U>
  	class Inner   //#2
    {
      private:
      	static int count;  //#3
    };
  	static int code;     //#4
  	void print() const   //#5
    {
      std::cout<<"generci";
    }
};

template<typename T>
int Outer<T>::code=6;  //#6

template<typename T>
template<typename U>
int Outer<T>::Inner<U>::count=7;  //#7

template<>
class Outer<bool>  //#8
{
	public:
		template<typename U>
		class Inner   //#9
		{
			private:
				static int count;  //#10
		};
		void print() const   //#11
		{
		
		}
};

template<>
int Outer<void>::code=12;  //#12

template<>
void Outer<void>::print() const  //#13
{
  std::cout<<"Outer<void>"
}
```

对于类Outer<void>而言，#12和#13分别是\#4和\#5的完全特化，而该类其他的成员则仍然依据主模板替换参数而来。但是要注意的是，有了这样完全特化的成员之后，就不能再提供一个Outer<void>的显式特化了。或者说，我们可以给类Outer<void>的每一个成员提供一个完全特化的定义，但是不能再提供一个`templaete<> class Outer<void>{...};`的定义。

对于成员模板Outer<T>::Inner也是可以特化的，但这个特化是针对每一个T而言的

```c++
template<>
template<typename X>
class Outer<wchar_t>::Inner  //部分特化
{
	public:
		static long count;  //注意类型改了
};
template<>
template<typename X>
long Outer<wchar_t>::Inner<X>::count;

template<>
template<>
class Outer<char>::Inner<wchar_t>  //完全特化
{
	public:
		enum {count=1};
};

template<typename X>
template<>
class Outer<X>::Inner<void>;  //error.
```

上面最后的错误在于template<>不能跟在别的参数列表后面，但是跟在别的template<>后面是可以的。

另外，对于前面的#8，也就是类Outer<bool>，因为这是个完全特化的类实例，不能再加template<>修饰，所以特化其成员类模板Inner时就只需要一个template<>，而这一个template<>是针对Inner的。

```c++
template<>
class Outer<bool>::Inner<wchar_t>
{
	public:
		enum {count=2};
};
```

最后，尽管对于普通类，其成员函数和静态数据成员的类外非定义声明是不允许的，然而，对于类模板的成员进行特化时是可以有非定义的声明的。

```c++
template<>
int Outer<void>::code;
template<>
void Outer<void>::print() count;
```

如前面所述，之所以需要这种声明存在，是为了避免重复定义。

## 16.4 Partial Class Template Specialization

```c++
template<typename T>
class List   //#1
{
  public:
  	void append(T const&);
  	inline std::size_t length() const;
};
```

又要append不大可能会变成inline，所以会造成所谓代码膨胀的问题，然而，我们也知道，对于指针类型的List，我们可以提供一个统一的append接口。所以

```c++
template<typename T>
class List<T*>  //#2
{
  private:
  	List<void*> impl;
  public:
  	inline void append(T *p)
    {
      impl.append(p);
    }
  	inline std::size_t length() const
    {
      return impl.length();
    }
};

template<>
class List<void*>  //#3
{
	void append(void *p);
	inline std::size_t length() const;
}
```

\#2就是部分特化模板，而\#3的意义则是为了避免对List<void*>的递归调用。

部分特化模板有一些限制：

1. 部分特化模板的实参列表必须与主模板的参数种类一致，如类型、非类型、模板参数，也就是说部分特化的参数不能另搞一套，而只是对主模板的某些参数给出了具体的类型。
2. 部分特化的实参列表不能有自己的缺省参数，而是使用主模板的缺省参数，这点与完全特化是一致的。
3. 部分特化的非类型实参应该是一个独立的值或者一个简单的非类型参数，不能是像2*N这样的表达式，其中N是一个模板参数。也就是说，编译器不会对非类型实参做计算，只是简单地带入。
4. 部分特化的实参列表不能与主模板的一样，仅仅改名也是不行的。与1的区别可以看下面的例子。
5. 如果有一个模板参数是参数集展开，必须放在参数列表的最后。

```c++
template<typename T,int I=3>
class S;   //主模板

template<typename T>
class S<int,T>;  //违反第一条，同时注意这里模板参数列表不包含缺省参数

template<typename T=int>
class S<T,10>;  //违反第二条

template<int I>
class S<int,I*2>;  //违反第三条

template<typename U,int K>
class S<U,K>;  //违反第四条

template<typename... Ts>
class Tuple;

template<typename Tail,typename... Ts>
class Tuple<Ts...,Tail>;  //违反第五条

template<typename Tail,typename... Ts>
class Tuple<Tuple<Ts...>,Tail>;  //OK,嵌套没有违反限制
```

最后，必须指出，部分特化类模板的参数个数不需要必须与主模板的参数个数相同。还是以前面的List为例

```c++
template<typename C>
class List<void* C::*>  //#4
{
  public:
  	using ElementType = void* C::*;
  	void append(ElementType pm);
  	inline std::size_t length() const;
}

template<typename T,typename C>
class List<T* C::*>  //#5
{
  private:
  	List<void* c::*> impl;
  public:
  	using ElementType T* C::*;
  	inline void append(ElementType pm)
    {
      impl.append(static_cast<void* C::*>(pm));
    }
  	inline std::size_t length() const
    {
      return impl.length();
    }
}
```

另外，模板实参的个数可能会与主模板的模板参数列表中参数的个数不同，如前面出现过的使用缺省参数和可变长参数的情况

```c++
template<typename... Elements>
class Tuple; //主模板

template<typename T1>
class Tuple<T1>;  //只有一个，原书显然印刷错误

template<typename T1,typename T2,typename... Rest>
class Tuple<T1,T2,Rest...>;
```

最后，再强调一点就是，把主模板、完全特化、部分特化联系起来的是模板的名字，如类名和函数名，如果同名，那么就会一起考虑，有错出错，没错继续。以及，部分特化只适用于类模板和变量模板。

##  16.5 Partial Variable Template Specicialization

这是个依赖于实现的特性，虽然标准已经提出，但是并没有完全的规范，注意是部分特化，不是完全特化，完全特化前面已经讲过了。

```c++
template<typename T> constexpr std::size_t SZ=sizeof(T);
template<typename T> constexpr std::size_t SZ<T&>=sizeof(void*);

template<typename T> typename T::iterator null_iterator;
template<typename T,std::size_t N> T* null_iterator<T[N]>=null_ptr;
```

对类模板部分特化的限制同样也是适用于变量模板部分特化的。

