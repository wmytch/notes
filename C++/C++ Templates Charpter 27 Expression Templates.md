# Chapter 27 Expression Templates

[TOC]

表达式模板本来是用于支持数值数组类的。数值数组类支持对整个数组对象的数值运算。比方说

```c++
Array<double> x(1000),y(1000);
...
x=1.2x+x*y;
```

这种运算通常来说要求尽可能的高效，这并不是一个简单的任务，不过可以利用表达式模板来处理。

表达式模板通常依赖于深度嵌套的模板实例化，这与模板元编程很类似。实际上两者最初都是为了支持数组或者说矩阵的高性能运算发展而来的，并且两者是互补的，模板元编程适用于比较小的固定大小的数组，而表达式模板则适用于中到大规模并且大小动态变化的数组。

## 27.1 Templates and Split Loops

我们从一个直接的例子开始。

```c++
#include <cstddef>
#include <cassert>

template<typename T>
class SArray
{
    public:
        explicit SArray(std::size_t s):storage(new T[s]),storage_size(s)
        {
            init();
        }

        SArray(SArray<T> const& orig):storage(new T[orig.size()]),storage_size(orig.size())
        {
            init();
        }

        ~SArray()
        {
            delete[] storage;
        }

        SArray<T>& operator=(SArray<T> const& orig)
        {
            if(&orig!=this)
            {
                copy(orig);
            }
            return *this;
        }

        std::size_t size() const
        {
            return storage_size;
        }

		T const& operator[](std::size_t idx) const
        {
            return storage[idx];
        }
    
        T& operator[](std::size_t idx)
        {
            return storage[idx];
        }

    protected:
        void init()
        {
            for(std::size_t idx=0;idx<size();++idx)
            {
                storage[idx]=T();
            }
        }
        void copy(SArray<T> const& orig)
        {
            assert(size()==orig.size());
            for(std::size_t idx=0;idx<size();++idx)
            {
                storage[idx]=orig.storage[idx];
            }
        }
    private:
        T* storage;
        std::size_t storage_size;
};
template<typename T>
SArray<T> operator+(SArray<T> const& a,SArray<T> const& b)
{
    assert(a.size()==b.size());
    SArray<T> result(a.size());
    for(std::size_t k=0;k<a.size();++k)
    {
        result[k]=a[k]+b[k];
    }
    return result;
}

template<typename T>
SArray<T> operator*(SArray<T> const& a,SArray<T> const& b)
{
    assert(a.size()==b.size());
    SArray<T> result(a.size());
    for(std::size_t k=0;k<a.size();++k)
    {
        result[k]=a[k]*b[k];
    }
    return result;
}

template<typename T>
SArray<T> operator*(T const& s,SArray<T> const& a)
{
    SArray<T> result(a.size());
    for(std::size_t k=0;k<a.size();++k)
    {
        result[k]=s*a[k];
    }
    return result;
}
```

上面的这种实现不论是时间复杂度还是空间复杂度效率都很低：

1. 除了赋值操作符外，其他的运算符的实现都至少创建了一个临时的数组，这将会是相当大的空间占用，通常是不可接受的。
2. 每一个运算符都需要对参数和结果数组做额外的遍历，这也是很耗费时间的。

比方说对于
```c++
SArray<double> x(1000),y(1000);
…
x=1.2*x+x*y;
```
就会有

```c++
tmp1=1.2*x;  //循环1000次，加上对tmp1的创建和析构
tmp2=x*y;    //循环1000次，加上对tmp2的创建和析构
tmp3=tmp1+tmp2; //循环1000次，加上对tmp3的创建和析构
x=tmp3;      //1000次读和1000次写
```

为了减少对临时对象的使用，可以使用所谓的计算赋值，比方说`+=`、`*=`这样的，这样就不需要创建临时的对象而是使用调用者的空间：

```c++
template<typename T>
SArray<T>& SArray<T>::operator+=(SArray<T> const& b)
{
    assert(size()==b.size()); //书上这里是orig，显然应该是b，下面*=也是如此，就不另外说明了
    for(std::size_t k=0;k<size();++k)
    {
        (*this)[k]+=b[k];
    }
    return *this;
}
template<typename T>
SArray<T>& SArray<T>::operator*=(SArray<T> const& b)
{
    assert(size()==b.size()); 
    for(std::size_t k=0;k<size();++k)
    {
        (*this)[k]*=b[k];
    }
    return *this;
}
template<typename T>
SArray<T>& SArray<T>::operator+=(T const& s)
{
    for(std::size_t k=0;k<size();++k)
    {
        (*this)[k]+=s;
    }
    return *this;
}
```

如果我们这样来调用

```c++
SArray<double> x(1000),y(1000);
...
SArray<double> tmp(x);
tmp *=y;
x*=1.2;
x+=tmp;
```

也还是有些问题：

- 这种写法很拙劣

- 仍然需要一个临时对象tmp

- 仍然需要额外的读写

换句话就是这种实现，仅仅减少了两个临时对象的创建和析构，但是需要的读写操作并没有变化。

而实际上，有一种简单的实现，还是上面的x，y

```c++
for(int idx=0;idx<x.size(),++idx)
{
    x[idx]=1.2*x[idx]+x[idx]*y[idx];
}
```

也就是说，并不需要去实现`operator+`和`operator*`，一个循环就做完了，每次迭代只需要各自读一次x[idx]和y[idx]，写一次x[idx]，没有临时对象，也没有额外的读写。

## 27.2 Encoding Expressions in Template Arguments

解决问题的关键是在完整的看完一个表达式前不要计算其部分值。因此，在计算之前，需要记录哪些操作作用于哪些对象。这是可以在编译时进行的，因此可以使用模板参数来表示。

比方说对于`1.2*x+x*y`，`1.2*x`并不是一个新的数组，而是把x的每一个值都乘以1.2，同样`x*y`也不是一个新数组，只是把x和y对应的元素相乘。最后，我们需要结果的时候，再做计算，并存储下来以备后用。

我们可以这样来实现这个运算：`A_Add<A_Mult<A_Scalar<double>,Array<double>>,A_Mult<Array<double>,Array<double>>>`。大体上这就是个前缀表达式。

### 27.2.1 Operands of the Expression Templates

我们需要把每个A_Add和A_Mult对象的参数的引用保存起来，同时也要把A_Scalar对象中的标量的值或者引用记录下来

```c++
template<typename T>          
class A_Scalar;
   
template<typename T>          
class A_Traits                
{  
    public:                   
        using ExprRef=T const&;        
};

template<typename T>
class A_Traits<A_Scalar<T>>
{  
    public:
        using ExprRef=A_Scalar<T>;     
}; 
                              
template<typename T,typename OP1,typename OP2>
class A_Add
{  
    private: 
        typename A_Traits<OP1>::ExprRef op1;
        typename A_Traits<OP2>::ExprRef op2;
   
    public:                   
        A_Add(OP1 const& a,OP2 const& b):op1(a),op2(b){}
                              
        T operator[](std::size_t idx) const
        {                     
            return op1[idx]+op2[idx];      
        }
   
        std::size_t size() const       
        {
            assert(op1.size()==0||op2.size()==0||op1.size()==op2.size());
            return op1.size()!=0?op1.size():op2.size();
        }
}; 
template<typename T,typename OP1,typename OP2>
class A_Mult
{
    private:
        typename A_Traits<OP1>::ExprRef op1;
        typename A_Traits<OP2>::ExprRef op2;

    public:
        A_Mult(OP1 const& a,OP2 const& b):op1(a),op2(b){}

        T operator[](std::size_t idx) const
        {
            return op1[idx]*op2[idx];
        }

        std::size_t size() const
        {
            assert(op1.size()==0||op2.size()==0||op1.size()==op2.size());
            return op1.size()!=0?op1.size():op2.size();
        }
};

template<typename T>
class A_Scalar
{
    private:
        T const& s;
    public:
        constexpr A_Scalar(T const& v):s(v){}
        constexpr T const& operator[](std::size_t) const
        {
            return s;
        }
        constexpr std::size_t size() const
        {
            return 0;
        }
};                     
```

首先看A_Traits，从定义可以看到，对于通用参数，定义的是引用类型，而对于A_Scalar定义的是一个值类型。这里的意义在于，大多数临时节点绑定在表达式的最高层，因此其生存期会直到整个表达式求值完毕。但是对于A_Scalar节点，则是绑定于运算符函数，因此可能不能生存到整个表达式求值完成，因此，为了避免成员指向一个不存在的标量，A_Scalar作为操作数需要按值复制。也就是说，我们需要的成员是

- 通用情况下的常引用

```c++
OP1 const& op1;
OP2 const& op2;
```

- 对于标量则是普通的值

```c++
OP1 op1;
OP2 op2;
```

这也是特性类的一个完美的应用。

另外，对于A_Scalar本身，其成员可以是引用类型的，这与A_Scalar作为参数是两回事。

其余的问题代码很直白，就不多说了，只是要注意下A_Scalar中constexpr在这里并不是必须的，虽然其目的是为了这个类可以在编译时使用这些值。

### 27.2.2 The `Array` Type

为了能够使用一个轻量级的表达式模板来编码表达式，需要创建一个知道表达式模板的Array类型来控制实际的存储。当然，应该使存储数据的真实数组的接口与保存表达式而生成的数组的接口保持相似性。

```c++
template<typename T,typename Rep=SArray<T>>
class Array
{
    private:
        Rep expr_rep;
    public:
        explicit Array(std::size_t s):expr_rep(s){}
        Array(Rep const& rb):expr_rep(rb){}

        Array& operator=(Array const& b)
        {
            assert(size()==b.size());
            for(std::size_t idx=0;idx<b.size();++idx)
            {
                expr_rep[idx]=b[idx];
            }
            return *this;
        }

        template<typename T2,typename Rep2>
        Array& operator=(Array<T2,Rep2> const& b)
        {
            assert(size()==b.size());
            for(std::size_t idx=0;idx<b.size();++idx)
            {
                expr_rep[idx]=b[idx];
            }
            return *this;
        }

        std::size_t size() const
        {
            return expr_rep.size();
        }

        decltype(auto) operator[](std::size_t idx) const
        {
            assert(idx<size());
            return expr_rep[idx];
        }
    
		T& operator[](std::size_t idx)
        {
            assert(idx<size());
            return expr_rep[idx];
        }

        Rep const& rep() const
        {
            return expr_rep;
        }

        Rep& rep()
        {
            return expr_rep;
        }
};
```

Array的参数Rep可以是一个SArray，也可以是嵌套的A_Add或者A_Mult模板id，前者代表了一个真实存储数据的数组，后两者则表示了代表表达式的编码。另外，由于这里并没有完全使用SArray的所有特性，所以另外实现一个数组也是可以的。

不论哪种情况，我们都只需要处理Array的实例化，并不需要对Rep参数做特化，即使其某些成员无法用A_Mult来替换，这时候不用就是了。同时，我们看到，Array的操作都转发到了Rep对象上。

这里要特别说明下标操作符，对于const版本，没有返回`T const&`，而是使用了decltype，原因在于，如果Rep是A_Add或者A_Mult，这两个对象的下标操作符返回的是一个临时值，也就是一个prvalue，因此不能通过一个常量引用返回，而decltype则可以把一个prvalue推断为一个非引用。另一方面，如果Rep是`SArray<T>`，其返回的是一个`const lvalue`，推断出来的类型是匹配的。

### 27.2.3 The Operators

如前面所暗示过的，这些运算符只是把表达式模板对象组合起来，而不是对结果数组进行计算。对于每一个二元运算符，我们都需要实现三个版本：数组-数组，数组-标量，和标量-数组。

```C++
template<typename T,typename R1,typename R2>
Array<T,A_Add<T,R1,R2>> operator+(Array<T,R1> const& a,Array<T,R2> const& b)
{
    return Array<T,A_Add<T,R1,R2>>(A_Add<T,R1,R2>(a.rep(),b.rep()));
}

template<typename T,typename R1,typename R2>
Array<T,A_Mult<T,R1,R2>> operator*(Array<T,R1> const& a,Array<T,R2> const& b)
{
    return Array<T,A_Mult<T,R1,R2>>(A_Mult<T,R1,R2>(a.rep(),b.rep()));
}

template<typename T,typename R2>
Array<T,A_Mult<T,A_Scalar<T>,R2>> operator*(T const& s,Array<T,R2> const& b)
{
    return Array<T,A_Mult<T,A_Scalar<T>,R2>>(A_Mult<T,A_Scalar<T>,R2>(A_Scalar<T>(s),b.rep()));
}
```

当然，这里没有全部实现。代码看起来有点长，不过其实并没有做太多，比方说对于`operator+`，首先创建一个A_Add对象来代表操作符和操作数：

```C++
A_Add<T,R1,R2>(a.rep(),b.rep())
```

然后，把这个对象用一个Array对象封装起来并返回

```c++
return Array<T,A_Add<T,R1,R2>>(...)
```

因为这些操作都很类似，所以可以使用宏来实现其它的运算符。

### 27.2.4 Review

接下来我们自顶向下来看看这里发生了什么。

```c++
int main()
{
    Array<double> x(1000),y(1000);
    ...
    x=1.2*x+x*y;
}
```

在声明x和y的时候省略了Rep，所以取了缺省值`SArray<double>`，因此，x和y就是实际存储数据的数组。

接下来，编译器解析表达式`1.2*x+x*y`，首先是`1.2*x`，从而选用了这个版本的`operator*`:

```c++
template<typename T,typename R2>
Array<T,A_Mult<T,A_Scalar<T>,R2> operator*(T const& s,Array<T,R2> const &b)
{
    return Array<T,A_Mult<T,A_Scalar<T>,R2>>
        (A_Mult<T,A_Scalar<T>,R2>(A_Scalar<T>(s),b.rep()));
}
```

因此，结果的类型就是

```c++
Array<double,A_Mult<double,A_Scalar<double>,SArray<double>>>
```

结果的值则是一个`A_Scalar<double>`对象和一个`SArray<double>`对象构造而成的对象的引用，这里A_Scalar对象由1.2构造而来，SArray由x构造而来。

接下来`x*y`和`+`同理，就不详细说明了。总之，这个表达式最终编码的结果是

```c++
Array<double,
		A_Add<double,
				A_Mult<double,A_Scalar<double>,SArray<double>>,
				A_Mult<double,SArray<double>,SArray<double>>>
```

最后是赋值操作，在这个操作符里边对操作数，不论是数组还是标量，逐个的进行了实际的运算。就不详细说明了。

### 27.2.5 Expression Templates Assignments

上面的例子中，对于Rep参数为A_Add或者A_Mult的数组实例，写入操作是不可能的，实际上，`a+b=c`确实是毫无意义的。但是对于其他情况，还是有必要定义写操作的。比方说对于`x[y]=2*x[y]`，应该等价于

```c++
for(auto idx=0;idx<y.size();++idx)
{
    x[y[idx]]=2*x[y[idx]];
}
```

这样也就意味着建立在表达式模板基础上的数组其行为就是一个左值，也就是说可写的。

```c++
template<typename T,typename A1,typename A2>
class A_Subscript             
{  
    public:                   
        A_Subscript(A1 const& a,A2 const& b):a1(a),a2(b){}                                                                                                                                       
                              
        decltype(auto) operator[](std::size_t idx) const                                                                                                                                         
        {                     
            return a1[a2[idx]];        
        }
   
        T& operator[](std::size_t idx) 
        {
            return a1[a2[idx]];        
        }
    
        std::size_t size() const       
        { 
            return a2.size(); 
        }
    private:
        A1 const& a1;         
        A2 const& a2;         
};
```

在此基础上，要实现前面的`x[y]=2*x[y]`实际上还需要扩展Array的`operator[]`，原书上的例子是有问题的，这里就不继续讨论了。

## 27.3 Performance and Limitations of Expression Templates

表达式模板仍然在发展中，本章前面的例子依赖于编译器的处理，比方说正确处理inline函数，以及对总多小模板对象的处理，至少不能不如手写的循环。另外对于像`x=A*x`这样的向量矩阵运算结果是不正确的，因为x的值一直在被替换，而对于`x=A*y`，只要x和y不是同一个向量，结果就是正确的，这是表达式模板本身的问题。