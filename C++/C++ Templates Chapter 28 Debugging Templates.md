# Chapter 28 Debugging Templates

[TOC]

模板调试有两个挑战。一个是对模板开发者的：对于符合文档要求的任意模板参数，如何确保模板都能够正常工作。另一方面，则是对模板的使用者：当模板不能像文档所说的那样工作时，怎样找出不符合文档的模板参数。

对于模板参数的限制，如果违反了会引发编译错误，则称这种限制为句法限制，比方说要求有构造函数，调用函数时不出现不明确，等等。其它的情况则称为语义限制，这种限制很难界定，比方说对于一个模板类型参数需要定义`operator<`，这本身是一个句法限制，但是通常我们也需要为此定义某种形式的顺序，这就是一个语义限制。

concept这个术语，通常用来表示在模板库里持续要求的一系列限制。比如STL依赖于像随机访问迭代器和可缺省构造这样的concept。我们可以这么说，调试模板代码，大量的工作就是确定在模板的实现和使用中是如何违背concept的。

## 28.1 Shallow Instantiation

```c++
template<typename T>
void clear(T& p)
{
    *p=0;  //需要一个类指针的类型
}

template<typename T>
void core(T& p)
{
    clear(p);
}

template<typename T>
void middle(typename T::Index p)
{
    core(p);
}

template<typename T>
void shell(T const& env)
{
    typename T::Index i;
    middle<T>(i);
}

class Client
{
    public:
        using Index=int;
};  

int main()
{
    Client mainClient;
    shell(mainClient);
}
```

显然，由于最后传入clear的参数是个int，并不能对其做解引用，于是出错。对于本章的内容，这里的关键不是错误的原因，而是编译出错时的一大堆提示。

有两种方法来尽可能早的确定参数是否满足concept：通过语言扩展或者尽可能早的使用参数。前者可参见17.8和附录E，后者包括使错误发生在浅层实例化，也就是在代码中插入一些不会使用的代码，这些代码的的目的只是在实例化时，如果模板参数不符合在代码深层才出现的模板的要求，就引发错误。比方说前面的例子

```c++
template<typename T>
void ignore(T const&)
{}

template<typename T>
void shell(T const& env)
{
    class ShallowChecks
    {
        void deref(typename T::Index ptr)
        {
            ignore(*ptr);
        }
    };
    typename T::Index i;
    middle(i);
}
```

这样如果T是像T::Index这样无法解引用的类型，那么在就会在局部类ShallosChecks中引发错误，而由于这个类并没有被使用，所以并不会对编译结果和运行产生负面的影响。

### Concept Checking

显然，像这种使用哑代码的方法会增加代码的复杂度，于是出现了一些库，比方说Concept Check Library，这是Boost的一部分。另外，可移植性不好。

## 28.2 Static Assertions

`assert()`宏通常用来在程序执行时检查某些条件，如果不符合，则程序中止运行。C++11引入的static_assert目的相同，不过是在编译时做断言。比方说`static_assert(sizeof(void*)*CHAR_BIT==64,”Not a 64-bit platform”);`可以用来检查一个平台是否64位。

为了解决上一节的问题，可以使用19.4介绍的技术

```c++
#include <utility>
#include <type_traits>
template<typename T>
class HasDereference
{
    private:
        template<typename U> struct Identity;
        template<typename U>
        static std::true_type test(Identity<decltype(*std::declval<U>())>*); //#1
        template<typename U>
        static std::false_type test(...);
    public:
        static constexpr bool value=decltype(test<T>(nullptr))::value;
};
...
template<typename T>
void shell(T const& env)
{
    static_assert(HasDereference<T>::value,"T is not deferenceable");
    typename T::Index i;
    middle<T>(i);
}
```

为了避免忘记，稍微解释一下#1，这里关键就是这一句，test的参数是一个指针，这个指针指向一个Identity类型，当然这不是重点，重要的是这个模板参数的处理，declval构建一个对象，如果U是一个可解引用的类型，则decltype(*U)不会出错，否则，这里decltype就会出错，于是选择另外一个版本的test，这样就可以给static_assert返回对应的value。

静态断言非常有用而且好用，标准库也提供了许多特性来可以用于静态断言中，可参见附录D。

## 28.3 Archetypes

写模板的挑战在于，对符合文档所述的所有参数，都应该可以通过编译。比方说对于这样的一个文档

```c++
//T 必须是EqualityComparable,也就是两个类型T的对象可以通过==来比较，并将结果转换成bool
template<typename T>
int find(T const* array,int n,T const& value);
```

我们可以想象得到如下这种很直接的实现

```c++
template<typename T>
int find(T const* array,int n,T const& value)
{
    int i=0;
    while(i!=n && array[i]!=value)
        ++i;
    return i;
}
```

这种实现是有问题的，即便是对于技术上满足文档的参数T。

我们可以设计一个原型，用来测试模板的定义是否满足文档。原型只提供模板定义所需要的操作，如果通过编译，则说明模板定义没有使用超出文档的操作。

```c++
class EqualtyComparableArchetype
{};
class ConvertibleToBoolArchetype
{
    public:
    	operator bool() const;
};
ConvertibleToBoolArchetype operator==(EqulityComparableArchetype const&,
                                      EqulityComparableArchetype const&);
```

显然，因为定义了`operator==`，EqulityComparableArchetype是满足上面文档要求的。

接下来，我们用EqulityComparableArchetype作为参数来实例化`find<T>`，于是出错，因为find里对T的比较操作依赖于`!=`，而不是`==`，因此，我们找到了第一个问题。把`array[i]!=value`改成`!(array[i]==value)`后，编译通过，当然，因为只是声明而没有定义`operator==`，链接时会出错，不过这个没关系，原型本来就是用来编译期检查的。

然而还有一个问题，就是这里依赖于用户定义的bool转换以及内建的`operator!`，如果我们重新定义ConvertibleToBoolArchetype

```c++
class ConvertibleToBoolArchetype
{
    public:
    	operator bool() const;
    	operator!()=delete;
};
```

那么就又出错了。

原型是可以扩展的，一般来说，模板开发者会依据文档里定义的concept来开发一个原型，用来检查每一个模板的定义是否符合文档要求。

## 28.4 Tracers

跟踪器是用户自定义类，用来作为待测试模板的参数，因为通常需要能够满足待测试模板的参数要求，因此也是一个原型。更重要的是，跟踪器可以生成跟踪信息。这样就可以在运行时对模板进行调试。

```c++
#include <iostream>
#include <algorithm>

class SortTracer
{
    private:
        int value;
        int generation;
        inline static long n_created=0;
        inline static long n_destroyed=0;
        inline static long n_assigned=0;
        inline static long n_compared=0;
        inline static long n_max_live=0;

        static void update_max_live()
        {
            if(n_created - n_destroyed > n_max_live)
            {
                n_max_live=n_created - n_destroyed;
            }
        }

    public:
        static long creations()
        {
            return n_created;
        }

        static long destructions()
        {
            return n_destroyed;
        }

        static long assignments()
        {
            return n_assigned;
        }

        static long comparisons()
        {
            return n_compared;
        }
        
        static long max_live()
        {
            return n_max_live;
        }

        SortTracer(int v=0):value(v),generation(1)
        {
            ++n_created;
            update_max_live();
            std::cerr<<"SortTracer #"<<n_created
                    <<",created generation "<<generation
                    <<" (total: "<<n_created - n_destroyed
                    <<")\n";
        }

        SortTracer(SortTracer const& b):value(b.value),generation(b.generation)
        {
            ++n_created;
            update_max_live();
            std::cerr<<"SortTracer #"<<n_created
                    <<",copied as generation "<<generation
                    <<" (total: "<<n_created - n_destroyed
                    <<")\n";
        }

        ~SortTracer()
        {
            ++n_destroyed;
            update_max_live();
            std::cerr<<"SortTracer generation "<<generation
                    <<" destroyed (total: "<<n_created-n_destroyed
                    <<")\n";
        }
        
        SortTracer& operator=(SortTracer const& b)
        {
            ++n_assigned;
            std::cerr<<"SortTracer assignment #"<<n_assigned
                    <<" (generation "<<generation
                    <<" = "<<b.generation
                    <<")\n";
            value=b.value;
            return *this;
        }

        friend bool operator< (SortTracer const& a,SortTracer const& b)
        {
            ++n_compared;
            std::cerr<<"SortTracer comparison #"<<n_compared
                    <<" (generation "<<a.generation
                    <<" < "<<b.generation
                    <<")\n";
            return a.value<b.value;
        }

        int val() const
        {
            return value;
        }
};

int main()
{
    SortTracer input[]={7,3,5,6,4,2,0,1,9,8};

    for (int i=0;i<10;++i)
    {
        std::cerr<<input[i].val()<<' ';
    }
    std::cerr<<std::endl;

    long created_at_start=SortTracer::creations();
    long max_live_at_start=SortTracer::max_live();
    long assigned_at_start=SortTracer::assignments();
    long compared_at_start=SortTracer::comparisons();

    std::cerr<<"---[ Start std::sort() ]-----------------------\n";
    std::sort<>(&input[0],&input[9]+1);
    std::cerr<<"---[ End std::sort() ]-------------------------\n";
    
    for (int i=0;i<10;++i)
    {
        std::cerr<<input[i].val()<<' ';
    }
    std::cerr<<std::endl;
    
    std::cerr<<"std::sort() of 10 SortTracer's"
            <<" was performed by:\n "
            <<SortTracer::creations()-created_at_start
            <<" temporary tracers\n"
            <<" up to "
            <<SortTracer::max_live()
            <<" tracers at the same time ("
            <<max_live_at_start<<" before)\n "
            <<SortTracer::assignments()-assigned_at_start
            <<" assignments\n "
            <<SortTracer::comparisons()-compared_at_start
            <<" comparisons\n\n";
}
```
运行结果输出很长，不过我们可以只看下面这些
```bash
std::sort() of 10 SortTracer's was performed by:
 8 temporary tracers
 up to 11 tracers at the same time (10 before)
 16 assignments
 31 comparisons
```

具体的就不说了，跟踪器起了两个作用：一方面是证明了标准`sort()`算法不需要我们的跟踪器提供更多的功能，比方说并不需要`==`和`>`，另一方面，直观的显示了这个算法的运行代价。然而，这里对排序模板的正确性并没有太多的说明。

## 28.5 Oracles

跟踪器相对简单并且有效，但是我们只能用其来跟踪特定数据和特定行为的执行情况。比方说，前面的例子只是测试出来跟踪器的比较操作其行为确实与整数的小于操作相似，但是，并没有说明对于排序算法，什么样的比较操作是有意义的或者说正确的。

跟踪器有一种扩展，在某些圈子里称为oracle，或者运行时分析oracle，这种跟踪器连接一个推理引擎，这是一个程序，可以记住断言并对其进行推理，最终得到一个结论。

oracle可以动态的验证模板算法，而不需要完全替换模板参数或者指定输入数据，因为其本身就可以做为模板参数。由于其复杂性，这里只是提一下。