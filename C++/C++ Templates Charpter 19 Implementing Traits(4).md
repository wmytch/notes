# Charpter 19 Implementing Traits(4)

[TOC]

## 19.8 Type Classification

有时候我们会需要确定一个模板参数是一个什么样的类型，然后在程序中就可以这样使用

```c++
if(IsClass<T>::value) {...}

if constexpr (IsClass<T>) {...}

template<typename T,bool=IsClass<T>>
class C {...};
template<typename T>
class C<T,true>{...};
```

此外，像`IsPointerT<T>::value`这样的表达式，会是一个布尔常量，因而可以作为合法的非类型模板参数，从而可以构造出各种各样的模板来。

另外要注意在一些情况下，像上面例子这样，在模板参数中直接使用`IsClass<T>`这样的非类型参数有可能是非SFINAE友好的，如前面章节所言。

### 19.8.1 Determining Fundamental Types

```c++
template<typename T>
struct IsFundaT:std::false_type {};
#define MK_FUNDA_TYPE(T)		\
template<> struct IsFundaT<T>:std::true_type {};

MK_FUNDA_TYPE(void)
MK_FUNDA_TYPE(bool)
MK_FUNDA_TYPE(int)
MK_FUNDA_TYPE(char32_t)
MK_FUNDA_TYPE(std::nullptr_t)
....
template<typename T>
void test(T const&)
{
    if(IsFundaT<T>::value)
    {
        std::cout<<"funda"<<std::endl;
    }
    else
    {   
        std::cout<<"not funda"<<std::endl;
    }
}

int main()
{
    test(7);  //funda
    test("hello"); //not funda
    test(2<3);   //funda
    test(nullptr); //funda
    test(std::vector<int>{});  //not funda
}
```

这里要提醒的只是，true_type/false_type类型都有个value值，所以不需要在模板中再定义一次。

用同样的方法，还可以定义IsIntegralT和IsFloatingT，顾名思义就不解释了。

c++标准库中也有类似的特性，不过比这里的例子划分的更为精细，比方说is_void，is_integral，is_pointer这样的划分，具体就看文档吧。

### 19.8.2 Determining Compound Types

枚举也是组合类型，但不被认为是简单复合类型，因为其并不是由多个依赖类型构造出来的。简单复合类型可以通过部分特化归类出来。

#### Pointers

```c++
template<typename T>
struct IsPointerT:std::false_type {};
template<typename T>
struct IsPointerT<T*>:std::true_type
{
	using BaseT=T;
};
```

std::is_pointer<>没有提供像BaseT这样的成员。

#### References

```c++
template<typename T>
struct IsLValueReferenceT:std::false_type {};
template<typename T>
struct IsLValueReferenceT<T&>:std::true_type 
{
	using BaseT=T;
};

template<typename T>
struct IsRValueReferenceT:std::false_type {};
template<typename T>
struct IsRValueReferenceT<T&&>:std::true_type 
{
	using BaseT=T;
};
```

可以组合起来

```c++
template<typename T>
class IsReferenceT:public IfThenElseT<IsLValueReferenceT<T>::value,	
									  IsLValueReferenceT<T>, 
									  IsRValueReferenceT<T>
						   >::Type
{
};
```

C++标准库也提供了`std::is_lvalue_reference<>`和`std::is_rvalue_reference<>`。

#### Arrays

```c++
template<typename T>
struct IsArrayT:std::false_type {};
template<typename T,std::size_t N>
struct IsArrayT<T<N>>:std::true_type  //#1
{
	using BaseT=T;
	static constexpr std::size_t size=N;
};
template<typename T>
struct IsArrayT<T[]>::std::true_type  
{
	using BaseT=T;
	static constexpr std::size_t size=0;
};
```

c++标准库有`std::is_array<>`，以及用来查询数组维度的`std::rank<>`和查询某个维度大小的`std::extent<>`。

要注意的是部分特化的模板参数列表可以跟主模板的模板参数列表不同，但是部分特化模板模板**实参**是要与主模板的参数列表对应的。以上面为例，主模板参数列表只有一个类型参数T，而部分特化模板#1的模板参数列表有两个参数，但是其特化的实参只有一个`T<N>`。而如果主模板有缺省参数，特化模板可以提供也可以不提供对应的参数。这里只是提醒下。

#### Pointers to Members

```c++
template<typename T>
struct IsPointerToMemberT:std::false_type {};
template<typename T,typename C>
struct IsPointerToMemberT<T C::*>:std::true_type
{
	using MemberT=T;
	using ClassT=C;
};
```

对应的标准库有`std::is_member_object_pointer<>`和`std::is_member_function_pointer<>`，以及`std::is_member_pointer<>`。

### 19.8.3 Identifying Function Types

```c++
template<typename T>
struct IsFunctionT:std::false_type {};
template<typename R,typename... Params>
struct IsFunctionT<R (Params...)>::std::true_type
{
	using Type=R;
	using ParamsT=Typelist<Params...>;
	static constexpr bool variadic=false;
};
template<typename R,typename... Params>
struct IsFunctionT<R (Params...,...)>:std::true_type
{
	using Type=R;
	using ParamsT=Typelist<Params...>;
	static constexpr bool variadic=true;
};
```

注意上面的variadic是为了表明函数是不是采用了C风格的可变长参数。

然而，这个IsFunctionT并不能适应所有的函数类型，比方说`using MyFuncType=void (int&) const;`，显然这只能是一个成员函数，然而这里的const并不能移除掉。此外，函数类型还可以被各种修饰符修饰，如`const/volatile/&/&&/noexcept`，所以需要对应的各种部分特化，据称目前统计出来有48种，就举几个例子好了

```c++
template<typename R,typename... Params>
struct IsfunctionT<R (Params...) const>:std::true_type
{
	using Type=R;
	using ParamsT=Typelist<Params...>;
	static constexpr bool variadic=false;
};
template<typename R,typename... Params>
struct IsfunctionT<R (Params...,...) volatile>:std::true_type
{
	using Type=R;
	using ParamsT=Typelist<Params...>;
	static constexpr bool variadic=true;
};
template<typename R,typename... Params>
struct IsfunctionT<R (Params...,...) const volatile>:std::true_type
{
	using Type=R;
	using ParamsT=Typelist<Params...>;
	static constexpr bool variadic=true;
};
template<typename R,typename... Params>
struct IsfunctionT<R (Params...,...) &>:std::true_type
{
	using Type=R;
	using ParamsT=Typelist<Params...>;
	static constexpr bool variadic=true;
};
template<typename R,typename... Params>
struct IsfunctionT<R (Params...,...) const&>:std::true_type
{
	using Type=R;
	using ParamsT=Typelist<Params...>;
	static constexpr bool variadic=true;
};
```

标准库对应的是`std::is_function<>`。

### 19.8.4 Determining Class Types

实际上并不能直接检测类类型。对类类型而言，最方便使用的属性是只有类才能够被用于指向成员指针的基础类型。也就是说像`X Y::*`这样的表达式，Y必须是类类型才是合法的表达式，再配合使用SFINAE友好的特性，就可以定义一个IsClassT

```c++
template<typename T,typename=std::void_t<>>
struct IsClassT:std::false_type {};
template<typename T>
struct IsClassT<T,std::void_t<int T::*>>:std::true_type {};  //int是任意选的
```

这个特性对于lambda表达式和union都是得到true的。

```c++
auto l=[]{};
static_assert<IsClassT<decltype(l)>::value,"">; //成功
```

这里只是要注意IsClassT只提供了一个模板实参，因为主模板有缺省参数，于是这个调用就等价于`IsClassT<decltype(l),void>`，然后会对所有模板进行评估，而实际上，不论是主模板还是部分特化模板，参数替换之后如果不出错都是这个结果，由于特化模板优先，于是选择部分特化。

标准库对应的有`std::is_class<>`和`std::is_union<>`。不过，这些特性需要编译器的特别支持，因为标准语言核心技术不能把class/struct与union区分开来。

### 19.8.5 Determining Enumeration Types

对枚举类型而言，可以通过检测其是否可以**显式**的转换成整型，同时又不是标准类型、类类型、引用类型、指针类型以及成员指针类型，这些其他类型都可以转换成整型，但又不是枚举类型。但是，我们可以发现，只要不是其他的任意类型，那么就必然是个枚举类型

```c++
template<typename T>
struct IsEnumT{
    static constexpr bool value=!IsFundaT<T>::value &&
        						!IsPointerT<T>::value &&
                                !IsReferenceT<T>::value &&
                                !IsArrayT<T>::value &&
                                !IsPointerToMemberT<T>::value &&
                                !IsFunctionT<T>::value &&
                                !IsClassT<T>::value;
};
```

标准库有`std::is_enum<>`。不过，编译器通常应该能够直接支持对枚举的检测，而不是像这样定义成一个*其它*。

## 19.9 Policy Traits

之前所讨论的特性，都是用来确定模板参数的属性的：参数的类型，以及对那个类型做了运算之后所得到结果的类型，等等。所以，这些特性被称为*属性特性*。对应的所谓的*策略特性*，则是定义了对一些类型可以做的操作。虽然策略跟特性的界限不是那么清楚，不过策略特性更强调与模板参数关联的独特的属性，而策略类则通常是独立于其他模板参数的。

属性特性通常通过类型函数来实现，而策略特性则通常封装成成员函数，当然，并不绝对。

### 19.9.1 Read-Only Parameter Types

传值还是传引用，在c和c++中不是一个小问题。通常来说，可以通过一个策略特性来决定一个模板的类型成员是值类型还是引用类型

```c++
template<typename T>
struct RParam
{
    using Type=typename IfThenElseT<sizeof(T)<=2*sizeof(void*),T,T const&>::Type;
};
```

这就是所谓的类型函数，这个函数把参数类型T映射到T或者T const&。可以看到，策略特性也是可以通过类型函数来实现的。

另外一方面，容器类型通常会有昂贵的复制构造函数，因此，即使其sizeof返回一个很小的值，这时候用传值也是不合适的，所以还需要加上许多特化，比如

```c++
template<typename T>
struct RParam<Array<T>>
{
	using Type=Array<T> const&;
};
```

不过我们也可以

```c++
template<typename T>
struct RParam
{
    using Type =
        IfThenElse<(sizeof(T)<=2*sizeof(void*) &&
        			std::is_trivially_copy_constructible<T>::value &&
                    std::is_trivially_move_constructible<T>::value),
    				T,T const&>;
 };
```

注意这里用的是别名模板，只是提醒下。

假定有两个类，其中一个类对于只读参数使用传值更好，比方说下面定义中的MyClass2

```c++
class MyClass1
{
    public:
    	MyClass1()
        {
            std::cout<<"MyClass1 default constructor\n";
        }
    	MyClass1(MyClass const&)
        {
            std::cout<<"MyClass1 copy constructor\n";
        }
};
class MyClass2
{
     public:
    	MyClass2()
        {
            std::cout<<"MyClass2 default constructor\n";
        }
    	MyClass2(MyClass const&)
        {
             std::cout<<"MyClass2 copy constructor\n";
        }
};
template<>
class RParam<MyClass2>
{
	public:
		using Type=MyClass2;
};

template<typename T1,typename T2>
void foo(typename RParam<T1>::Type p1, typename RParam<T2>::Type p2)
{
    ...
}
int main()
{
    MyClass1 mc1;
    MyClass2 mc2;
    foo<MyClass1,MyClass2>(mc1,mc2);
}
```

这里可以看到RParam的用法，不过就这个问题，使用RParam的不足之处在于，首先，foo函数的声明很啰嗦，其次，无法使用类型推断。所以，我们可以用一个函数封装一下

```c++
template<typename T1,typename T2>
void foo_core(typename RParam<T1>::Type p1, typename RParam<T2>::Type p2)
{
    ...
}
template<typename T1,typename T2>
void foo(T1&& p1, T2&& p2)
{
    foo_core<T1,T2>(std::forward<T1>(p1),std::forward<T2>(p2));
}
int main()
{
    MyClass1 mc1;
    MyClass2 mc2;
    foo(mc1,mc2);
}
```

不过，对于上面的代码，foo的两个参数都是右值引用，并且由于完美转发，之后并没有再构造新的对象，也就是这段代码只是在声明mc1和mc2的时候使用了缺省构造函数，之后就没有再调用任何一个构造函数。具体怎么改才能达到预期的结果，就不讨论了，反正问题已经指明。

## 19.10 In the Standard Library

像前面所说的`std::is_trivially_copy_constructible<T>`和`std::is_trivially_move_constructible<T>`以及`std::is_union`这样的特性，C++语言本身并没有解决方案，而是依赖于编译器实现。对于语言本身就有的解决方案，编译器也提供了自己的支持，以提高编译效率。

## 19.11 Afternotes

客户代码(比如标准库的使用者)通常不用处理特性，缺省的特性类已经可以安全的解决大多数需求了，并且因为是缺省的，也不会出现在客户代码中。所以缺省特性的名字都比较长，如果客户代码自己提供了一个特性代替缺省的特性参数，这个特性类的名字最好遵循标准库的命名规则，然后声明一个别名，这样就两全其美了。

特性还可以作为一种反射的形式，像IsClassT和PlusResultT这样的特性，被称为编译时反射。