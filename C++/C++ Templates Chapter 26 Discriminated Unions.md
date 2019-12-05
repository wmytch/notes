# Chapter 26 Discriminated Unions

[TOC]

所谓union，本质上指的是对一片存储空间不同的解释。而Discriminated Union指的是一个union变量知道自己是什么类型，或者说怎么去解释这片存储空间，从而提供了相对于c/c++的union更多的类型安全。

对于像`Variant<int,double,string> field`这样的变量，通常称为封闭的可识别union，声明之后就不能再往类型集合中添加新的类型了。而开放的可识别union指的则是在变量创建之后还可以往其中添加类型，比方说22.2讨论的FunctionPtr。

## 26.1 Storage

我们可以使用Tuple来实现一个可识别的union

```c++
template<typename... Types>
class Variant
{
    public:
    	Tuple<Types...> storage;
    	unsigned char discriminator;
};
```

这里的`discriminator`的作用是指示元组中的哪个类型是有效的，比方说如果discriminator是0，那么`get<0>(storage)`才会返回一个有效的值，如果discriminator是1，那么就需要`get<1>(storage)`。于是这种实现的问题是，需要占用的空间是其所有元素类型各自所占空间的总和，显然这跟union的本意是冲突。另外还有一些其他问题，比方说需要其元素类型具有缺省构造函数。

既然是union，就应该用union来定义

```c++
template<typename... Types>
union VariantStorage;
template<typename Head,typename... Tail>
union VariantStorage
{
    Head head;
    VariantStorage<Tail...> tail;
};
template<>
union VariantStorage<>
{};
```

虽然这样解决了存储空间的问题，但是就其操作而言，由于union无法继承，这种实现使用起来会非常困难。

所以，还是回到class

```c++
template<typename... Types>
class VariantStorage
{
    	using LargestT=LargestType<Typelist<Types...>>;
    	alignas(Types...) unsigned char buffer[sizeof(LargestT)];
    	unsigned char discriminator=0;
    public:
    	unsigned char getDiscriminator() const {return discriminator;}
    	void setDiscriminator(unsigned char d) {discriminator=d;}
    	void* getRawBuffer() {return buffer;}
    	const void* getRawBuffer() const {return buffer;}
    
    	template<typename T>
    	T* getBufferAs() 
    	{
            return std::launder(reinterpret_cast<T*>(buffer));
        }
    	template<typename T>
    	T const* getBufferAs() const
    	{	
        	return std::launder(reinterpret_cast<T const*>(buffer));
        }
};
```

这里需要解释的是

- `LargestType<Typelist<Types…>>`可以参见24.2.2
- `alignas(pack…)`对参数包中的每一个成员应用alignas，然后选取出一个最大或者说最严格的的对齐要求作为最终的对齐要求，以确保buffer的对齐能够适用于所有的值类型。
- `std::launder`返回一个指针`T*`，这个指针所指向的空间被解释成`T`类型，实际上也就是说作为其参数的指针类型所指的对象必须与T是同一个类型。下面会有更详细的解释。
- c++中的四种cast就不多说了，由于这里需要转换的是指针，所以使用了reinterpret_cast。

## 26.2 Design

我们首先列出Variant的定义，之后再逐步解释里边的东西。

```c++
template<typename... Types>
class Variant:private VariantStroage<Types...>,private VariantChoice<Types,Types...>...
{
        template<typename T,typename... OtherTypes>
        friend class VariantChoice;
    public:
        template<typename T> bool is() const;
        template<typename T> T& get() &;
        template<typename T> T const& get() const&;
        template<typename T> T&& get() &&;

        template<typename R=ComputedResultType,typename Visitor>
        VisitResult<R,Visitor,Types&...> visit(Visitor&& vis) &;
        template<typename R=ComputedResultType,typename Visitor>
        VisitResult<R,Visitor,Types const&...> visit(Visitor&& vis) const&;
        template<typename R=ComputedResultType,typename Visitor>
        VisitResult<R,Visitor,Types&&...> visit(Visitor&& vis) &&;

        using VariantChoice<Types,Types...>::VariantChoice...;
        Variant();
        Variant(Variant const& source);
        Variant(Variant&& source);
        template<typename... SourceTypes>
        Variant(Variant<SourceTypes...> const& source);
        template<typename... SourceTypes>
        Variant(Variant<SourceTypes...>&& source);

        using VariantChoice<Types,Types...>::operator=...;
        Variant& operator=(Variant const& source);
        Variant& operator=(Variant&& source);
        template<typename... SourceTypes>
        Variant& operator=(Variant<SourceTypes...> const& source);
        template<typename... SourceTypes>
        Variant& operator=(Variant<SourceTypes...>&& source);

        bool empty() const;
      	~Variant(){destroy();}
        void destroy();
};
```

可以看到，Variant私有继承了两个类，一个是前面定义的VariantStroage，这个类解决了Variant的存储问题，另一个类是VariantChoice，提供了对buffer的核心操作，其定义如下，具体内容后面会有解释。

```c++
template<typename List,typename T,unsigned N=0,bool Empty=IsEmpty<List>::value>
struct FindIndexOfT;

template<typename List,typename T,unsigned N>
struct FindIndexOfT<List,T,N,false>
    :public IfThenElse<std::is_same<Front<List>,T>::value,
                       std::integral_constant<unsigned,N>,
                       FindIndexOfT<PopFront<List>,T,N+1>>
{};

template<typename List,typename T,unsigned N>
struct FindIndexOfT<List,T,N,true>
{};

template<typename T,typename... Types>
class VariantChoice
{
        using Derived=Variant<Types...>;
        Derived& getDerived(){ return *static_cast<Derived*>(this);}
        Derived const& getDerived() const {return *static_cast<Derived const*>(this);}
    protected:
        constexpr static unsigned Discriminator=FindIndexOfT<Typelist<Types...>,T>::value+1;
    public:
        VariantChoice(){}
        VariantChoice(T const& value);
        VariantChoice(T&& value);
        bool destroy();
        Derived& operator=(T const& value);
        Derived& operator=(T&& value);
};
```

模板参数包Types来自Variant的模板参数包，而T则是Variant当前或者将要使用的类型，通过元函数FindIndexOfT来获取T在Types中的位置，这个元函数很直白就不解释了。要注意的是这里使用了21.2介绍的CRTP，当然这里并没有把派生类本身作为基类的模板参数传入基类，而是传入了派生类的模板参数，然后在基类中进行类型转换，或者说在基类中组合了其派生类，毕竟其派生类是已知并且确定的。

回头看Variant，有一个唯一的VariantStroage基类和数个VariantChoice基类，其基类都是私有继承，因为这些基类不属于公共接口，而为了让VariantChoice中的getDerived能够进行类型转换，必须声明其为Variant的友元类。

最后我们看`VariantChoice<Types,Types...>…`，这是所谓的嵌套包展开，详细说明可以参见12.4.4。比方说对于`Variant<int,double,std::string>`，VariantChoice会分成两步展开，首先是针对外部包，会展开成

```c++
VariantChoice<int,Types...>,VariantChoice<double,Types...>,VariantChoice<std::string,Types...>
```

然后展开内部的参数包，从而得到

```c++
VariantChoice<int,int,double,std::string>,
VariantChoice<double,int,double,std::string>,
VariantChoice<std::string,int,double,std::string>
```

显然，一个Variant参数列表中有几个类型，就会有几个VariantChoice基类，而各自Discriminator的值分别就是1，2，3。于是当`VariantStorage::discriminator`的值与某个`VariantChoice<...>::Discriminator`的值匹配时，其所存储的对象的类型便是对应的VariantChoice所表示的类型。这里要注意的的是`VariantsStorage::discriminator`为0表示没有存储任何对象，或者说是一个空值，后面会对此专门讨论。

## 26.3 Value Query and Extraction

首先，必须重申下Discriminator Union的含义，也就是一个知道其存储的值当前的活动类型的union，下面讨论的is和get的实现就立足于并且体现了这个特性。

`is()`成员函数用来查询一个活动值是否某一个特定类型T

```c++
template<typename... Types>
template<typename T>
bool Variant<Types...>::is() const
{
    return this->getDiscriminator()==VariantChoice<T,Types...>::Discriminator;
}
```

当前VariantStroage存储的值的类型可以由discriminator指示，这里要明确一点，就是一个union可以看成是由几个类型共享一片存储空间，disriminator指示的是当前对这片空间的解释，而不是这片空间只能是这个类型。如果指定类型T在Types中的序号与discriminator不匹配，其实也没关系，并不是错误，只是个指示而已，而如果T并不存在于Types中，则会引发编译错误，这才是is函数的真正意义。

`get()`函数则是获取存储值的一个引用，当然必须指定类型，因而也必须保证当前存储的值的活动类型是指定的类型

```c++
class EmptyVariant:public std::exception{};
template<typename... Types>
template<typename T>
T& Variant<Types...>::get() &
{
    if(empty())
    {
        throw EmptyVariant();
    }
    assert(is<T>());
    return *this->template getBufferAs<T>();
}
template<typename... Types>
template<typename T>
T&& Variant<Types...>::get() &&
{
    if(empty())
    {
        throw EmptyVariant();
    }
    assert(is<T>());
    return std::move(*this->template getBufferAs<T>());
}
template<typename... Types>
template<typename T>
T const& Variant<Types...>::get() const&
{
    if(empty())
    {
        throw EmptyVariant();
    }
    assert(is<T>());
    return *this->template getBufferAs<T>();
}
```

return语句中的template就不解释了，总之不要忘记模板实例化的两步查找。

## 26.4 Element Initialization,Assignment and Destruction

这些都是VariantChoice的职责。

### 26.4.1 Initialization

```c++
template<typename T,typename... Types>
VariantChoice<T,Types...>::VariantChoice(T const& value)
{
    new(getDerived().getRawBuffer()) T(value);
    getDerived().setDiscriminator(Discriminator);
}
template<typename T,typename... Types>
VariantChoice<T,Types...>::VariantChoice(T&& value)
{
    new(getDerived().getRawBuffer()) T(std::move(value));
    getDerived().setDiscriminator(Discriminator);
}
```

这里的new是个放置式new，就不解释了，然后是设置discriminator。当然我们最终需要实现的是像`Variant<int,double,string> v(“hello”)`这样的初始化，而在Variant的定义中，使用using引入了其基类VariantChoice的构造函数，也就是上面代码里的定义。而实际上，通过这个using会生成对应于Types中每一个T的构造函数，比方说对于前面的例子，会有

```c++
Variant(int const&);
Variant(int &&);
Variant(double const&&);
Variant(double &&);
Variant(string const&);
Variant(string&&);
```

至于VariantChoice会怎么根据Variant的参数展开，前面已经解释过了，于是，对应于v(“hello”)，推断T为string，或者更准确一点，把一个`const char *`转换成了string，于是使用了移动构造函数`Variant(string&&)`，或者说`VariantChoice<string,int,float,string>::VariantChoice(string&&)`。

### 26.4.2 Destruction

```c++
template<typename T,typename... Types>
bool VariantChoice<T,Types...>::destroy()
{
    if(getDerived().getDiscriminator()==Discriminator)
    {
        getDerived().template getBufferAs<T>()->~T();
        return true;
    }
    return false;
}
```

这个函数只有当类型匹配是才会进行析构，但是我们通常在析构时并不打算关注当前活动类型是什么，所以

```c++
template<typename... Types>
void Variant<Types...>::destroy()
{
    bool results[]={VariantChoice<Types,Types...>::destroy()...};
    this->setDiscriminator(0);
}
```

这里要注意一下，对`VariantChoice<Types,Types...>::destroy()…`的展开及调用是编译期在results的初始化器中进行的，并不能够在运行期迭代进行，而results本身存在的意义就是这么一个初始化器，在c++17，使用12.4.6介绍的折叠表达式甚至可以把这个results去掉

```c++
template<typename... Types>
void Variant<Types...>::destroy()
{
    (VariantChoice<Types,Types...>::destroy(),...);
    this->setDiscriminator(0);
}
```

### 26.4.3 Assignment

```c++
template<typename T,typename... Types>
auto VariantChoice<T,Types...>::operator=(T const& value) -> Derived&
{
    if(getDerived().getDiscirminator()==Discriminator)
    {
        *getDerived().template getBufferAs<T>()=value;
    }
    else
    {
        getDerived().destroy();
        new(getDerived().getRawBuffer()) T(value);
        getDerived().setDiscriminator(Discriminator);
    }
    return getDerived();
}
template<typename T,typename... Types>
auto VariantChoice<T,Types...>::operator=(T&& value) -> Derived&
{
    if(getDerived().getDiscirminator()==Discriminator)
    {
        *getDerived().template getBufferAs<T>()=std::move(value);
    }
    else
    {
        getDerived().destroy();
        new(getDerived().getRawBuffer()) T(std::move(value));
        getDerived().setDiscriminator(Discriminator);
    }
    return getDerived();
}
```

前面已经看到Variant通过using引入了其基类的赋值运算符的定义，这里提一下，使用using的原因是VariantChoice是private继承的。

对于赋值运算符，尤其是进入两步处理分支的时候，需要考虑这么几个问题

- 自赋值
- 异常
- std::launder()

#### Self-Assignment

自赋值可能发生在像这种情况

```c++
v=v.get<T>()
```

这个时候，如果进行的是两步处理，旧值在被复制之前就被销毁了，于是可能引发内存崩溃。不过，在这里由于自赋值总是意味着类型匹配，所以不会进入到两步处理的分支中。

#### Exceptions

如果现值已经被析构，而新值在初始化时抛出异常，那么会是一个什么状态？在`Variant::destroy()`的实现里，我们把discriminator的值置为0，如果没有异常，那么在初始化完成后这个值会被正确设置，如果新值初始化的时候发生异常，discriminator仍然会保持为0，表明Variant并没有保存一个有效的值。为此定义了empty函数

```c++
template<typename... Types>
bool Variant<Types...>::empty() const
{
    return this->getDiscriminator()==0;
}
```

#### std::launder()

c++编译器通常试图生成高性能的代码，而其主要机制是避免重复的从内存复制数据。为此，编译器必须做一些假设，其中之一是某种数据在其生存期是不会变更的，其中包括const的数据和引用(可以初始化，但是此后不能更改)，以及多态对象中保存的某些数据，比方说用来派遣虚函数，定位虚基类，以及处理typeid和dynamic_cast的相关数据。

对于前面定义的两步赋值操作而言，其主要问题在于偷偷的结束了一个值的生存期，同时又在原地开始了一个新的值的生存期，而编译器可能无法辨识这种情况，于是就会假定其所需要的一个Variant对象之前的状态仍然有效，而实际上这并不是事实，这样的bug很难定位，一则很少见，二则从代码并不能看出来。

在c++17，通过`std::launder()来解决这个问题，虽然这个函数只是返回其参数本身，但是通过这个函数，可以告诉编译器不要假定这个地址上没有发生变化。具体的机制无关紧要了，这个函数也在修订当中。

## 26.5 Visitors

visit函数用来访问Variant的数据，不过需要现做一些准备

```c++
template<typename R,typename V,typename Visitor,typename Head,typename... Tail>
R variantVisitImpl(V&& variant,Visitor&& vis,Typelist<Head,Tail...>)
{
    if(variant.template is<Head>())
    {
        return static_cast<R>(std::forward<Visitor>(vis)(std::forward<V>(variant).template get<Head>())); //#1
    }
    else if constexpr(sizeof...(Tail)>0)
    {
        return variantVisitImpl<R>(std::forward<V>(variant),std::forward<Visitor>(vis),Typelist<Tail...>());
    }
    else
    {
        throw EmptyVariant();
    }
}
```

因为variant和vis都是作为右值传入，所以使用了std::forward来确保不会失效，vis是一个函数或者函数对象甚至是一个lambda，对其做了一次调用，函数参数是variant的Head，这便是#1的意义。

```c++
template<typename... Types>
template<typename R,typename Visitor>
VisitResult<R,Visitor,Types&...>
Variant<Types...>::visit(Visitor&& vis) &
{
    using Result=VisitResult<R,Visitor,Types&...>;
    return variantVisitImpl<Result>(*this,std::forward<Visitor>(vis),Typelist<Types...>());
}

template<typename... Types>
template<typename R,typename Visitor>
VisitResult<R,Visitor,Types const&...>
Variant<Types...>::visit(Visitor&& vis) const&
{
    using Result=VisitResult<R,Visitor,Types const&...>;
    return variantVisitImpl<Result>(*this,std::forward<Visitor>(vis),Typelist<Types...>());
}

template<typename... Types>
template<typename R,typename Visitor>
VisitResult<R,Visitor,Types&&...>
Variant<Types...>::visit(Visitor&& vis) &&
{
    using Result=VisitResult<R,Visitor,Types&&...>;
    return variantVisitImpl<Result>(std::move(*this),std::forward<Visitor>(vis),Typelist<Types...>());
}
```

这三个函数的区别在于Variant是以&，const&还是&&传入variantVisitImpl。

### 26.5.1 Visit Result Type

我们来看上面的`VisitResult<R,Visitor,Types&&…>`是个啥。

```c++
template<typename R,typename Visitor,typename... ElementTypes>
class VisitResultT
{
    public:
    	using Type=R;
};
template<typename R,typename Visitor,typename... ElementTypes>
using VisitResult=typename VisitResultT<R,Visitor,ElementTypes...>::Type;
```

我们可以指定这里的R的类型作为返回类型，而在Variant的定义中我们看到R的缺省值是ComputedResultType，这是一个不完全类，即只有一个声明`class ComputedResultType;`，据此我们可以定一个VisitResultT的特化，详见下一节。

###  26.5.2 Common Result Type

```c++
using std::declval;
template<typename T,typename U>
class CommonTypeT
{
    public:
    	using Type=decltype(true?declval<T>():declval<U>());
};
template<typename T,typename U>
using CommonType=typename CommonType<T,U>::Type;
```

与1.3.3中的common_type特性一样，这里利用了3元表达式`b?x:y`的一个特性，就是其返回值的类型是x和y的共同类型，比方说int和double的共同类型就是double，因为int会提升为double，当然，这也是c或者c++对表达式中不同数据类型混合运算的一个隐式处理。

```c++
template<typename Visitor,typename T>
using VisitElementResult=decltype(std::declval<Visitor>()(std::declval<T>()));

template<typename Visitor,typename... ElementTypes>
class VisitorResultT<ComputedResultT<ComputedResultType,Visitor,ElementTypes...>
{
    	using ResultTypes=Typelist<VisitElementResult<Visitor,ElementTypes>...>;
    public:
    	using Type=Accumulate<PopFront<ResultTypes>,CommonTypeT,Front<ResultTypes>>;
};
```

首先，利用VisitElementResult生成一个所有可能的Visitor返回的类型的Typelist，当然，和以前一样，这是在编译期通过对decltype和declval的分析得到的结果。然后，通过24.2.6介绍的Accumulate计算出这个Typelist中所有类型的公共类型，如果这些类型有不兼容的，则产生一个编译错误。

之前我们说过的`std::common_type()`可以用在这里，并且可以简化VisitResultT的定义，换句话说就是，这个特性合并了上面CommonTypeT和Accumulate的工作。

```c++
template<typename Visitor,typename... ElementTypes>
class VisitResult<ComputedResultType,Visitor,ElementTypes...>
{
    public:
    	using Type=std::common_type_t<VisitElementResult<Visitor,ElementTypes>...>;
};
```

对于下面的程序

```c++
int main()
{
    Variant<int,short,double,float> v(1);
    auto result=v.visit([](auto const& value)
                    {
                        return value+1;                
                    });
    std::cout<<typeid(result).name()<<'\n';
}
```

输出会是`d`，因为这是类型列表所有类型都可以转化的类型。另外要注意的是这里类型列表里的类型都是可兼容的类型，放个string进去就要出错了，当然，这是visit的参数里lambda表达式的原因。

## 26.6 Variant Initialization and Assignment

### Default Initialization

缺省构造函数是有必要的，但是其语义是什么？如果用discriminator为0来表示一个缺省构造的Variant，虽然说我觉得也没什么问题，但是从理论上说会给一个空的Variant增加额外的责任。按照c++17标准里`std::variant<>`的语义，缺省构造的结果是其参数类型列表中第一个元素类型的值，也就是说，必然存了一个值，其值的类型是这一串类型中第一个类型的缺省值。

```c++
template<typename... Types>
Variant<Types...>::Variant()
{
    *this=Front<Typelist<Types...>>();
}
```

只是要注意一下，这里调用了基类VariantChoice的`operator=`，可参见前面的内容，就不详细说了。

### Copy/Move Initialization

要复制一个源Variant，需要知道其当前保存的值的活动类型，把这个值复制构造到buffer中，然后设置discriminator。可以用`visit()`来处理源Variant的活动类型，用VariantChoice的复制构造函数来把这个值构造到buffer中。

```c++
template<typename... Types>
Variant<Types...>::Variant(Variant const& source)
{
    if(!source.empty())
    {
        source.visit([&](auto const& value)
                     {
                         *this=value;
                     }
                    );
    }
}
template<typename... Types>
Variant<Types...>::Variant(Variant && source)
{
    if(!source.empty())
    {
        std::move(source).visit([&](auto const& value)
                     {
                         *this=std::move(value);
                     }
                    );
    }
}
```

visit会迭代类型列表，直到找到一个合适的类型，然后通过VariantChoice的复制构造函数将值构造到buffer上，另外一方面，虽然`=`出现在这个lambda中，但实际的工作还是由VariantChoice做的。

基于visit的这种实现，也适用于模板形式的复制和移动构造函数

```c++
template<typename... Types>
template<typename... SourceTypes>
Variant<Types...>::Variant(Variant<SourceTypes...> const& source)
{
    if(!source.empty())
    {
        source.visit([&](auto const& value)
                     {
                    	*this=value;     
                     }
                    );
    }
}
template<typename... Types>
template<typename... SourceTypes>
Variant<Types...>::Variant(Variant<SourceTypes...>&& source)                                                                                                                                     
{   
    if(!source.empty())       
    {
        source.visit([&](auto const& value)         
                     {        
                        *this=std::move(value);     
                     }        
                    );        
    }
}
```

### Assignment

赋值操作与复制构造和移动构造是类似的。

```c++
template<typename... Types>
Variant<Types...>& Variant<Types...>::operator=(Variant const& source)
{
    if(!source.empty())
    {
        source.visit([&](auto const& value)
                     {
                         *this=value;
                     }
                    );
    }
    else
    {
        destroy();
    }
    return *this;
}
```

这里唯一比较有趣的是当源Variant为空，也就是其discriminator为0的时候，我们直接调用destroy，而不需要再做别的操作。

