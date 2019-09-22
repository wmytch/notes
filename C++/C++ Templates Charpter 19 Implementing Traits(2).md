# Charpter 19 Implementing Traits(2)

[TOC]



## 19.4 SFINAE-Based Traits

SFINAE一方面避免函数重载时的虚报错误，至于什么是虚报错误，下面再解释，另一方面也可以在编译时确定一个类型或者表达式是否合法。据此我们就可以写出这样的特性，可以决定一个类型是否存在某个成员，是否支持某个操作，或者就是某种类型。

所谓基于SFINAE的特性这里指的是通过SFINAE剔除函数重载以及剔除部分特化。

### 19.4.1 SFINAE Out Function Overloads

```c++
template<typename T>
struct IsDefaultConstructibleT
{
  private:
  	template<typename U,typename=decltype(U())>
  	static char test(void*);
  	template<typename>
  	static long test(...);
  public:
  	static constexpr bool value
      =IsSameT<decltype(test<T>(nullptr)),char>::value;
};
```

这里有两个重载函数，先说第二个，这是所谓的fallback，就是在没有别的选择的情况下，可以选择这个，姑且称为后备，或者像switch语句那样叫缺省也可以，这个重载函数可以匹配所有的参数，但是由于其参数是可变长参数，这在解析重载时是最后才考虑的。

对于第一个，这样设计的目的是要检查U是否具有缺省构造函数，不论是显式的还是隐式的，意思就是能否缺省的构造一个T类型的对象。当然在下面IsSameT调用`decltype(test<T>(nullptr)`的地方，我们可以看到这个U就是T，这点没关系，只是提醒一下。用`delctype(U())`使得这里产生的是一个类型而不是一个对象。这里不能做推断，因为函数没有调用参数，我们也不会显式的提供一个模板参数，因此，如果U不能缺省构造，就会引发SFINAE，这个函数就会被剔除掉。

另外，**要注意的是这里的两个重载函数并不需要定义，声明就可以了，下面的例子也是如此**。

因此`IsDefaultConstructibleT<int>::value`的值是true，而对于

```
struct S
{
	S()=delete;
};
```

`IsDefaultConstructibleT<S>::value`的值就是false。

注意我们不能在第一个test中直接使用模板参数T：

```C++
template<typename T>
struct IsDefaultConstructibleT
{
  private:
  	//ERROR
  	template<typename,typename=decltype(T())>
  	static char test(void*);
  ...
};
```

因为这个时候如果T是个不能缺省构造的类型，由于所有的T都会被替换，那么就会引发一个编译错误，而不是在重载解析时引发SFINAE。

#### Alternative Implementation Strategies for SFINAE-based Traits

基于SFINAE的特性的关键是声明两个返回类型不同的重载函数模板，这是一种在C++98标准公布之前就存在的技术。

```c++
template<...> static char test(void*);
template<...> static long test(...);
//那时候还不存在nullptr和constexpr
enum { value= sizeof(test<...>(0))==1 };
```

如果在某个平台上`sizeof(char)==sizeof(long)`，那么也可以这样

```c++
using Size1T=char;
using Size2T=struct{ char a[2];};
```

或者

```c++
using Size1T=char(&)[1];
using Size2T=char(&)[2];
```

于是

```c++
template<...> static Size1T test(void*);
template<...> static Size2T test(...);
```

注意体会这两种替代方法所体现出来的思想，这样的技巧仍然还是很常见的。

注意对于test()来说，其调用参数的类型是什么是没有关系的，这里的关键是考察模板参数是否具有某种属性，比方说前面的是否有缺省构造函数的例子，我们可以这样

```c++
template<...> static Size1T test(int);
template<...> static Size2T test(...);
enum {value=sizeof(test<...>(42))==1};
```

#### Making SFINAE-based Traits Predicate Traits

当然，技术发展到现在，也有其它方法来解决在`sizeof(char)==sizeof(long)`的平台上的问题

```c++
template<typename T>
struct IsDefaultConstructibleHelper
{
  private:
  	template<typename U,typename = decltype(U())>
  	static std::true_type test(void*);
  	template<typename>
  	static std::false_type test(...);
  public:
  	using Type=decltype(test<T>(nullptr));
};
template<typename T>
struct IsDefaultConstructibleT::IsDefaultConstructibleHelper<T>::Type
{};
```

因此，`IsDefaultConstructibleT<T>`就根据T最终继承了`std::true_type`或者`std::false_type`，注意不是`IsDefaultConstructibleHelper<T>`。使用的时候就需要`IsDefaultConstructibleT<T>::value`，这是个bool值，与前面的例子形式上是一样的。

#### 我们来看看这里发生了什么。

1. 为了决定一个类型T是否可以缺省构造一个实例，创建了两个重载函数，其返回值的类型各自不同；

1. 用返回值的类型与一个选定的类型作比较，当然这个选定的类型是特意的，也就是在满足类型T可以缺省构造的情况下，所选定的重载函数的返回值类型，比如char。如果比较结果两个类型一致，那么说明类型T满足条件。

1. 介绍了一种长久以来，也就是还没有constexpr和decltype的时候，所使用的方法，也就是用enum来得到一个常量值，以及使用sizeof来比较返回值。

1. 介绍了在某些平台上，如果无法确定不同数据类型长度不一样的情况下，所采用的的一种技巧，也就是人为的构造两个大小肯定不同的类型。

1. 介绍了直接使用test函数的返回值做为结果的方法，就是不再做比较，而是直接使用返回的布尔值类型，从而也解决了平台限制的问题。

1. 为什么要使用特性？实际上，上面3、4所介绍的方法就不必一定放到特性里边去，但是由于enum不能适用模板，只能这样使用

   ```c++
   template<typename U,typename=decltype(U())>
   char test(void*);
   template<typename>
   long test(...);
   //随便什么名字吧，不用value用isDefaultConstructible当然也是可以的
   template<typename T>
   bool value=sizeof(test<T>(0))==1;
   
   ...
     
   std::cout<<value<IsDefaultConstructibleT<int>><<std::endl; //1
   std::cout<<value<IsDefaultConstructibleT<S>><<std::endl;   //1
   std::cout<<value<S><<std::endl;  //0
   std::cout<<value<int><<std::endl; //1
   ```

1. 为什么test不直接返回bool而是返回两个其他类型？因为这里并不是要检查test函数的返回值的值，而只是检查其类型，如上面看到的，这两个函数甚至只有声明而没有定义，所以返回bool类型是没有意义的。这也是前面最后一个方法返回true_type和false_type的原因，这里也可以看出标准设立这两个类型的目的。

1. 最后说一句，所谓泛型，泛的就是数据类型，虽然generic本身没有体现出型来，但是这个翻译还是很贴合本意的。

### 19.4.2 SFINAE Out Partial Specializations

```c++
template<typename...> using VoidT=void;

template<typename, typename=VoidT<>>
struct IsDefaultConstructibleT:std::false_type
{
};

template<typename T>
struct IsDefaultConstructibleT<T,VoidT<decltype(T())>>:std::true_type
{
};

int main()
{
	std::cout<<IsDefaultConstructibleT<int>::value<<std::endl;
	std::cout<<IsDefaultConstructibleT<int,void>::value<<std::endl;
	std::cout<<IsDefaultConstructibleT<int,int>::value<<std::endl;
	std::cout<<IsDefaultConstructibleT<int,S>::value<<std::endl;
	std::cout<<IsDefaultConstructibleT<S>::value<<std::endl;
	std::cout<<IsDefaultConstructibleT<S,void>::value<<std::endl;
}
```

运行结果是

```bash
1
1
0
0
0
0
```

我们看看发生了什么。

- 这里`VoidT<>`不管参数是什么，最终都会是一个void，因此不论T是什么，只要能够缺省构造，`IsDefaultConstructibleT<T,VoidT<decltype(T())>>`生成的模板都是`IsDefaultConstructibleT<T,void>`
- 对于`IsDefaultConstructibleT<int>`，由于主模板有缺省模板参数，部分特化会继承主模板的缺省参数，这里是void，因此`IsDefaultConstructibleT<int>`便等同于`IsDefaultConstructibleT<int,void>`。在寻找匹配时，由于int是可以缺省构造的，部分特化不引发SFINAE，于是匹配。
- 对于`IsDefaultConstructibleT<int,int>`和`IsDefaultConstructibleT<int,S>`，由于第二个参数与void不匹配，因此部分特化被SNIFAE掉，选择了主模板。
- 对于`IsDefaultConstructibleT<S>`，如前说述，等同于`IsDefaultConstructibleT<S,void>`，而S不能缺省构造，因此部分特化被SNIFAE掉，选择了主模板。

说句题外话，从书上例子可以看出来几位作者的缩进风格或者代码规范是不一样的。当然，本笔记里抄录的代码基本上是按照我自己习惯做了改变的，除非忘了。

在C++17中，有个特性`std::void_t<>`等同于上面的`VoidT<>`。在c++17之前，我们要么像上面那样定义一个`VoidT<>`，要么在std中增加定义：

```C++
#ifndef __cpp_lib_void_t
namespace std 
{
  template<typename...> using void_t=void;
}
#endif
```

正如以前说过的，标准委员会是不允许修改std的，然而这只是所谓的未定义行为，所以一些编译器实现允许这么做，反正是未定义的，只要编译器支持就行，并没有超出标准规定。

像__cpp_lib_void_t这样所谓的特性宏，是标准委员会建议编译器加以支持，用来表示其实现了标准的哪一部分，不过不知道msvc现在支持了没有，gcc肯定是没问题的。

### 19.4.3 Using Generic Lambdas for SFINAE

在c++17，我们可以使用泛型lambda简化前面的问题。首先要说明的是decltype当中不能直接使用一个lambda表达式，下面只是出于方便说明的目的，才直接把一个lambda表达式放进decltype当中。当然，一个lambda表达式类型的变量是可以放进decltype当中的。另外，`declval<T>()`返回一个T&&类型的对象，除非T是void，这时候会返回void。

```c++
template<typename F,typename... Args,
	typename=decltype(std::declval<F>()(std::declval<Args&&>()...))>
std::true_type isValidImpl(void*);

template<typename F,typename... Args>
std::false_type isValidImpl(...);

inline constexpr
auto isValid=[](auto f){
  return [](auto&&... args){
    return decltype(isValidImpl<decltype(f),decltype(args)&&...>(nullptr)){};
  };
};

template<typename T>
struct TypeT
{
  using Type=T;
};

template<typename T>
constexpr auto type=TypeT<T>{};

template<typename T>
T valueT(TypeT<T>);
```

我们从头开始看，这里有两个函数模板`isValidImpl()`的非定义声明。注意这里是函数模板重载，不是类模板，不存在主模板和部分特化，也就不存在继承缺省参数的问题。

第一个模板参数列表里有个缺省参数，SFINAE就是通过这个缺省参数来实现。而第二个模板的模板参数列表中第一个参数是可以不要的，也就是说改成`template<typename… Args>`也是可以的。

对于第一个模板的缺省参数`decltype(std::declval<F>()(std::declval<Args&&>()…))`:

- 我们知道`declval<F>()`总是能够构造出一个F&&对象来，如果将其直接作为decltype的参数，那么delctype生成的将是一个F&&类型。
- 这个F我们已知是`isValidImpl<>`的第一个模板参数，出于我们的目的，这个F是一个lambda表达式类型，而不是一个任意的类型，注意是类型。
- `declval<F>()`构造出来的这个F&&对象，也就是一个lambda表达式，做了一次调用，其参数是可变长参数`std::declval<Args&&>()…`，从而得到其返回值，也就是F这个lamda表达式的返回值。当然，并没有做真正的调用操作，只是通过分析句法得到而得到的一个表达式。
- 然后通过decltype获取到F的返回值的类型，这个返回值的类型就这里而言是无关紧要的。

这里还有一个问题要注意的是，对于第一个模板的参数列表，缺省参数位于可变长参数之后。通常我们不这么做，是因为这个参数无法直接获取。就此例而言所有列出来的模板实参都属于可变长参数集，然后加上一个没有显式指定的缺省参数，也就是说，缺省参数就是没有列出来的那个，但这并不是个语法错误。我们这里这样指定一个缺省参数，目的是让编译器在解析模板的时候通过这个缺省参数来决定是否匹配，或者说，我们自己是不会去使用这个缺省参数的，而是让编译器在参数替换时进行一些判断。

我们接下来看isValid，这是一个lambda表达式类型，并且是个constexpr类型的变量，因为c++没有直接定义闭包类型，所以这里使用了auto。在c++17之前，lambda表达式不能出现在常量表达式中。

isValid接受一个参数`f`，返回如下的一个lambda表达式

```c++
[](auto&&... args){
    return decltype(isValidImpl<decltype(f),decltype(args)&&...>(nullptr)){};
}
```

isValid返回的这个lambda表达式，其返回值的类型或者是true_type，或者是false_type。

到这里，必须要注意的是在`isValidImpl<decltype(f),decltype(args)&&…>`中对`f`的使用。按道理，`f`没有被捕获的情况下，在这个闭包里使用`f`是非法的。但是因为decltype以及declval都是针对类型进行运算，与变量的本身的值没有关系，或者说，编译器在解析模板时只是做了参数替换，还没有进入到处理函数调用的阶段，所以编译器不会报错。如果直接使用`f`，编译器就会报错了。也可以这样理解，像`decltype(f+arg) f+arg`这样的表达式，出现的两个`f+arg`是分别在编译的不同阶段处理的。

接下来看变量模板`type`，这是一个`TypeT<T>`类型的常量。而就`valueT`这个函数而言，我们不需要调用这个函数，只是在编译时获取其返回值类型，因此只做了声明而没有定义。假定我们有一个值`x=type<arg>`，那么我们可以通过`decltype(valueT(x))`来获取到这个arg，当然，这是个类型。

注意这里不论是x还是`type<arg>`都是一个值，不是类名，因此是不能用`x::Type`或者`type<arg>::Type`来获取这个arg的。而decltype(x)则是x本身的类型，即`TypeT<arg>`，因此，我们可以用`delctype(x)::Type`来获取Type，也就是arg，这与`decltype(valueT(x))`结果是一样的。

我们可以这样来调用isValid：

```c++
constexpr auto isDefaultConstructible
	= isValid([](auto x) -> decltype((void)decltype(valueT(x))()){});
```

可以看到isValid的调用参数是一个lambda表达式

```c++
[](auto x) -> decltype((void)decltype(valueT(x))()) {}
```

这个表达式的本体为空，返回值的类型是`decltype((void)decltype(valueT(x))())`，可以看到已经强制转换成了void，其实我们这里并不在意这个具体返回值类型是啥。

通过替换参数，也就是把isValid的参数`f`替换成上面这个lambda表达式，我们可以得到如下结果，注意这里只是示意说明，并不是合法的C++语句，因为lambda表达式不能直接出现在decltype表达式中，所以不要纠结原书上`f`的`{}`去哪了

```c++
constexpr auto isDefaultConstructible
	= [](auto&&.. args){
		return decltype(
			isValidImpl<decltype([](auto x) -> decltype((void)decltype(valueT(x))()) {}),
				decltype(args)&&...
			>(nullptr)){};
	};
```
我们回头看isValidImpl的声明

```c++
template<typename F,typename... Args,
	typename=decltype(std::declval<F>()(std::declval<Args&&>()...))>
std::true_type isValidImpl(void*);

template<typename F,typename... Args>
std::false_type isValidImpl(...);
```

对于第一个声明的缺省参数，如前所述，`std::declval<F>()`大体上也就是`std::declval<decltype([](auto x)-> decltype((void)decltype(valueT(x))()) {})>()`，于是生成一个lambda表达式的右值引用，这个lambda表达式是`[](auto x)-> decltype((void)decltype(valueT(x))())) {}`，然后用参数`std::declval<Args&&>()...`对这个生成的对象做了一次调用，得到一个返回值类型的表达式即`decltype((void)decltype(valueT(x))()))`。最终这里的问题就转化为后面的`decltype(valueT(x))`所构造出来的这个类型能否缺省构造，也就是如前面所述`x=type<arg>`中的arg能否缺省构造，再次强调下，这里x不是个任意的值，而是type这个模板变量。当然，也要再次强调下，如果这里能够缺省构造出一个对象，这个对象也会被转换成void类型，如前所述，这里我们并不在意这一点。

于是,如果x的类型可以缺省构造，那么就选择第一个isValidImpl模板，返回`std::true_type`，否则，第一个模板就被SFINAE掉，从而选择后备模板，返回`std::false_type`。

我们先看`isDefaultConstructible(type<int>)`，还是以上面的示意说明为基础，这时候

```c++
isDefaultConstructible(type<int>) =[](type<int>){
    return decltype(isValidImpl<
                    decltype([](auto x)-> decltype((void)decltype(valueT(x))()) {}),
                    decltype(type<int>)
                    >(nullptr)){};//注意这里的{}表示初始化，不是函数体
 }
```

如前面所述，isValidImpl还有个缺省参数，这个缺省参数此时变成

```c++
decltype(std::declval<F>()(std::declval<decltype<type<int>>()))
```

如前所述`declval<F>()`生成一个类型为F的对象实例，这里F指的是什么前面说过就不再多说了，这个对象实例就是lambda表达式`[](auto x)-> decltype((void)decltype(valueT(x))()){}`，并且这个表达式的形参是x，而实际参数会是`std::declval<decltype<type<int>>()`，`decltype<type<int>>`得到`TypeT<int>`，`declval<TypeT<int>>`得到一个类型为`TypeT<int>`的对象，我们姑且用x来表示。于是，isValidImpl的缺省参数最终就可以看成是

```c++
decltype((void)decltype(valueT(x))())
```

这样问题最终就变成前面所讨论过的`decltype(valueT(x))`能否缺省构造的问题，`decltype(valueT(x))`产生的类型是int，这是可以缺省构造的，换句话说就是第一个isValidImpl模板是匹配的，从而`isValidImpl<>(nullptr)`返回true_type，`decltype(true_type){}`得到的值为true，于是`isDefaultConstructible(type<int>)`就是true。

而对于`isDefaultConstructible(type<int&>)`，`int&`是不能缺省构造的，这时候引发SFINAE，选择了第二个也就是后备模板，于是最终`isDefaultConstructible(type<int&>)`就是false。

细节已经说了，接下来看看这里用lambda表达式来实现SFINAE的逻辑是什么。

首先，假定我们要解决的问题是判断一个类型能否缺省构造，其次我们要用lambda表达式来解决这个问题。为了实现这两个要求，我们可以构造一个lambda表达式：
```c++
constexpr auto isDefaultConstructible=[](auto&&... args){
    return delctype(isValidImpl<...>(nullptr)){};//返回真或假
}
```

我们可以通过`isDefaultConstructible(args...)`来判断`args…`是否可以缺省构造，当然，虽然这里使用了可变长参数，但是出于我们的目的，通常也只会传一个参数，而且，通过之前的讨论我们知道，这里并不是直接对`args…`本身做判断。

接下来问题是，通过什么方式来返回真或假，我们上面已经看到了，这里是通过`decltype(...){}`返回一个true_type或者false_type的实例。不同的返回值类型可以通过重载isValidImpl来得到。提醒一下，这里的工作都是在编译阶段进行，所以返回一个bool值是达不到目的的。

然后，怎样判断应该选择哪一个重载函数？当然这里是通过SFINAE，于是归结到isValidImpl函数上。如果只是限于判断对象能否缺省构造，也并不需要那么绕的办法，但是正如下面会说到的，可以通过isValidImpl来做很多判断，需要判断的内容可以作为其第一个模板参数传入，并在其缺省参数中进行判断。

所以这里就用到了isValid，一方面，生成一个用来断言的lambda表达式，另一方面，给isValidImpl绑定一个条件，或者说，isValid就是一个工厂，生产断言lambda表达式的工厂。

接下来的问题是，为什么判断的条件要用lambda表达式的形式传入，还有没有别的方式呢？当然是有的，不过这里不讨论就是了。而且，最关键的是，正如前面说的，把需要判断的内容放到一个lambda表达式的返回值表达式中，可以触发SFINAE而不是编译错误。

最后我们按照时间线来捋一下这里发生了什么。

- 首先是两个函数模板的声明，如以前一样，两个函数模板的返回值类型不同。

- 接下来是isValid的定义，我们知道c++对lambda表达式的处理是生成一个类，大体上会是这样

    ```c++
    class IsValid{
        public:
        	isValid();
        	auto operator()(auto f)
            {
                return [](auto&&... args){...};
            }
    } isValid;
    ```

    可以看到`operator()`的返回值也是个lambda表达式，处理成类之后定义大体上如下所示，
	```c++
	class XXX{
	    public XXX();
	    auto operator()(auto&&... args)
	    {
	        return decltype(isValidImpl<decltype(f),decltype(args)&&...>(nullptr)){};
	    }
	};
	```
- 之后TypeT、type和valueT的声明也可以先略过。
- 然后是isDefaultConstructible的声明，我们知道这是个lambda表达式，根据我们对前面isValid的讨论，可以知道其定义可以看成是

    ```c++
    class IsDefaultConstructible{
        public:
        	IsDefaultConstructible();
        	auto operator()(auto&&... args)
            {
                return decltype(isValidImpl<decltype(f),decltype(args)&&...>(nullptr)){};
            }
    } isDefaultContstructible;
    ```

    注意一下，上面的`f`指代isValid的参数`[](auto x)-> decltype((void)decltype(valueT(x))()){}`，就像之前说的，在decltype中不能直接使用lambda表达式，而c++会把lambda表达式处理成一个类，也就是

    ```c++
    class FFF{
        public:
        	FFF();
        	auto operator()(auto x) -> deltype((void)decltype(valueT(x))())
        	{
                
            }
    } f;
    ```

- 接下来，当编译器看到`isDefaultConstructible(type<int>)`时，就会开始生成一系列的代码。无关理解的地方就省略了。

    ```c++
    isDefaultConstructible(type<int>)
    => isDefaultConstructible(auto x{type<int>})
    => isDefaultConstructible.operator()(x)
       {
       		return decltype(isValidImpl<decltype(f),decltype(x)&&>(nullptr)){};
       }
    ```

- 然后就是对函数模板isValidImpl的匹配，如果匹配第一个模板，就得到true_type，否则得到false_type，细节前面已经说过，就不再啰嗦，只是要记得，这都是在模板参数替换过程发生的，在进入下一阶段的编译之前，这里的返回值已经确定了。

最后尝试解释下这里为什么只是引发SFINAE而不是个编译错误。我们已经知道SFINAE和编译错误发生在编译的不同阶段，SFINAE发生在模板参数替换阶段，而编译错误随后才会有可能发生，至于具体在哪个阶段，没有资料我不瞎说，总之，两者有个先后次序，SFINAE剔除掉的模板不会进入下一个编译阶段，也就不会引发编译错误。

这里的关键之一是isValidImpl模板的第一个参数是调用isValid的参数，也就是在构造IsConstructible这个类时调用isValid时的参数，这个参数传入之后，就绑定在isValidImpl的模板参数上，或者说替换了isValidImpl的第一个模板参数，从而变成isValidImpl的即时上下文的一部分，也就是说不需要进入这个参数的本体当中，只需要对其返回值的声明作分析就可以了。这也是书上说得益于isValid定义的细节从而不会引发编译错误的意思所在。这点对于引发SFINAE很重要，因为SFINAE只发生在即时上下文中。

关键之二就是如上面我们看到，经过一系列代码生成后，最后归结到对模板isValidImpl的匹配上，而这个时候其模板实参都已经确定，不需要再去找其形参的具体定义，也就是替换只发生在isValidImpl的即时上下文中。匹配之后，就可以生成一个确定的值，如上面所述，到了编译下一个阶段，已经看不到isValidImpl这个模板了，看到的只是一个true_type或者false_type。

当然，如果没有后备模板，引发SFINAE之后没有可选的重载函数，这时候还是会引发编译错误的。

我们也可以简化一下这里的形式

```c++
template<typename T>
using IsDefaultConstructibleT
	= decltype(isDefaultconstructible(std::declval<T>()));
```

区别在于，这种表示方式只能用于所谓的名字空间范围，也就是只能出现在函数外，而本节所用的isDefaultconstructible是可以出现在块范围内的。

虽然这里的方法比较复杂，但是一旦理解了isValid，我们可以很方便的生成各种特性，比方说，我们要检查一个对象是否有个叫first的成员，就可以这样

```c++
constexpr auto hasFirst
= isValid([](auto x) -> decltype((void)valueT(x).first){});
```
这里最直接的理解就是，判断作为isValid的参数的lambda表达式的返回值类型表达式是否满足某种条件，这个返回值表达式怎么获得呢？就是通过`std::declval<F>()(std::declval<Args&&>()…)`，对F做一次调用，得到其返回值的表达式。

### 19.4.4 SFINAE-Friendly Traits

通常来说，一个类型特性应该能够回答一个特定的查询，而不会使程序变成非良构的。基于SFINAE的特性通过把问题沉入到SFINAE上下文当中，使得那些可能本来会是错误的查询变成一个负面的结果。这几句算是对上一节的归纳。

在进入下面讨论前，我们需要复习一下SFINAE的即时上下文是什么，这是通过定义什么**不是**即时上下文而得到的定义。因此，下面列出的都**不是**即时上下文，除此之外的都是即时上下文：

- 类模板的定义，指其定义本体，也就是`{}`之间的部分，以及基类列表，也就是类名后`:`到`{}`之间的部分。
- 函数模板的定义，与上面类似，定义本体，如果是构造函数，还包括其初始化列表。
- 变量模板的初始化器，比方说`tempalte<T> T v=T{}`中`{}`之间的部分。
- 缺省参数。
- 缺省的成员初始化器，同样指相应的`{}`之间的部分。
- 异常指令。
- 以及在替换过程中触发的对成员函数的隐含定义。

接下来，我们可以先看个反面的例子，在19.3.4中有

```c++
template<typename T1,typename T2>
struct PlusResultT
{
    using Type=decltype(std::declval<T1>() + std::declval<T2>());
};
template<typename T1,typename T2>
using PlusResult=typename PlusResultT<T1,T2>::Type;
```

上面定义中的`+`是没有SFINAE保护的，如果碰上的类型没有合适的`+`运算，就会编译出错。

```c++
template<typename T>
class Array
{
    ...
};
//#1
template<typename T1,typename T2>
Array<typename PlusResultT<T1,T2>::Type>
operator+(Array<T1> const&,Array<T2> const&);


class A{};
class B{};
void addAB(Array<A> arrayA,Array<B> arrayB)
{
    //注意，实际代码中应该还要移除返回值的引用，这里做了简化，只是为了突出问题所在
    auto sum=arrayA+arrayB;
    ...
}
```

显然，`arrayA+arrayB`会在实例化`PlusResult<A,B>`时出错。那么如果我们添加一个重载函数模板会怎样？

```c++
//#2
Array<A> operator+(Array<A> const& arrayA,Array<B> const& arrayB);  
```

当然，这时候如果编译器不去实例化`PlusResultT<A,B>`问题自然就解决了。然而编译器在解决重载时，必然是会去查看#1的定义的，如果这时候能够引发SFINAE，那么问题也可以解决。

但是，在替换`Array<typename PlusResultT<T1,T2>::Type>`的模板参数时，对`std::declval<T1>() + std::declval<T2>()`中的`+`，需要查找其定义，然而这个运算符的定义并不存在，因为不论是A还是B都没有定义`operator+`。于是，在实例化`PlusResultT<A,B>`时其定义中出现了错误，我们知道模板的定义不属于SFINAE的即时上下文。因此，这里没有引发SFINAE，而是一个编译错误。

为此，我们需要定义一个SFINAE友好的PlusResultT。

```c++
template<typename,typename,typename=std::void_t<>>
struct HasPlusT:std::false_type
{
};
template<typename T1,typename T2>
struct HasPlusT<T1,T2,std::void_t<decltype(std::declval<T1>()+std::declval<T2>())>>
:std::true_type
{
};

template<typename T1,typename T2,bool=HasPlusT<T1,T2>::value>
struct PlusResultT
{
	using Type=decltype(std::declval<T1>() + std::declval<T2>());
};
template<typename T1,typename T2>
struct PlusResultT<T1,T2,false>
{   
};
```

如果A和B不能相加，`HasPlusT<T1,T2>::value`的值就是false，于是选择部分特化的PlusResultT模板，而这个模板并没有Type这个成员，于是在解析`operator+()`重载时#1引发SFINAE，编译器就会选择#2那个`operator+()`。

那么，这里的区别在哪里呢？最初版本的`std::declval<T1>() + std::declval<T2>()`出现在`PlusResultT<T1,T2>`的定义也就是`{}`当中，如前所述，类的定义本身不属于即时上下文，而在定义里边的运算会触发对`operator+`的隐式实例化，由于找不到其定义，并且这个错误发生**非即时上下文**中，于是引发编译错误。而后面的`std::declval<T1>() + std::declval<T2>()`出现在**HasPlusT**的模板参数列表中，当在A和B的定义中都找不到`operator+`的定义时，并不会触发对`operator+`的隐含的实例化，而这个错误发生在即时上下文中，替换终止并引发SFINAE。

最后，作为一个通用的设计原则，只要是合理的参数，特性模板在实例化时永远不应该失败，也就是说不论如何都能找到一个合适的特性模板。所以必须有一个后备模板。

对于SFINAE友好的特性，通常的原则或者说方法是依次做两次检测：先检查操作是否合法，然后计算再其结果。这里的意思是，如果在第一步能检查出操作是不合法的，那么这时候就可以引发SFINAE，如果不分开来处理那么就有可能引发编译错误。正如我们上面看到的，首先用`HasPlust<>`来检测`PlusResultImpl<>`中的`operator+`是否有效而不是直接通过operator+来做+运算。

据此，我们还可以对19.3.1的ElementT重新做个定义

```C++
#include <vector>
#include <list>
#include <iostream>
#include <typeinfo>

template<typename,typename=std::void_t<>>
struct HasMemberT_value_type:std::false_type
{
};
template<typename T>
struct HasMemberT_value_type<T,std::void_t<typename T::value_type>>
:std::true_type
{
};

template<typename C,bool=HasMemberT_value_type<C>::value>
struct ElementT
{
    using Type=typename C::value_type;
};

template<typename C>
struct ElementT<C,false>
{
    using Type=C;
};

template<typename T>
void printElementType(T const& c)
{
    std::cout<<"container of "<<typeid(typename ElementT<T>::Type).name()<<" elements.\n";
}

int main()
{
    std::vector<bool> s;
    printElementType(s);
    std::list<int> l;
    printElementType(l);
    int arr[42];
    printElementType(arr);
    int va[]{1,2,3};
    printElementType(va);
}
```

运行结果是

```bash
container of b elements.
container of i elements.
container of A42_i elements.
container of A3_i elements.
```
注意这里只是示例一下所谓SFINAE友好的特性的使用，所以对于不存在value_type的容器比方说上面的arr和va，相应的也输出了些内容。
## 19.5 IsConvertibleT

```c++
#include <type_traits>
#include <utility>
#include <iostream>
template<typename FROM,typename TO>
struct IsConvertibleHelper
{
    private:
        static void aux(TO);
    	//！，原书上这里是有问题的，下同
        template<typename F,typename=decltype(aux(std::declval<F>()))> 
        static std::true_type test(void*);
        template<typename> //!
        static std::false_type test(...);
    public:
        using Type=decltype(test<FROM>(nullptr));
};

template<typename FROM,typename TO>
struct IsConvertibleT:IsConvertibleHelper<FROM,TO>::Type
{
};

template<typename FROM,typename TO>
using IsConvertible=typename IsConvertibleT<FROM,TO>::type; //!

template<typename FROM,typename TO>
constexpr bool isConvertible=IsConvertibleT<FROM,TO>::value;

int main()
{
    std::cout<<isConvertible<int,int><<std::endl;
    std::cout<<isConvertible<int,std::string><<std::endl;
    std::cout<<isConvertible<char const*,std::string><<std::endl;
    std::cout<<isConvertible<std::string,char const*><<std::endl;
}
```

首先注意下上面有注释的地方，原书上的代码是有问题的。

其次要注意`IsConvertibleT<>`继承的是`IsConvertibleHelper<>::Type`，这个Type要么是个true_type，要么是个false_type，具体是哪个取决于模板参数，所以`IsConvertibleT<>`要么是个true_type类型，要么是个false_type类型。

然后注意`IsConvertible=typename IsConvertibleT<FROM,TO>::type`，原书上是Type，这是不对的。true_type和false_type的type就是其本身。

最后是`isConvertible=IsConvertibleT<FROM,TO>::value`，这个value是个bool值，也就是true或者false。

此外，其它内容都在前面介绍过，就不废话了。

### Handling Special Cases

有三种情况IsConvertibleT是不能正确处理的

1. 转换成数组类型应该总是false的，然而，在上面代码中aux的参数TO如果是个数组类型，会退化成指针，因此对某些FROM类型会产生true
2. 同样由于退化，对于函数类型也是如此。
3. void转换成(const/volatile修饰的)void应该产生true，然而，由于aux并不支持void参数，甚至都通不过编译。实际上，函数参数都不能是void的。

不过我们可以增加一个模板作为主模板，原来的主模板修改成部分特化

```c++
template<typename FROM,typename TO,
	bool=IsVoidT<TO>::value || IsArrayT<TO>::value || IsFunctionT<TO>::value>
struct IsConvertibleHelper
{
    using Type=std::integral_constant<bool,IsVoidT<TO>::value && IsVoidT<FROM>::value>;
};

template<typename FROM,typename TO>
struct IsConvertibleHelper<FROM,TO,false>
{
//原来的IsConvertibleHelper的定义
};
```

这样，当TO是上面所说的三种情况时，就会选择主模板，然后，只有当FROM和TO都是void时，才会返回true，否则返回false，这也是符合常规理解的。

而如果只有FROM是void，原先的代码正确返回false，因为在解决函数重载时，模板参数中对aux参数的替换会引发SFINAE。**注意**是void不是`void*`。

另外要注意的是主模板的bool是个类型，不是变量名，这里是个省略了变量名的非类型参数，其缺省值是=后面那一串，integral_constant的模板参数bool也是个类型，不要理解成主模板参数有个叫bool的变量，然后这里用了。

标准库有一个相应的类型特性`std::is_convertible<>`。

最后说一句，void，数组，函数，这三种类型属于模板的边界条件，必须要特别加以注意的。