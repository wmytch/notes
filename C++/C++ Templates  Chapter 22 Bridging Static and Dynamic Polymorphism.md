# Chapter 22 Bridging Static and Dynamic Polymorphism

[TOC]

静态多态有与非多态代码同样的性能，但在运行时可以使用的类型在编译时就决定了。

动态多态则可以用一个多态函数来处理编译时尚不可知的类型，但是因为必须从一个公共的基类继承，**所以其灵活性要低一些**。着重我自己加的，因为这个说法跟以前对灵活性的理解可谓大相径庭。

## 22.1 Function Objects,Pointers,and `std::function<>`

函数对象可以用来给模板提供定制行为。

```c++
template<typename F>
void forUpTo(int n,F f)
{
    for(int i=0;i!=n;++i)
    {
        f(i);
    }
}

void printInt(int i)
{
    std::cout<<i<<' ';
}

int main()
{
    std::vector<int> values;
    
    forUpTo(5,[&values](int i) { values.push_back(i); });
    forUpTo(5,pintInt);
}
```

forUpTo函数模板可以接受各种函数对象，包括lambda表达式，函数指针，实现了`operator()`或者可以转换成函数指针/引用的类，因此，有可能在实例化的时候造成代码膨胀，如果forUpTo的代码比较多，那么这个问题会是比较严重的。

为了避免这种代码膨胀，可以改成非模板形式

```c++
void forUpTo(int n,void (*f)(int))
{
    ....
}
```

然而这种形式不支持lambda表达式。标准库提供了一个类模板`std::function<>`，可以用来解决上面的问题

```c++
void forUpTo(int n,std::function<void(int)> f)
{
    ...
}
```

forUpTo展示了静态多态的一些特性，能够契合各种类型，包括函数指针，lambda表达式，以及具有`operator()`的类，而其自身却可以保持非模板的形式。这种技术称为类型擦除，可以桥接静态和动态的多态。

## 22.2 Generalized Function Pointers

`std::function<>`是个强有力的C++函数指针的通用形式，提供了函数指针同样的基本操作

- 可以用来调用一个函数，而不需要知道这个函数的任何信息
- 可以复制，移动以及赋值
- 可以从别的函数初始化或者赋值，只要签名是兼容的
- 有一个`null`状态来表示未绑定函数

不过与C++函数指针不同的是，`std::function<>`还可以保存lambda表达式，或者任意类型的具有适当的`operator()`的函数对象，所有这些其类型都是不同的。

我们看看这个通用函数指针的大概实现。出于其目的，主要工作是两个方面，绑定一个函数对象，然后调用这个函数对象。

```c++
template<typename Signature>
class FunctionPtr;

template<typename R,typename... Args>
class FunctionPtr<R(Args...)>
{
    private:
    	FunctorBridge<R,Args...>* bridge;
    public:
    	FunctionPtr():bridge(nullptr) {}
    	//复制构造函数，类外定义
    	FunctionPtr(FunctionPtr const& other);
    	//在初始化列表中调用上一个构造函数来初始化
    	FunctionPtr(FunctionPtr &other)
            :FunctionPtr(static_cast<FunctionPtr const&>(other)) 
        {
        }
        //非模板移动构造函数
        FunctionPtr(FunctionPtr&& other):bridge(other.bridge)
        {
            other.bridge=nullptr;
        }
        //类外定义，移动构造函数模板
    	template<typename F>
    	FunctionPtr(F&& f);
    	//赋值运算符
    	FunctionPtr& operator=(FunctionPtr const& other)
        {
            FunctionPtr tmp(other);
            swap(*this,tmp);
            return *this;
        }
        //移动语义赋值运算符
    	FunctionPtr& operator=(FunctionPtr&& other)
        {
            delete bridge;
            bridge=other.bridge;
            other.bridge=nullptr;
            return *this;
        }
        //赋值运算符模板
   		template<typename F>
    	FunctionPtr& operator=(F&& f)
    	{
            FunctionPtr tmp(std::forward<F>(f));
            swap(*this,tmp);
            return *this;
        }
    	~FunctionPtr()
        {
            delete bridge;
        }
   		friend void swap(FunctionPtr& fp1,FunctionPtr& fp2)
        {
            std::swap(fp1.bridge,fp2.bridge);
        }
    	explicit operator bool() const
        {
            return bridge==nullptr;
        }
        //类外定义
    	R operator()(Args... args) const;
};
```
要注意`FunctionPtr<R(Args…)>`的句法，R是函数对象的返回值类型，Args是函数对象的调用参数类型。这里也可以看到，针对模板有不少特别的句法，这些句法是不能用到通常的表达式和控制结构中的。

声明了一个主模板，因为只是使用其特化版本，所以只声明未定义，但是这个声明也是必不可少的。特化版本中定义了缺省构造函数，复制构造函数，移动构造函数，复制运算符，显式转换函数，函数运算符，以及几个相关的函数模板。

因为涉及到其它类模板的定义，所以有三个成员函数在类外定义。不过，模板的定义通常来说都会放在头文件当中，所以这些类的定义怎么组织，不过是运用之妙存乎一心的事情。

```c++
template<typename R,typename... Args>
template<typename F>
FunctionPtr<R(Args...)>::FunctionPtr(F&& f)
    :bridge(nullptr)
{
	using Functor=std::decay_t<F>;
    using Bridge=SpecificFunctorBridge<Functor,R,Args...>;
    bridge=new Bridge(std::forward<F>(f));
};

template<typename R,typename... Args>
FunctionPtr<R(Args...)>::FunctionPtr(FunctionPtr const& other):bridge(nullptr)
{
    if(other.bridge)
    {
        bridge=other.bridge->clone();
    }
}
template<typename R,typename... Args>
R FunctionPtr<R(Args...)>::operator()(Args... args) const
{
    return bridge->invoke(std::forward<Args>(args)...);
}
```
`bridge(nullptr)`的目的是，如果在构造过程中抛出了异常，确保这个指针不会乱指，从而是线程安全的。

## 22.3 Bridge Interface

```c++
template<typename R,typename... Args>
class FunctionBridge
{
    public:
    	virtual ~FunctorBridge(){}
    	virtual FunctorBridge* clone() const=0;
    	virtual R invoke(Args... args) const=0;
};
```
这个类的主要作用是取得底层函数对象的所有权并对其进行处理。当然，这是一个抽象类，需要具体类来实现。需要特别注意clone和invoke两个函数都是const的，目的是假如有人无意间重载了FunctionPtr的`operator()`，并且这个重载函数是非const的，如果通过一个const的FunctionPtr对象调用这个重载函数，编译器就会提示错误。

## 22.4 Type Erasure

FunctorBridge是个抽象类，所以必须要有派生类来实现其虚函数。

```c++
template<typename Functor,typename R,typename... Args>
class SpecificFunctorBridge:public FunctorBridge<R,Args...>
{
		Functor functor;
	public:
		template<typename FunctorFwd>
		SpecificFunctorBridge(FunctorFwd&& functor):functor(std::forward<FunctorFwd>(functor))
		{}
		virtual SpecificFunctorBridge* clone() const override
		{
			return new SpecificFunctorBridge(functor);
		}
		virtual R invoke(Args... args) const override
		{
			return functor(std::forward<Args>(args));
		}
};
```

我们回头看这个构造函数

```c++
template<typename R,typename... Args>
template<typename F>
FunctionPtr<R(Args...)>::FunctionPtr(F&& f)
    :bridge(nullptr)
{
	using Functor=std::decay_t<F>;
    using Bridge=SpecificFunctorBridge<Functor,R,Args...>;
    bridge=new Bridge(std::forward<F>(f));
};
```

虽然这个构造函数本身是个模板化的函数，但是其参数F只对SpecificFunctorBridge的实例才是可知的，创建一个新的Bridge对象，也就是一个`SpecificFunctorBridge<Functor,R,Args…>`实例之后，这个F就消失了，也就是说，在之后所有对bridge的操作中，操作者都不会知道F是什么，更无从去直接操作这个F，也就是通俗说的F对使用者是透明的。Bridge的涵义就体现在这个函数当中。虽然理论上通过dynamic_cast加上其他一些手段可以查询到F的类型，但是由于bridge是私有成员，所以实际上FunctionPtr的使用者是无法进行这种操作的。

这里我们可以再次强调下，一个这样的通用函数指针，最主要的工作是两个，一个是绑定并隐藏一个函数对象，这里由上面这个构造函数及clone函数来实现，另一个就是对这个函数对象的调用，这个由invoke函数来实现。这正好是前面看到的在FunctionPtr类外定义的两个构造函数和`operator()`的工作。

另外就是`std::decay_t<F>`，意义很明确，就不必多说了，但是至关重要，只有这样F才能够保存。

最后需要再强调的是，之所以采用一个抽象类来封装函数对象，目的在于避免代码膨胀问题。如前面forUpTo的例子，使用的时候只需要实例化一个通用函数指针，然后通过这个通用函数指针中的成员虚函数来调用函数对象，而不需要每一个不同的函数对象都去实例化一个forUpTo模板函数。但要注意的是这里解决了函数模板产生的代码膨胀问题，而不是消除了实例化通用函数指针模板需要的代价，只不过这样相对而言代价要小。

## 22.5 Optional Bridging

如果需要给上面的FunctionPtr增加关系运算，毕竟有时候我们需要判断两个指针是否一致，下面这样做的话是有问题的。

```c++
template<typename R,typename... Args>
class FunctionBridge
{
   ...
   virtual bool equals(FunctorBridge const* fb) const=0;
   ...
};

template<typename Functor,typename R,typename... Args>
class SpecificFunctorBridge:public FunctorBridge<R,Args...>
{
    ...
    virtual bool equals(FunctorBridge<R,Args...> const* fb) const override
    {
        if(auto specFb=dynamic_cast<SpecificFunctiorBridge const*>(fb))
        {
            return functor== specFb->functor;
        }
        return false;
    }
    ...
};

friend bool operator==(FunctionPtr const& f1,FunctionPtr const& f2)
{
    if(!f1||!f2)
    {
        return !f1&&!f2;
    }
    return f1.bridge->equals(f2.bridge);
}
```

这里的问题在于，如果一个函数对象，比方说lambda表达式，本身没有提供合适的`==`运算，就会编译出错，而这个错误发生在FunctionPtr用这个函数对象初始化或者被赋值的时候，而不是在使用`operator==`时。具体的说就是在初始化FunctionPtr时由于类型擦除，函数对象的类型信息丢失了，而初始化FunctionPtr需要知道函数对象完全的类型信息，这又依赖于初始化一个SpecificFunctorBridge对象，然后才能完成FunctionPtr的初始化，而初始化SpecificFunctorBridge需要实现equals虚函数，这时又用到了函数对象的`operator==`，于是就出错了。

当然，我们仍然可使用SFINAE来解决这样的问题。

```c++
template<typename T>
class IsEqualityComparable
{
    private:
    	static void* conv(bool);
    	template<typename U>
    	static std::true_type test(decltype(conv(std::declval<U const&>()
                                               ==std::declval<U const&>())),
                                   decltype(conv(!(std::declval<U const&>()
                                               ==std::declval<U const&>())))
                                  );
    	template<typename U>
    	static std::false_type test(...);
    public:
    	static constexpr bool value=decltype(test<T>(nullptr,nullptr))::value;
};
```

这个类很典型，就不多说了。

```c++
template<typename T,bool EqComparable=IsEqualityComparable<T>::value>
struct TryEquals
{
  static bool equals(T const& x1,T const& x2)
  {
      return x1==x2;
  }
};
class NotEqulityComparable:public std::exception
{};
template<typename T>
struct TryEquals<T,false>
{
    static bool equals(T const& x1,T const& x2)
    {
        throw NotEqualityComparable();
    }
};

template<typename Functor,typename R,typename... Args>
bool SpecificFunctorBridge::equals(FunctorBridge<R,Args> const* fb) const override
{
    if(auto specFb=dynamic_cast<SpecificFunctiorBridge const*>(fb))
    {
        return TryEquals<Functor>::equals(this->functor,sepcFb->functor);
    }
    return false;
}
```

## 22.6 Performance Considerations

类型擦除的性能总的来说与动态多态相近，因为都使用了虚函数来实现动态派遣。是否需要使用类型擦除，还是要看相对的工作量，这与前面所说的消除代码膨胀的情况是类似的。如果forUpTo本身的工作量很大，比方说数据库查询，容器排序，更新用户界面等等，这时候类型擦除本身的工作量与之相比基本上可以认为是微不足道的。