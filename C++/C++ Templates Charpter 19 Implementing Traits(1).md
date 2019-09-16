# Charpter 19 Implementing Traits(1)

[TOC]



## 19.1 An Example:Accumulating a Squence

### 19.1.1 Fixed Traits

```c++
template<typename T>
T accum(T const* beg,T const* end)
{
	T total{};  //假定这样可以创建一个0值
  while(beg!=end)
  {
    total+=*beg;
    ++bet;
  }
  return total;
}

int main()
{
  int num[]={1,2,3,4,5};
  std::cout<<accum(num,num+5)/5<<std::endl;
  
  char name[]="templates";
  int length=sizeof(name)-1;
  std::cout<<accum(name,name+length)/length<<std::endl;
}
```

运行输出的结果会是

```bash
3
-5
```

-5的原因是对name数组进行累加的结果超出了char所能表示的数值范围，也就是溢出了。所以这里可以这样调用

```c++
accum<int>(name,name+5)
```

不过我们还有别的方法，就是把调用accum()时的类型T和保存累加值的类型关联起来，这个保存值的类型就被称为T的特性，通常用模板特化来表示

```c++
template<typename T>
struct AccumulationTraits;

template<>
struct AccumulationTraits<char>
{
	using AccT=int;
};

template<>
struct AccumulationTraits<short>
{
	using AccT=int;
};

template<>
struct AccumulationTraits<int>
{
	using AccT=long;
};

template<>
struct AccumulationTraits<unsigned ing>
{
	using AccT=unsigned long;
};

template<>
struct AccumulationTraits<float>
{
	using AccT=double;
};

template<typename T>
auto accum(T const* beg,T const* end)
{
  using AccT=typename AccumulationTraits<T>::AccT;
  AccT total{};  //如前
  while(beg!=end)
  {
    total+=*beg;
    ++beg;
  }
  return total;
}
```

这样之后的运行结果就是正常的了。

模板AccumulationTraits称为特性模板，因为它持有其参数的特性。在这里没有提供一个泛型的定义，因为不好确定一个未知类型的trait是啥，不过要提供也是可以的，把最上面那个声明改成如下的定义就可以

```c++
template<typename T>
struct AccumulationTraits
{
  using AccT=T;
};
```

### 19.1.2 Value Traits

如前面所见，我们假定了一个AccT类型可以初始化一个0值，但实际上这是没有保证的，不过没关系，可以给AccumulationsTraits添加一个值trait

```c++
template<typename T>
struct AccumulationTraits;

template<>
struct AccumulationTraits<char>
{
	using AccT=int;
	static AccT const zero=0;
};

template<>
struct AccumulationTraits<short>
{
	using AccT=int;
	static AccT const zero=0;
};

template<>
struct AccumulationTraits<int>
{
	using AccT=long;
	static AccT const zero=0;
};

template<>
struct AccumulationTraits<unsigned int>
{
	using AccT=unsigned long;
	static AccT const zero=0;
};

template<>
struct AccumulationTraits<float>
{
	using AccT=double;
	static constexpr AccT zero=0.0f;  //注意与前面的不同点
};

template<typename T>
auto accum(T const* beg,T const* end)
{
  using AccT=typename AccumulationTraits<T>::AccT;
  Acct total=AccumulationTraits<T>::zero;
  while(beg!=end)
  {
    total+=*beg;
    ++beg;
  }
  return total;
}
```

传统上，当然现在也还是如此，在类的内部能够初始化的静态常量数据成员只能是整型或者枚举类型，但是我们现在可以使用constexpr来初始化非整型或者非枚举的类型，如浮点型。

然而，不论是const还是constexpr都不能用来初始化非字面量类型，这通常指那种会从堆获取存储空间的自定义类型，或者所需的构造函数不是constexpr的。

比方说

```c++
class BigInt
{
  BigInt(long long);
};

template<>
struct AccumulationsTraits<BigInt>
{
	using AccT=BigInt;
	static constexpr BigInt zero=BigInt{0}; //当然是错误的，BigInt不是个字面量类型，其构造函数不是constexpr
};
```

当然，这个问题也不是不可以解决，传统的方法就是把静态变量的声明和定义放在两个地方

```C++
template<>
struct AccumulationTraits<BigInt>
{
	using AccT=BigInt;
	static BigInt const zero;  //只是声明
};

BigInt const AccumulationTraits<BigInt>::zero=BigInt{0};  //这一条通常在某个cpp文件中，上面的声明通常在某个头文件中
```

不过，这里提到这个办法，只是温习一下，实际上在C++17中，像以前说过的，可以

```C++
template<>
struct AccumulationsTraits<BigInt>
{
	using AccT=BigInt;
	inline static BigInt const zero=BigInt{0}; 
};
```

我们还可以使用返回字面量的inline成员函数，这样上面的问题就可以用一种统一的形式来解决了，注意在类内部定义的成员函数默认是inline的，不用再加上inline

```c++
template<typename T>
struct AccumulationTraits;

template<>
struct AccumulationTraits<char>
{
	using AccT=int;
	static constexpr AccT zero()
	{
		return 0;
	}
};

template<>
struct AccumulationTraits<short>
{
	using AccT=int;
	static constexpr AccT zero()
	{
		return 0;
	}
};

template<>
struct AccumulationTraits<int>
{
	using AccT=long;
	static constexpr AccT zero()
	{
		return 0;
	}
};

template<>
struct AccumulationTraits<unsigned int>
{
	using AccT=unsigned long;
	static constexpr AccT zero()
	{
		return 0;
	}
};

template<>
struct AccumulationTraits<float>
{
	using AccT=double;
	static constexpr AccT zero()
	{
		return 0;
	}
};

template<>
struct AccumulationsTraits<BigInt>
{
	using AccT=BigInt;
	static constexpr AccT zero()
	{
		return BigInt{0};
	} 
};

template<typename T>
auto accum(T const* beg,T const* end)
{
  using AccT=typename AccumulationTraits<T>::AccT;
  Acct total=AccumulationTraits<T>::zero();  //注意是函数调用了
  while(beg!=end)
  {
    total+=*beg;
    ++beg;
  }
  return total;
}
```

最后，我们可以看到，trait可以是类型，可以是字面量，可以是函数，总之能跟T关联起来的那么一个实体都可以作为一个trait。或者说，trait就是T的特性，好吧，这里就是同义反复，这个特性并不是T生来具有的，而是赋给的，而赋给的方式则是通过模板特化。

### 19.1.3 Parameterized Traits

前面的函数模板accum()使用的特性被称为固定的，意思就是调用accum()时就确定了使用哪个特性，但是有时候我们可能需要能够自己指定所使用的特性，为此可以增加一个模板参数

```c++
template<typename T,typename AT=AccumulationTraits<T>>
auto accum(T const* beg,T const* end)
{
  typename AT::AccT total=AT::zero();
  while(beg!=end)
  {
    total+=*beg;
    ++beg;
  }
  return total;
}
```

这样，如果不指定AT，则使用AT的缺省值。

## 19.2 Traits versus Policies and Policy Classes

实际上，accumulation不只是summation，对于字符串来说，accumulation也可以是连接，所以对于accum函数来说，`total+=*beg`这个操作对于不同类型的数据而言，应该是有不同意义的或者说可能需要重新定义的，这个操作就是所谓的`policy`。

```c++
class SumPolicy
{
  public:
  	template<typename T1,typename T2>
  	static void accummalte(T1& total,T2 const& value)
  	{
      total+=value;
    }
};

template<typename T,
				 typename Policy=SumPolicy,
				 typename Traits=AccumulationTraits<T>>
auto accum(T const* beg,T const* end)
{
    using AccT=typename Traits::AccT;
    AccT total=Traits::zero();
    while(beg!=end)
    {
      Policy::accumulate(total,*beg);  //当然，这里只是一个示例，不要纠结这里的效率问题
      ++beg;
    }
    return total;
}
```

如前所述，我们还可以定义一个使用乘法进行累记的策略

```c++
class MultPolicy
{
  public:
  	template<typename T1,typename T2>
  	static void accumulate(T1& total,T2 const& value)
  	{
      total*=value;
    }
};
```

然而，如果我们就这么直接使用这个策略，比方说

```C++
int num[]={1,2,3,4,5};
accum<int,MultiPolicy>(num,num+5);
```

那么结果将会是0，因为total的初值是0。这里可以看到，特性和策略有时是有相关性的，在这种时候，就有可能出错，有可能出错，就必定会出错。所以，就这个例子而言，初值也应该是策略的一部分。当然，并不是所有问题都必须通过特性和策略来解决，比方说就像std::accumulate()那样，把初值作为第三个参数传入。

### 19.2.1 Traits and Policies:What’s the Difference?

- **特性**：代表一个模板参数的自然的额外的属性。
- **策略**：代表对泛型函数和类型的可配置的行为。

总之，特性代表属性，策略代表行为。

### 19.2.2 Member Templates versus Template Template Parameters

前面定义策略时使用了普通类的模板成员函数，使用类模板也是可以的，这样在参数列表中的策略就是一个模板模板参数。

```C++
template<typename T1,typename T2>
class SumPolicy
{
  public:
  	static void accumulate(T1& total,T2 const& value)
    {
      total+=value;
    }
};

template<typename T,
				template<typename,typename> class Policy=SumPolicy,
				template<typename Traits=AccumulationTraits<T>>
auto accum(T const* beg,T const* end)
{
	using AccT=typename Traits::AccT;
  AccT total=Traits::zero();
  while(beg!=end)
  {
    Policy<Acct,T>::accumulate(total,*beg);
    ++beg;
  }
  return total;
}
```

当然，特性也是可以使用模板模板参数的形式的，就上面例子而言，我们也可以这样

```C++

template<typename T,template<typename> typename Traits=AccumulationTraits>
class SumPolicy
{
  public:                     
    using SumTraits=Traits<T>;   //实际上，这里的T也是可以作为模板参数另外传入的，
  			//但是出于由策略自己来决定使用哪个Traits的考虑，这里用了T
  			//下面MultiPolicy则直接使用了float
  			//不必在意细节，只是示例
    using AccT=typename SumTraits::AccT; 
    static AccT accumulate(T const* beg,T const* end)
    {
        AccT total=SumTraits::zero();  
        //AccT total=Traits::zero();   
        while(beg!=end)
        {
            total+=*beg;      
            ++beg;            
        }                     
        return total;
    }
}; 

template<typename T,template<typename> typename Traits=AccumulationTraits>
class MultiPolicy
{                             
  public:
    using MulTraits=Traits<float>;  //如前述
    using AccT=typename MulTraits::AccT; 
    static AccT accumulate(T const* beg,T const* end)
    {                         
        AccT total=MulTraits::one();    //与zero类似的一个静态函数，只是返回值为1
        while(beg!=end)
        {
            total*=*beg;
            ++beg;
        }
        return total;
    }
};

//注意这里模板参数中声明Traits和Policy的句法，分别要与上面的定义匹配
//或者说，上面Traits与Policy的定义要与这里的声明匹配
//具体的说就是Traits的“template<typename> typename”是不能与“typename”匹配的
//Policy的“template<typename,template<typename> typename> typename”
//与“template<typename,typename> typename”也是不匹配的，
//注意到这点尤为重要,因为这里是模板模板参数，是不能与模板参数匹配的
template<typename T,
        template<typename,template<typename> typename> typename Policy=SumPolicy,
				template<typename> typename Traits=AccumulationTraits>
auto accum(T const* beg,T const* end)
{
    return Policy<T,Traits>::accumulate(beg,end);
}

int main()
{
  int num[]={1,2,3,4,5};
  std::cout<<accum(num,num+5)/5<<std::endl;
  std::cout<<accum<int,MultiPolicy>(num,num+5)/5<<std::endl;
	...
}
```

### 19.2.3 Combining Multiple Policies and/or Traits

有多个策略和特性同时存在时，安排它们的顺序是个有趣的问题。一个简单的考虑就是根据使用它们的缺省值的可能性升序排列，也就是说，最有可能使用缺省值的参数放在最后。实际上，这样大多数情况下特性都会放在策略后面，就像上面例子那样。

### 19.2.4 Accumulation with General Iterators

实际上，accum函数是可以使用迭代器的，并且使用迭代器的版本也是可以匹配指针的，因为标准库的迭代器提供了迭代器特性来支持指针。

```c++
#include <iterator>

template<typename Iter>
auto accum(Iter start,Iter end)
{
  using VT=typename std::iterator_traits<Iter>::value_type;
  VT total{};  //假定这样可以初始化一个0值
  while(start!=end)
  {
    total+=*start;
    ++start;
  }
  return total;
}
```

大体上，iterator_traits在std中是这样的

```c++
namespace std
{
  template<typename T>
  struct iterator_traits<T*>
  {
  	using difference_type=ptrdiff_t;
  	using value_type = T;
  	using pointer=T*;
  	using reference=T&;
  	using iterator_category=random_access_iterator_tag;
  }
}
```

因此，如果我们传入一个指针，value_type就是这个指针所指的对象的类型。

那么，如果我们传入的就是一个迭代器，这时候的value_type又会是什么？比方说

```c++
std::vector<float> vf{1,2,3,4,5};
accum(vf.begin(),vf.end());
```

实际上，对于iterator_traits还有另外一个定义，或者说特化

```c++
template< class Iter >
struct iterator_traits
{
  using difference_type=Iter::difference_type;
	using value_type=Iter::value_type;
	using pointer=Iter::pointer;
	using reference=Iter::reference;
	using iterator_category=Iter::iterator_category;
};
```

于是，

`std::iterator_traits<std::vector<float>::iterator>::value_type`=`std::vector<float>::iterator::value_type` = float。

## 19.3 Type Functions

前面特性的例子说明了我们可以根据类型来决定行为。所谓值函数，就是用值作为参数，返回另外一个值作为结果，而通过模板，我们可以类型函数，也就是用类型做参数，然后生成一个类型或者常量做未结果。

类型函数的典型例子就是sizeof，我们也可以用模板来封装一下sizeof

```c++
template<typename T>
struct TypeSize
{
  static std::size_t const value=sizeof(T);
};
```

就这个封装本身看起来意义不大，因为我们直接使用sizeof就足够了，但是，要注意`TypeSize<T>`实际上是个类型，因而是可以作为模板参数使用的，另一方面，TypeSize是个模板，因而也是可以作为模板模板参数使用的。或者说，这个封装拓展了sizeof的使用范围，毕竟sizeof返回的是一个常量，但也就这么多了，就这个例子而言。另外，sizeof可以接受一个变量作为参数，而`TypeSize<T>`的参数必须是个类型。

```c++
template<typename T>
struct TypeSize
{
    static std::size_t const value=sizeof(T);
};

template<template<typename> typename TTP,typename T>
struct TtpSize
{
    static std::size_t const value=sizeof(TTP<T>);
};

class t0
{
    static int const value=2;
};
class t1
{
    static int const value=2;
    long long b;
};

class t2
{   
    static int const value=2;
    public: 
        int aa;
        explicit t2(int a):aa{a}{}
};
class t3
{
};

int main()
{
    std::cout<<TypeSize<int>::value<<std::endl;
  	std::cout<<TypeSize<TypeSize<int>>::value<<std::endl;
    std::cout<<TtpSize<TypeSize,int>::value<<std::endl;
    std::cout<<TtpSize<TypeSize,float>::value<<std::endl<<std::endl;

    t0 vt0;
    std::cout<<TypeSize<t0>::value<<std::endl;
    std::cout<<sizeof(vt0)<<std::endl;
    t1 vt1;
    std::cout<<TypeSize<t1>::value<<std::endl;
    std::cout<<sizeof(vt1)<<std::endl;
    t2 vt2(3);
    std::cout<<TypeSize<t2>::value<<std::endl;
    std::cout<<sizeof(vt2)<<std::endl;
    t3 vt3;
    std::cout<<TypeSize<t3>::value<<std::endl;
    std::cout<<sizeof(vt3)<<std::endl;

}
```

结果是

```bash
4
1
1
1

1
1
8
8
4
4
1
1
```

其实也没什么可解释的，标准便是如此规定，只是提醒下空类的大小为1，成员函数以及静态数据成员不占类的空间。

### 19.3.1 Element Types

可以使用部分特化来实现这样一个函数，给定一个容器类型作为参数，得到元素的类型。这里说的容器类型包括但不限于`std::vector<>`，`std::list<>`这样的模板以及内建数组。

```c++
template<typename T>
struct ElementT;

template<typename T>
struct ElementT<std::vector<T>>
{
	using Type=T;
};

template<typename T>
struct ElementT<std::list<T>>
{
	using Type=T;
};

template<typename T,std::size_t N>
struct ElementT<T[N]>
{
	using Type=T;
};

template<typename T>
struct ElementT<T[]>
{
	using Type=T;
};

... //注意，我们需要提供针对各种类型数组的部分特化，边界已知的，边界未知的，引用的，非引用的，以及指针
  
template<typename T>
void printElementType(T const& c)
{
  std::cout<<typeid(typename ElementType<T>::Type).name()<<std::endl;
}
```

这样定义的类型函数对于容器是透明的，也就是说容器不需要知道这个函数的存在。

然而，一些容器本身定义了像value_type这样的成员类型，因此我们可以提供一个主模板

```c++
template<typename T>
struct ElementT
{
  using Type=typename T::value_type;
};
```

另外，在模板中通常也会对类模板参数定义一个成员类型，或者说提供一个别名，但不仅仅只是一个别名的作用，而是可以作为类成员使用的。

```C++
template<typename T1,typename T2,...>  //这里...不是可变参数的意思，只是表示省略
class X
{
  using ...=T1;
  using ...=T2;
};
```

类型函数使我们在使用容器类时，不用额外关注其元素类型。

比方说，对于

```c++
template<typename T,typename C>
T sumOfElements(C const& c);
```

我们通常需要这样来调用`sumOfElements<int>(list)`。但如果我们这样定义

```C++
template<typename C>
typename ElementT<C>::Type sumOfElements(C const& c);
```

则只需要这样来调用`sumOfElements(list)`。

实际上，我们还可以使用别名模板

```c++
template<typename T>
using ElementType=typename ElementT<T>::Type;

template<typename C>
ElementType<C> sumOfElements(C const& c);
```

这里ElementT被称为特性类，因为它被用来访问一个给定容器类C的特性。

实际上特性类不单单只用来描述容器参数的属性，而是可以访问各种各样的所谓主参数的特性的。

### 19.3.2 Transformation Traits

除了访问主模板参数的某个特定属性，特性还可以用来类型转移，比方说增加或者移除引用、const和volatile。

#### Removing References

有时候，一个类型在构造的时候就成为了引用类型，比方说在对函数模板参数进行推断时，由于引用坍缩，T&&有时候会变成T&，这时候移除引用就很重要了。

```c++
template<typename T>
struct RemoveReferenceT
{
  using Type=T;
};

template<typename T>
struct RemoveReferenceT<T&>
{
  using Type=T;
};

template<typename T>
struct RemoveReferenceT<T&&>
{
  using Type=T;
};

template<typename>
using RemoveReferenc=typename RemoveReferenceT<T>::Type;
```

c++标准库也提供了一个特性：`std::remove_reference<>`。

#### Adding References 

类似的，我们也可以给一个类型加上左值或者右值引用。

```c++
template<typename T>
struct AddLValueReferenceT
{
  using Type=T&;
};

template<typename T>
using AddLValueReference=typename AddLValueReferenceT<T>::Type;

template<typename T>
struct AddRValueReferenceT
{
  using Type=T&&;
};

template<typename T>
using AddRValueReference=typename AddRValueReferenceT<T>::Type;
```

这里引用坍缩的规则仍然适用，比方说调用`AddLValueReference<int&&>`会得到`int&`，因为`&&`和`&`得到`&`。因此不需要专门为&和&&提供部分特化。

实际上，如果我们不需要提供任何部分特化的话，直接定义成这样就可以了

```c++
template<typename T>
using AddLValueReferenceT=T&;
template<typename T>
using AddRValueReferenceT=T&&;
```

但是，这样就会缩减了这两个特性的使用范围，因为这时候不能使用void做参数，而必须使用最初的定义，因为需要特化，但别名模板是不会拿来特化的。

```c++
template<>
struct AddLValueReferenceT<void>
{
	using Type=void;
};

template<>
struct AddLValueReferenceT<void const>
{
	using Type=void const;
};

template<>
struct AddLValueReferenceT<void volatile>
{
	using Type=void volatile;
};

template<>
struct AddLValueReferenceT<void const volatile>
{
	using Type=void const volatile;
};
```

c++标准库也提供了相应的特性：`std::add_lvalue_reference<>`和`std::add_rvalue_reference<>`，并且都包括了对void的特化。

#### Removing Qulifiers

```c++
template<typename T>
struct RemoveConstT
{
  using Type=T;
};

template<typename T>
struct RemoveConstT<T const>
{
  using Type=T;
};

template<typename T>
using RemoveConst=typename RemoveConst<T>::Type;

```

转移特性还可以组合

```c++
template<typename T>
struct RemoveCVT:RemoveConstT<typename RemoveVolatileT<T>::Type>
{};

template<typename T>
using RemoveCV=typename RemoveCVT<T>::Type;
```

显然，RemoveCVT是RemoveConstT的子类，而RemoveConstT是一个类模板实例，其模板参数是`RemoveVolatileT<T>::Type`，也就是去除了T的volatile修饰之后的Type，RemoveConstT本身的Type成员则是把`RemoveVolatileT<T>::Type`的const去除的结果。

这里要注意两件事：

1. 使用RemoveConstT和RemoveVolatileT的顺序没有关系，反过来也是一样的。
2. RemoveCVT继承了RemoveConstT的成员Type，而不是自己声明一个，因为RemoveCVT中的Type不会有任何变化。

我们也可以使用别名模板，如果RemoveCVT不需要特化的话。

```c++
template<typename T>
using RemoveCV=RemoveConst<RemoveVolatile<T>>;
```

C++标准库也提供了相应的特性：`std::remove_volatile<>`，`std::remove_const`，`std::remove_cv<>`。

#### Decay

所谓退化，在c语言中指的是从数组和函数类型变成相应的指针类型，在c++中还包括移除或者忽略掉最顶层的const、volatile，以及引用。

```c++
template<typename T>
void f(T)
{
  
}

template<typename A>
void printParameterType(void (*)(A))
{
  std::cout<<"parameter type:"<<typeid(A).name()<<std::endl;
  std::cout<<"- is int:"<<std::is_same<A,int>::value<<std::endl;
  std::cout<<"- is const:"<<std::is_const<A>::value<<std::endl;
  std::cout<<"- is pointer:"<<std::is_pointer<A>::value<<std::endl;
}

int main()
{
  printParameterType(&f<int>);
  printParameterType(&f<int const>);
  printParameterType(&f<int[7]);
  printParameterType(&f<int(int)>);
}
```

运行结果是

```bash
parameter type:i
- is int:1
- is const:0
- is pointer:0
parameter type:i
- is int:1
- is const:0
- is pointer:0
parameter type:Pi
- is int:0
- is const:0
- is pointer:1
parameter type:PFiiE
- is int:0
- is const:0
- is pointer:1
```

可以看到，const被移除了，数组和函数变成了指针。另外要注意的是调用printParameterType的句法，显然其参数是个函数指针，而实际参数&f并不能带括号及参数，比方说类似于`printParameterType(f(7))`这样的，因为在这时候，首先会计算f(7)，而f函数的返回值是void，这与printParameterType的签名是不符的。不过我们可以`printParameterType(f<int>)`或者`printParameterType<int>(f)`。或者说，printParamaterType的模板参数是从其函数实际参数推断出来的，并且这个模板参数就是其函数实际参数的模板参数。因此，用函数指针做参数时，并不能简单的把这个函数的参数带上，但是可以带上其模板参数。当然，这里只是针对本例而言的解释，其它情况就还要具体分析了。

```c++
template<typename T>
struct DecayT:RemoveCVT<T>
{
//如前所述，继承了Type，而不是自己在顶一个。
};
template<typename T>
struct DecayT<T[]>
{
	using Type=T*;
};
template<typename T,std::size_t N>
struct DecayT<T[N]>
{
	using Type=T*;
};
template<typename R,typename... Args>
struct DecayT<R(Args...)>
{
  using Type=R (*)(Args...);
};
template<typename R,typename... Args>
struct DecayT<R(Args...,...)> //这个特化匹配C风格的可变长参数函数
{
	using Type=R (*)(Args...,...);
};
template<typename T>
void printDecayedType()
{
  using A=typename DecayT<T>::Type;
  
  std::cout<<"parameter type:"<<typeid(A).name()<<std::endl;
  std::cout<<"- is int:"<<std::is_same<A,int>::value<<std::endl;
  std::cout<<"- is const:"<<std::is_const<A>::value<<std::endl;
  std::cout<<"- is pointer:"<<std::is_pointer<A>::value<<std::endl;
}

int main()
{
  printDecayedType<int>();
  printDecayedType<int const>();
  printDecayedType<int[7]>();
  printDecayedType<int(int)>();
}
```

运行结果是

```bash
parameter type:i
- is int:1
- is const:0
- is pointer:0
parameter type:i
- is int:1
- is const:0
- is pointer:0
parameter type:Pi
- is int:0
- is const:0
- is pointer:1
parameter type:PFiiE
- is int:0
- is const:0
- is pointer:1
```

与前面的例子结果是一样的。只不过前面的例子说明了自动进行的退化，后面这个例子说明了收工进行的退化。

当然，我们还是可以使用别名模板

```c++
template <typename T>
using Decay=typename DecayT<T>::Type;
```

这时候printDecayedType就会变成

```c++
template<typename T>
void printDecayedType()
{
  std::cout<<"parameter type:"<<typeid(Decay<T>).name()<<std::endl;
  std::cout<<"- is int:"<<std::is_same<Decay<T>,int>::value<<std::endl;
  std::cout<<"- is const:"<<std::is_const<Decay<T>>::value<<std::endl;
  std::cout<<"- is pointer:"<<std::is_pointer<Decay<T>>::value<<std::endl;
}
```
同样的，c++标准库也提供了`std::decay<>`。

### 19.3.3 Predicate Traits

前面所看到的类型函数都是一个参数的例子，实际上参数可以有很多个，而类型断言就是这种类型特性的典型例子。

#### IsSameT

```C++
template<typename T1,typename T2>
struct IsSameT
{
  static constexpr bool value=false;
};
template<Typename T>
struct IsSameT<T,T>
{
  static constexpr bool value=true;
};
```

这里需要理解的是，类型决定之后才会决定使用哪个特化，所以如果两个参数的类型一致，会使用第二个特化，如果不一致，会使用第一个。事实上，这就是模板实例化的规则。

对于生成常量的特性，我们是不能使用别名的，不过可以使用constexpr

```c++
template<typename T1,typename T2>
constexpr bool isSame=IsSameT<T1,T2>::value;
```

c++标准库也提供了`std::is_same<>`。

#### true_type and false_type

我们还可以改进上面IsSameT的定义

```C++
template<bool val>
struct BoolConstant
{
  using Type=BoolConstant<val>;
  static constexpr bool value=val;
};
using TrueType=BoolConstant<true>;  //Type=BoolConstant<true>,value=true
using FalseType=BoolConstant<false>;

template<typename T1,typename T2>
struct IsSameT:FalseType
{

};
template<typename T>
struct IsSameT<T,T>:TrueType
{

};

template<typename T>
void fooImpl(T,TrueType)
{
  std::cout<<"fooImpl(T,true) for int called\n";
}
template<typename T>
void fooImpl(T,FalseType)
{
  std::cout<<"fooImpl(T,false) for other type called\n";
}

template<typename T>
void foo(T t)
{
  fooImpl(t,IsSameT<T,int>{});  //注意这里的{}，根据T的类型初始化一个TrueType或者FalseType的临时对象
}

int main()
{
  foo(42);  //TrueType
  foo(7.7); //FalseType
}
```

这里的意义在于，可以在编译时把`IsSameT<T,int>`隐式转换成其基类TrueType或者FalseType，除了仍然可以得到其成员value的值，还可以根据其类型，这里是TrueType或者FalseType，使用不同的函数或者部分类模板特化。也就是说，就函数来说是重载，就类来说是部分特化。这种技术被称为tag dispatching。

原书上关于IsSame的定义是有问题的，因为仅仅使用`IsSameT<T>`是不符合IsSameT的签名的。而且，就上面例子而言，使用IsSame代替IsSameT也没什么实际意义，因为仅仅只是少打一个T，该用`{}`还是得用。所以就无视这一段了。

通常来说，生成布尔值的特性通过继承像TrueType和FalseType这样的类型来实现对标签派发的支持。为了避免每个泛型库都各自定义自己的布尔常量true和false的类型，c++标准库在<type_traits>中做了如下定义

c++11和c++14

```c++
namespace std
{
  using true_type=integral_constant<bool,true>;
  using false_type=integral_constant<bool,false>;
}
```

c++17

```c++
namespace std
{
  using true_type=bool_constant<true>;
  using true_false=bool_constant<false>;
}

template<bool B>
using bool_constant=integral_constant<bool,B>;
```

### 13.3.4 Result Type Traits

```c++ 
template<typename T1,typename T2>
struct PlusResult
{
  using Type=decltype(T1()+T2());
};
template<typename T1,typename T2>
using PlusResult=typename PlusResult<T1,T2>::Type;

template<typename T1,typename T2>
Array<PlusResult<T1,T2>>
operator+(Array<T1> const&,Array<T2> const&);
```

正如以前说过的的，像operator+这样的函数，其返回值的类型是必须决定的，以前介绍过一些方法，这里使用了trait模板，通过decltype来计算返回值的类型，也就是说把类型提升以及运算符重载的动作交给了编译器。

但这里的问题是，decltype所保存的东西太多，主要指的是引用、const及volatile这些，比方说，对于这么个类以及+运算

```c++
class Integer {...};
Integer const operator+(Integer const&,Integer const&);
```

那么两个`Array<Integer>`会怎么样呢？因为decltype里边的+运算使用的就是上面这个重载+的定义，而其返回值是一个Integer const类型，也就是说`operator+(Array<T1> const&,Array<T2> const&)`的返回值是`Array<Integer const>`，这点可能不是我们想要的。注意这里说的是decltype里边的相加运算的返回值，这是由+运算符的重载定义决定的，与推断无关的。所以，我们需要

```c++
template<typename T1,typename T2>
Array<RemoveCV<RemoveReference<PlusResult<T1,T2>>>>
operator+(Array<T1> const&，Array<T2> const&);
```

注意，这里对T1和T2实际上是有限制的：必须有可访问未删除的的缺省构造函数，或者就不是一个类类型。

#### declval

不过，c++标准提供了`std::declval<>`来解决这个问题，其定义在<utility>中：

```c++
namespace std
{
  template<typename T>
  add_rvalue_reference_t<T> declval() noexcept;
}
```

这个函数模板是没有定义的，这也是有意为之，它只能用在decltype，sizeof当中或者其他一些不需要定义的地方，除此外其它地方是不能显式调用的。我们可以理解为编译器碰到这个符号就去构造出一个值来，而不是进行通常的函数调用工作，也就是类似于inline函数的就地展开。

`declval<T>()`可以生成一个类型T的值，而不需要一个缺省构造函数或者其他的什么操作。或者说，这个函数返回的是一个类型的右值引用，不需要做任何计算，只要能找到模板参数的类型定义，就能返回这个类型的一个右值引用。

要注意的是对于可引用的类型，其返回值总是一个对这个类型的右值引用类型，这样declval甚至可以应用于那些通常无法从函数返回的类型上，比方说抽象类类型(有纯虚函数的类)和数组类型。另外，对于`declval<T>()`而言，如果T是一个对象类型，那么无论T还是T&&都是一样的，因为反正返回的都是一个右值引用，而如果T是一个左值引用，由于引用坍缩，那么返回的还是一个左值引用。

```c++
template<typename T1,typename T2>
struct PlusResultT
{
  using Type=decltype(std::declval<T1>()+std::declval<T2>());
};

template<typename T1,typename T2>
using PlusResult=typename PlusResult<T1,T2>::Type;
```

这里不管T1、T2类型如何定义，通过declval总能构造出相应的对象，decltype就可以决定运算结果的类型。

通过结果类型特性，我们可以精确地知道一个运算的返回类型，对于描述函数模板的结果类型特别有用。

