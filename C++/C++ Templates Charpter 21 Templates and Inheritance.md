# Charpter 21 Templates and Inheritance

[TOC]

## 21.1 The Empty Base Class Optimization(EBCO)

C++类的类型成员、非虚成员函数以及静态数据成员通常在运行时其内部表示不占类本身的内存空间。而另外一方面，非静态数据成员、虚函数以及虚基类运行时还是需要占用一些空间的。这里指的是运行时一个类实例的内存占用情况，当然，如我们曾经看到的，即便是对一个没有任何成员的类类型做sizeof运算也不会输出0，而通常会是1或者别的一个数字，比方说由于严格的对齐规则，有些编译器会输出4。要区分类的定义和类类型本身，类的定义本身并不会占用空间，只是作为编译时的一个参照。

### 21.1.1 Layout Principles

避免类的大小为0有很多原因。比方说，一个大小为0的类的数组必然大小也为0，这样就会使得通常的指针运算失效。比方说

```c++
ZeroSizedT z[10];  //ZeroSizedT意思如字面所示
...
&z[i]-&z[j]; //两个指针或者地址之间的距离，以元素大小为单位
```

通常来说像上面求差运算是通过用元素的大小来除两个地址之间的字节数得到的，如果类的大小为0的话，显然这是无法实现的。顺便说句，有时候逛nga或者hupu，时不时就能看到讨论除和除以到底有没有区别的，再次验证这俩地方用户的平均学历水平不会超过高中。

c++标准明确指明，如果一个空类作为基类，那么不需要为这个空基类另外分配空间，只要其地址不与同类型的另一个对象或者子对象相同。这里是什么意思呢？首先这是一条优化规则，前提是满足某个条件，也就是说，编译器可以实现也可以不实现，当然通常都是会实现的。其次，是不另外分配空间，而不是不占任何空间，也就是说，与继承其的子类共享空间，并且有相同的地址。最后，所满足的条件事实上指的是没有多个同类型的空基类对象共享一个地址，需要说明的是前面提到的子对象指的是继承了同样一个基类的另外一个子类。

```c++
class Empty
{
    using Int = int;
};
class EmptyToo:public Empty{};
class EmptyThree:public EmptyToo{};
class NonEmpty:public Empty,public EmptyToo{};

int main()
{
    std::cout<<"sizeof(Empty): "<<sizeof(Empty)<<std::endl;
    std::cout<<"sizeof(EmptyToo): "<<sizeof(EmptyToo)<<std::endl;
    std::cout<<"sizeof(EmptyThree): "<<sizeof(EmptyThree)<<std::endl;
    std::cout<<"sizeof(NonEmpty): "<<sizeof(NonEmpty)<<std::endl;
}                                        
```

通常结果如下

```bash
sizeof(Empty): 1
sizeof(EmptyToo): 1
sizeof(EmptyThree): 1
sizeof(NonEmpty): 2
```

实际上，编译的时候可能会有一条警告

```bash
 warning: direct base 'Empty' is inaccessible due to ambiguity:
    class NonEmpty -> class Empty
    class NonEmpty -> class EmptyToo -> class Empty [-Winaccessible-base]
class NonEmpty:public Empty,public EmptyToo{};

```

前面三条输出，说明没有为空基类另外分配空间，否则输出应该是1 2 3。而第四条输出，说明为其中一个空基类另外分配了空间，否则其两个基类就会指向同一个地址，从而输出1，这也就是上面约束条件所要表达的意思，这条警告实际上也说明了这个问题。因为不论如何，NonEmpty都会被分配空间，它有两个基类，如前所述，两个都是大小为1的空类，如果不给其中一个基类分配空间，那么这两个基类就会指向同样的一个地址，这个地址也是NonEmpty的地址，这就是标准指明要排除的情况，这里之所以要避免两个同样类型的对象分配到同一个地址，可以参见前面数组的例子，为了满足常规的指针运算规则。

继承空类的意义是什么呢？前面我们已经看到有很多的操作依赖于引入一个类的别名，而这个类从概念上说就是个空类。

### 21.1.2 Members as Base Classes

上面的EBCO规则，并不适用于数据成员，原因是对于指向成员的指针不太好表示。

```c++
template<typename T1,typename T2>
class Myclass
{
    private:
    	T1 a;
    	T2 b;
    ...
};
```

某些时候T1和T2可能会是被空类替换，这时候的MyClass可能就会浪费一些空间。假如我们使用私有继承

```c++
template<typename T1,typename T2>
class Myclass:private T1,private T2
{};
```

如果T1，T2被空类替换，由于EBCO，并不会给T1，T2分配空间，因此节省了一些空间。然而这种方式也有一些问题

- 如果T1，T2被非类类型或者union类型替换就会出错。
- 如果T1，T2被同样的一个类型替换，也会出错。
- 如果一个类是final的，由于不能被继承，也就不能用了替换模板参数。

实际上，即便这些不是问题，增加一个基类会从根本上改变一个类的接口，这才是一个更加严重的问题。

至于说把一个可能会是空类的模板参数与其他成员组合起来，比方说

```c++
template<typename CustomClass>
class Optimizable
{
    private:
    	CustomClass info;
    	void * storage;
    ...
};
```

改成

```c++
template<typename Base,typename Member>
class BaseMemberPair:private Base
{
	private:
		Member mem;
	public:
		BaseMemberPair(Base const& b,Member const& m):Base(b),mem(m){}
		
		Base const& base() const
		{
			return static_cast<Base const&>(*this);
		}
		Base& base()
		{
			return static_cast<Base&>(*this);
		}
		Member const& member() const
		{
			return this->mem;
		}
		Member& member()
		{
			return this->mem;
		}
};
template<typename CustomClass>
class Optimizable
{
    private:
    	BaseMemberPair<CustomClass,void*> info_and_storage;
    ...
};
```

然而，实践表明这种方式只是增加了复杂性。

## 21.2 The Curiously Recurring Template Pattern(CRTP)

所谓CRTP，指的是把派生类作为其基类的模板参数。比方说

```c++
template<typename Derived>
class CuriousBase
{
    ...
};

class Curious:public CuriousBase<Curious>
{
	...
};
```

不过更加能体现这种模式本质的用法是

```c++
template<typename Derived>
class CuriousBase
{
    ...
};

template<typename T>
class CuriousTemplate:public CuriousBase<CuriousTemplate<T>>
{
	...
};
```

通过把派生类作为模板参数传递给其基类，基类就可以为派生类定制自身的行为，而不需要通过虚函数。CRTP特别适用于能够归纳出共性的操作，比如像构造函数、析构函数和下标操作这样的成员函数，或者依赖于派生类自身的操作。

其中一个简单的应用是追踪一个类类型创建了多少个对象。这可以通过在构造函数中递增一个整型静态数据成员并在析构函数中递减这个成员来实现。不过，如果在每一个类中都编写这样的代码会很多余，而且，如果使用一个单独的非CRTP基类，在其派生类中计数就可能造成混乱。所以，我们通常这样做

```c++
template<typename CountedType>
class ObjectCounter
{
    private:
        inline static std::size_t count=0;
    protected:
        ObjectCounter()
        {
            ++count;
        }
        ObjectCounter(ObjectCounter<CountedType>&&)
        {
            ++count;
        }
        ~ObjectCounter()
        {
            --count;
        }
    public:
        static std::size_t live()
        {
            return count;
        }
};

template<typename CharT>
class MyString:public ObjectCounter<MyString<CharT>>
{
	...
};

int main()
{
    MyString<char> s1,s2;
    MyString<wchar_t> ws;
    std::cout<<"num of MyString<char>: "<<MyString<char>::live()<<std::endl;
    std::cout<<"num of MyString<wchar_t>: "<<MyString<wchar_t>::live()<<std::endl;
}
```

结果是

```bash
num of MyString<char>: 2
num of MyString<wchar_t>: 1
```

这里两个MyString各自继承的基类是不同，这一点必须要明确。另外，count在MyString中是不可见的，但是调用基类中的`live()`可以访问，实际上这个live函数可以不是static的而是继承来的，当然调用方式也要相应更改，并且还可以体会不同对象比方说这里的s1、s2对基类的静态变量count的共享。

### 21.2.1 The Barton-Nackman Trick

假定我们需要为一个类模板Array定义一个`==`运算，这个运算并不适合定义为成员函数，因为成员函数的第一个参数是与this指针绑定的，其转换规则可能与第二个参数不同，另外，`==`操作通常被认为是对称的，所以，更好的方法是定义成名字空间作用域的函数。

```c++
template<typename T>
class Array
{
 	...
};

template<typename T>
bool operator==(Array<T> const& a,Array<T> const& b)
{
    ...
}
```

而所谓Barton-Nackman戏法指的是在类定义中定义友元函数。

```c++
class S{};
template<typename T>
class Wrapper
{
    private:
    	T object;
    public:
    	Wrapper(T obj):object(obj){}
    	friend void foo(Wrapper<T> const&){}
};
int main()
{
    S s;
    Wrapper<S> w(s);
    foo(w);  //Ok
    foo(s);  //编译提示 error: use of undeclared identifier 'foo'
}
```

这里的问题是`foo(s)`为什么会出这样的错误，原因当然很简单，对于s来说，`void foo(Wrapper<T> const&)`与其是不相关的，编译器并不会去Wrapper的定义中寻找这个函数的定义。这就是ADL的特性之一。

我们还可以看看关于友元函数的一个性质。如果我们在S中增加这么一个友元函数`friend void foo(Wrapper<S> const&){}`，那么编译时会提示foo重复定义的错误。换句话说，一个在类定义中定义的友元函数实际上其作用域是名字空间的。在类定义中定义一个友元函数的作用主要是，第一在类定义中定义的函数缺省是inline的，第二，在类定义中定义的友元函数，其函数体中对类的成员以及模板参数的引用可以简化。

### 21.2.2 Operator Implementations

当我们实现一个提供运算符重载的类时，通常会提供一系列相关的不同运算符的重载，比方说，如果有`==`，通常也会提供`!=`，如果有`<`，通常也会提供`>`、`<=`、`>=`，并且这些运算符可以简单的通过一个运算符来定义。当然，这应该是常识。

标准库在`<utility>`中提供了一系列使用模板的关系运算，但是这些运算是在`std::rel_ops`名字空间中定义的。不在std中定义的原因是可能会影响正常的关系运算。

我们可以通过CRTP来实现模板关系运算操作

```c++
template<typename Derived>
class EqualityComparable
{
    public:
    	friend bool operator!=(Derived const& x1,Derived const x2)
        {
            return !(x1==x2);
        }
};
class X:public EqualityComparable<X>
{
	public:
		friend bool operator==(X const& x1,X const& x2)
		{
			//比较并返回结果
		}
};

int main()
{
    X x1,x2;
    if(x1!=x2){}
}
```

这里混合使用了CRTP和Barton-Nackman戏法。注意这里是在派生类中定义的`==`，这是符合逻辑的，因为`!=`的定义依赖于`==`，而只有派生类才能具体的定义其关系运算。

如前面所说，CRTP特别适用于把派生类的操作提取到基类中，同时又保持了派生类的特征，与Barton-Nackman戏法配合可以基于一些标准运算符来定义一系列通用运算符。这是C++模板库作者惯用的技术。

### 21.2.3 Facades

所谓外观模式，指的是按照CRTP派生类所暴露的较小的但是更容易实现的接口，由CRTP基类定义一个类的大多数或者全部公共接口。传统上外观模式就是一个定义了很多操作的外观类，这个外观类所定义的操作由许多类来实现。

```c++
template<typename Derived,typename Value,typename Category,
		 typename Reference=Value&,typename Distance=std::ptrdiff_t>
class IteratorFacade
{
    public:
    	using value_type=typename std::remove_const<Value>::type;
    	using reference=Reference;
    	using pointer=Value*;
    	using difference_type=Distance;
    	using iterator_category=Category;
    	
    	//input iterator interface
    	reference operator*() const {...}
    	pointer   operator->() const {...}
    	Derived&  operator++() {...}
    	Derived   operator++(int) {...}
    	friend	bool operator==(IteratorFacade const& lhs,IteratorFacade const& rhs){...}
    	
    	//bidirectional iterator interface
    	Derived& operator--() {...}
    	Derived  operator--(int) {...}
    	
    	//random access iterator interface
    	reference operator[](difference_type n) const {...}
    	Derived&  operator+=(difference_type n) {...}
    
    	friend difference_type operator-(IteratorFacade const& lhs,IteratorFacade const& rhs)
        {...}
    	friend bool operator<(IteratorFacade const& lhs,IteratorFacade const& rhs) {...}   
};
```

这个接口操作很多，但事实上这些操作可以通过一些核心操作来实现，比如

- 对所有的迭代器:
    - `dereference()`：解引用，通常使用`*`和`->`
    - `increment()`：移动到序列中的下一项
    - `equals()`：判等
- 对双向迭代器
    - `decrement()`：移动到序列中的前一项
- 对随机访问迭代器
    - `advance()`：前向或者后向移动n步
    - `measureDistance()`：决定一个序列中两个迭代器之间的距离，或者说有几个元素

因此实现IteratorFacade主要就是把基类的操作映射到在派生类中实现的核心操作上。比如

```c++
//IteratorFacade的成员函数，获取CRTP派生类
Derived& asDerived() 
{ 
	return *static_cast<Derived*>(this); 
}
Derived const& asDerived() const
{
	return *static_cast<Derived const*>(this); 
}

reference operator*() const
{
    return asDerived().dereference();
}
Derived& operator++()
{
    asDerived().increment();
    return asDerived();
}
Derived operator++(int)
{
    Derived result(asDerived());
    asDerived().increment();
    return result;
}
friend bool operator==(IteratorFacade const& lhs,IteratorFacade const& rhs)
{
    return lhs.asDerived().equals(rhs.asDerived());
}
friend bool operator!=(IteratorFacade const& lhs,IteratorFacade const& rhs)
{
    return !(lhs==rhs);  //后面adapters程序必须的
}
```

#### Defining a Linked-List Iterator

作为一个派生类，必然会与具体的数据类型有比较紧密的耦合，表现在其所定义的核心操作是与具体的数据类型紧密结合的。而基类的作用则是屏蔽这种耦合。

```c++
template<typename T>
class ListNode
{
    public:
    	T value;
    	ListNode<T>* next=nullptr;
    	~ListNode() {delete next;} //示例，一个程序员应该知道这里在做什么。
};

template<typename T>
class ListNodeIterator:public IteratorFacade<ListNodeIterator<T>,T,std::forward_iterator_tag>
{
		ListNode<T>* current=nullptr;
	public:
		T& dereference() const
		{
			return current->value;
		}
		void increment()
		{
			current=current->next;
		}
		bool equals(ListNodeIterator const& other) const
		{
			return current==other.current;
		}
		
		ListNodeIterator(ListNode<T>* current=nullptr):current(current){}
};
```

#### Hiding the interface

上面的ListNodeIterator的一个不足之处是需要把`dereferenc()`，`advace()`，`equals()`作为公共接口暴露出来。因此我们增加一个类，使得IteratorFacade必须通过这个类来操作CRTP派生类。

```c++
class IteratorFacadeAccess
{
    template<typename Derived,typename Value,typename Category,
    		 typename Reference,typename Distance>
    friend class IteratorFacade;
    
    template<typename Reference,typename Iterator>
    static Reference dereference(Iterator const& i)
    {
        return i.dereference();
    }
    
    template<typename Iterator>
    static void decrement(Iterator& i)
    {
        return i.decrement();
    }
    
    template<typename Iterator,typename Distance>
    static void advance(Iterator& i,Distance n)
    {
        return i.advance(n);
    }
};
```

所有这些方法都是静态并且是私有的。我们可以在ListNodeIterator的定义中声明这个类为友元类`friend class IteratorFacadeAccess;`。

#### Iterator Adapters

使用上面的IteratorFacade可以很容易实现一个迭代器适配器，意思就是针对一个已经存在的迭代器，通过一个新的迭代器来转换并展示底层依赖序列的视图。

```c++
struct Person
{
    std::string firstName;
    std::string lastName;
    
    friend std::ostream& operator<<(std::ostream& strm,Person const& p)
    {
        return strm<<p.lastName<<","<<p.firstName;
    }
};

template<typename Iterator,typename T>
class ProjectionIterator:public IteratorFacade<
    		ProjectionIterator<Iterator,T>,T,
			typename std::iterator_traits<Iterator>::iterator_category,T&,
			typename std::iterator_traits<Iterator>::difference_type>
{
	using Base=typename std::iterator_traits<Iterator>::value_type;
	using Distance=typename std::iterator_traits<Iterator>::difference_type;
	
	Iterator iter;
	T Base::* member;
	
	friend class IteratorFacadeAccess;
public: 	//为简单计，把public移到这里来了，IteratorFacadeAccess实际上也就不需要了
	T& dereference() const
	{
		return (*iter).*member;
	}
	
	void increment()
	{
		++iter;
	}
	
	bool equals(ProjectionIterator const& other) const
	{
		return iter==other.iter;
	}
	
	void decrement()
	{
		--iter;
	}

	ProjectionIterator(Iterator iter,T Base::* member):iter(iter),member(member){}
	
};

template<typename Iterator,typename Base,typename T>
auto project(Iterator iter,T Base::* member)
{
    return ProjectionIterator<Iterator,T>(iter,member);
}

int main()
{
    std::vector<Person> authors={ {"David","Vandevoorde"},
                                  {"Nicolai","Josuttis"},
                                  {"Douglas","Gregor"}
                                };
    
    std::copy(project(authors.begin(),&Person::firstName),
              project(authors.end(),&Person::firstName),
              std::ostream_iterator<std::string>(std::cout,"\n")
             );
}
```

我们先忽略掉IteratorFacadeAccess的问题，看看这个程序。

project函数就是一个adapter，如名字所言，把一个迭代器投射到一个类成员类型上去，并将这个投射作为一个对象返回。

`std::copy(first, last, result))`是把从first开始到last的数据复制到result上去，当然这里都是迭代器，要注意什么这里也不必多说。这个算法执行过程中，会自动前移迭代器，所以ProjectionIterator和IteratorFacade都要实现相应的方法，如`operator++`和`increment()`。算法的终止条件是`first==last`，所以也需要各自实现`equals()`和`operator==`及`operator!=`。

除此之外，本身也没什么特别要说的。接下来我们总结下，顺便也看看这段程序里如何使用IteratorFacadeAccess。

#### 总结以及IteratorFacadeAccess

外观模式结合CRTP和Barton-Nackman戏法，提供一个完备并且正确的公共接口集，这个接口的实现只依赖于一个较小的核心操作集。

以IteratorFacade为例，这是一个基类，作为一个公共接口，其中定义了一系列的成员函数和友元函数。使用友元函数的原因是因为一些操作只适合于使用友元函数，这主要指关系运算。而把友元函数的定义放在类定义的中便是所谓的Barton-Nackman戏法，其目的如前述。这些函数的实现依赖于其派生类提供的核心操作，派生类作为模板参数传入，为了能够在基类中引用这个参数，定义了`asDerived()`，从定义可知，这里做了一个转换，根据是派生类与基类是个**is a**的关系。有了派生类的引用，就可以通过这个引用来实现基类的接口，于是可以看到，基类屏蔽了派生类的操作的细节，而我们在使用派生类实例对象时，使用的也是基类提供的接口，而不是自己本身的一个核心操作集。或者说，派生类是个与具体数据类型紧耦合的类，核心操作是与具体数据类型紧密相关的，通过CRTP基类，我们屏蔽了这种紧耦合的关系，基类便是一个公开的接口，通过传入的派生类实现这些接口。于是，一个派生类通过继承成为这个接口的实现本身，同时把自身传入基类成为这个接口实现的基础。可以与虚函数比较下，对于虚函数而言，基类只是个接口，通常不实现其中的方法，具体的方法由派生类实现。而CRTP则把这个实现分成了两部分，派生类实现与数据类型直接相关的部分，基类提供一个公共的接口，派生类通过继承把这两部分合二为一。

接下来我们看使用IteratorFacadeAccess的完整的程序，不过为了表示一些区别，这回我们要获取lastName

```c++
#include <vector>
#include <algorithm>
#include <iterator>
#include <iostream>

class IteratorFacadeAccess
{   
    template<typename Derived,typename Value,typename Category,
             typename Reference,typename Distance>
    friend class IteratorFacade;
    
    template<typename Reference,typename Iterator>
    static Reference dereference(Iterator const& i)
    {   
        return i.dereference();
    }
    
    template<typename Iterator>
    static bool equals(Iterator& il,Iterator& ir)
    {   
        return il.equals(ir);
    }
    
    template<typename Iterator>
    static void decrement(Iterator& i)
    {   
        return i.decrement();
    }
    
    template<typename Iterator>
    static void increment(Iterator& i)
    {   
        return i.increment();
    }
    
    template<typename Iterator,typename Distance>
    static void advance(Iterator& i,Distance n)
    {   
        return i.advance(n);
    }
};

template<typename Derived,typename Value,typename Category,
         typename Reference=Value&,typename Distance=std::ptrdiff_t>
class IteratorFacade
{
    public:
        using value_type=typename std::remove_const<Value>::type;
        using reference=Reference;
        using pointer=Value*;
        using difference_type=Distance;
        using iterator_category=Category;
      

        Derived& asDerived()
        {
            return *static_cast<Derived*>(this);
        }
        Derived const& asDerived() const
        {
            return *static_cast<Derived const*>(this);
        }
    
		//input iterator interface
        reference operator*() const
        {
          	return IteratorFacadeAccess::dereference<Reference>(asDerived());
        }
        Derived& operator++()
        {
            IteratorFacadeAccess::increment(asDerived());
          	return asDerived();
        }
        Derived operator++(int)
        {
            Derived result(asDerived());
            IteratorFacadeAccess::increment(asDerived());
         	return result;
        }
        friend bool operator==(IteratorFacade const& lhs,IteratorFacade const& rhs)
        {
            return IteratorFacadeAccess::equals(lhs.asDerived(),rhs.asDerived());
        }
        friend bool operator!=(IteratorFacade const& lhs,IteratorFacade const& rhs)
        {
            return !(lhs==rhs);
        }
	
    	//bidirectional iterator interface
        Derived& operator--() {...}
        Derived  operator--(int) {...}
        
        //random access iterator interface
        reference operator[](difference_type n) const {...}
        Derived&  operator+=(difference_type n) {...}

        friend difference_type operator-(IteratorFacade const& lhs,IteratorFacade const& rhs)
        {...}
        friend bool operator<(IteratorFacade const& lhs,IteratorFacade const& rhs) {...}   
};

struct Person
{   
    std::string firstName;
    std::string lastName;
   
    //就本例而言，这个函数并没有用到 
    friend std::ostream& operator<<(std::ostream& strm,Person const& p)
    {   
        return strm<<p.lastName<<","<<p.firstName;
    }
};

template<typename Iterator,typename T>
class ProjectionIterator
:public IteratorFacade<ProjectionIterator<Iterator,T>,
			T,
            typename std::iterator_traits<Iterator>::iterator_category,
			T&,
            typename std::iterator_traits<Iterator>::difference_type>
{
    using Base=typename std::iterator_traits<Iterator>::value_type;
    using Distance=typename std::iterator_traits<Iterator>::difference_type;

    Iterator iter;
    T Base::* member;

    friend class IteratorFacadeAccess;

    T& dereference() const
    {
        return (*iter).*member;
    }
    void increment()
    {
        ++iter;
    }
    bool equals(ProjectionIterator const& other) const
    {
        return iter==other.iter;
    }

    void decrement()
    {
        --iter;
    }
public:
    ProjectionIterator(Iterator iter,T Base::* member):iter(iter),member(member){}

};

template<typename Iterator,typename Base,typename T>
auto project(Iterator iter,T Base::* member)
{
    return ProjectionIterator<Iterator,T>(iter,member);
}

int main()
{
    std::vector<Person> authors={ {"David","Vandevoorde"},
                                  {"Nicolai","Josuttis"},
                                  {"Douglas","Gregor"},
                                };

    std::copy(project(authors.begin(),&Person::lastName),
              project(authors.end(),&Person::lastName),
              std::ostream_iterator<std::string>(std::cout,"\n")
             );
}
```

## 21.3 Mixins

```c++
class Point
{
    public:
    	double x,y;
    	Point():x(0.0),y(0.0) {}
    	Point(double x,double y):x(x),y(y) {}
};

template<typename P>
class Polygon
{
    private:
    	std::vector<P> points;
    public:
    	...
};

class LabeledPoint:public Point
{
	public:
		std::string label;
		LabelPoint():Point(),label("") {}
		LabelPoint(double x,double y):Point(x,y),label("") {}
};
```

这是一种比较传统的方法，这种方法的不足之处在于，Point必须暴露给使用者，以便使用者可以继承它，同时，比方说LabelPoint的作者，必须提供与Point相同的接口，如继承或者提供与Point相同的构造函数，否则就不能用于Polygon，因此如果Point做了更改，其所有的派生类都要做相应的改动。

于是就有了混入，先看例子

```c++
template<typename... Mixins>
class Point:public Mixins...
{
    public:
    	double x,y;
    	Point():Mixins()...,x(0.0),y(0.0) {}
    	Point(double x,double y):Mixins()...,x(x),y(y) {}
};

class Lable
{
    public:
    	std::string label;
    	Label():label("") {}
};

class Color
{
    public:
    	unsigned char red=0,green=0,blue=0;
};
using LabeledPoint=Point<Label>;
using MyPoint=Point<Label,Color>;

template<typename... Mixins>
class Polygon
{
    private:
    	std::vector<Point<Mixins...>> points;
    public:
    	...
};
```

Label和Color作为Point的基类混入了Point之中，而不是另外派生两个类，换句话说就是混入颠倒了通常的继承关系，新加入的属性作为基类被继承而不是通过继承来增加属性。应该说这是一种看起来是多重继承但本质上更像组合的方法。

混入主要用在对一个模板做少许定制化的场合，比方说使用用户指定的数据来修饰内部存储的对象，这样就不需要暴露库的内部数据类型和接口，从而也不需要为这些内容编写文档。比方说在上面这个Polygon模板中，甚至都不需要暴露Point这个类。如果Point发生了变化，LabelPoint和MyPoint都自然随之变化，并不需要另外去修改。

### 21.3.1 Curious Mixins

```C++
template<template<typename>... Mixins>
class Point:public Mixins<Point>...
{
    public:
    	double x,y;
    	Point():Mixins<Point>()...,x(0.0),y(0.0) {}
    	Point(double x,double y):Mixins<Point>()...,x(x),y(y) {} 
};
```

对于这种模式，需要仔细考虑混入类也就是基类的设计，首先必须是类模板，其次，混入类可以依据被混入类，也就是派生类，来裁剪其操作，比方说前面讨论过的ObjectCounter可以作为Point的一个基类，用来统计由Polygon创建的Point实例的数量。这里什么意思呢，可以这么理解，就是一个类把自身作为参数传递给一个混入类，把一些操作**委托**给这个混入类，然后通过继承这个混入类，使得这些被委托的操作成为自己的操作，而被委托的操作的实现如前所述是依赖于委托类自身的核心操作的。

### 21.3.2 Parameterized Virtuality

```c++
class NotVirtual
{
};

class Virtual
{
    public:
    	virtual void foo(){}
};

template<typename... Mixins>
class Base:public Mixins...
{
    public:
    	void foo()
        {
            std::cout<<"Base::foo()"<<std::endl;
        }
};

template<typename... Mixins>
class Derived:public Base<Mixins...>
{
	public:
		void foo()
		{
			std::cout<<"Derived::foo()"<<std::endl;
		}
};

int main()
{
    Base<NotVirtual> * p1=new Derived<NotVirtual>;
    p1->foo();  //Base::foo()
    
    Base<Virtual> * p2=new Derived<Virtual>;
    p2->foo(); //Derived::foo()
}
```

我们看看这里做了什么。首先，分别定义了两个类，一个是具体类或者说实体类NotVirtual，一个是抽象类Virtual。然后定义了一个类模板Base，其中定义了一个函数，这个函数的签名除了没有virtual关键字外与Virtual中的同名函数一致。最后是一个派生类模板Derived。光看这两个类模板的定义并不能看出什么来，而传入不同的模板参数，实例化出来完全不同的继承树，一个是普通的实体类层次，一个是抽象类层次。或者说，Base是个开关，通过类模板参数决定接下来的继承树的分支。这种方法，在设计的时候就要考虑比方说同样的一个方法，要能够适应虚函数或者非虚函数的情况，这就是所谓的引入无谓的复杂性，所以，在实践中更多的还是把不同性质的继承树分开而不是凑在一起。

## 21.4 Named Template Arguments

对于这样的一个模板

```c++
template<typename Policy1=DefaultPolicy1,
		 typename Policy2=DefaultPolicy2,
		 typename Policy3=DefaultPolicy3,
		 typename Policy4=DefaultPolicy4
		 >
class BreadSlicer
{
    ...
};
```

我们可能希望像这样`BreadSlicer<Policy3=Custom>`引用，而不是必须`BreadSlicer<DefaultPolicy1,DefaultPolcy2,Custom>`。这种所谓的关键字参数已经在很多现代语言中使用了，但是在17.4章我们已经看到对于函数参数有过提案但是被否决了，因为这样一来，函数参数的名字就会变成接口的一部分，似乎比较麻烦，于是标准委员会就拒绝了。不过对于类模板，我们可以采用变通的方法

```c++
template<typename Base,int D>
class Discriminator:public Base 
{
};

template<typename Setter1,typename Setter2,
		 typename Setter3,typename Setter4>
class PolicySelector: public Discriminator<Setter1,1>,
					  public Discriminator<Setter1,1>,
					  public Discriminator<Setter1,1>,
					  public Discriminator<Setter1,1>
{
};

//这里的几个DefaultPolicy是预先定义好的类型
class DefaultPolicies
{
  using P1=DefaultPolicy1;
  using P2=DefaultPolicy2;
  using P3=DefaultPolicy3;
  using P4=DefaultPolicy4;
};

//注意这里的虚继承
class DefaultPolicyArgs:virtual public DefaultPolicies
{
};

template<typename Policy>
class Policy1_is:virtual public DefaultPolicies
{
	public:
		using P1=Policy;
};

template<typename Policy>
class Policy2_is:virtual public DefaultPolicies
{
	public:
		using P2=Policy;
};

template<typename Policy>
class Policy3_is:virtual public DefaultPolicies
{
	public:
		using P3=Policy;
};

template<typename Policy>
class Policy4_is:virtual public DefaultPolicies
{
	public:
		using P4=Policy;
};

template<typename PolicySetter1=DefaultPolicyArgs,
		 typename PolicySetter2=DefaultPolicyArgs,
		 typename PolicySetter3=DefaultPolicyArgs,
		 typename PolicySetter4=DefaultPolicyArgs
		 >
class BreadSlicer
{
    using Policies=PolicySelector<PolicySetter1,PolicySetter2,PolicySetter3,PolicySetter4>;
    ...
};
```

如果我们进行这样一个实例化`BreadSlicer<Policy3_is<CustomPolicy>>> bc`，那么Policies的定义就是`PolicySelector<Policy3<CustomPolcy>,DefaultPolicyArgs,DefaultPolicyArgs,DefaultPolicyArgs>`。通常按道理来说，一个类不能有多个同样的基类，比方说这里的DefaultPolicyArgs，而由于Discriminator，使得PolicySelector的基类变得不同了。但是这些基类又有一个同样的虚基类DefaultPolicies，因而都继承了P1，P2，P3，P4，除了`Policy3_is<>`重新定义了P3。在BreadSlicer中可以这样使用这4个P，注意除了P3之外，其它3个P都是DefaultPolicyArgs

```c++
template<...>
class BreadSlicer
{
    ...
    public:
    	void print()
        {
            Policies::P3::doPrint();
        }
    ...
};
```

这里需要理解的一个问题是，通过虚继承使得只有一个基类DefaultPolicies存在，而Policy3_is重新定义了P3，那么为什么不会引发冲突？这就涉及到“domination rule”，也就是说由于这条规则，重新定义的P3覆盖了基类的P3。另外，上面的doPrint显然是个静态方法，如果需要调用非静态方法，那么还是需要创建一个P3的实例的。

这里使用的4个参数还是很容易扩展的，只不过为了达到使用上的便利性，增加了不少开发的复杂性，如果不是开发库的话，没有需要这样做的理由。









