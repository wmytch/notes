# Charpter 19 Implementing Traits

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

