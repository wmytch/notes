# [Go 笔记 反射](https://blog.golang.org/laws-of-reflection)

[TOC]



## 类型和接口

go是静态类型语言，所以

```go
type MyInt int

var i int
var j MyInt
```

i和j是不相容的。并且

```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
```

不论r的具体值是什么，其类型总是io.Reader。因为接口在go中也是静态的类型。尽管在运行时，一个接口变量保存的具体值的类型会变，但这个值总是要满足接口，也就是必须实现接口中声明的方法的。

一个接口类型变量保存一个(值，类型)二元组：赋给这个变量的具体值，这个值的类型描述符。更精确的说，值就是具体的底层数据，类型就是这个底层数据的类型

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

于是，r包含的(value,type)二元组就是(tty,*os.File)。我们知道os.File不止实现了Read方法，所以下面这个赋值语句

```go
var w io.Writer
w = r.(io.Writer)
```

是一个类型断言，判断在r中的值，注意不是r，是否实现了io.Writer，如果是，就把它赋给w。接下来，我们可以

```go
var empty interface{}
empty = w
```

于是，空接口empty就包含了相同的二元组(tty,*os.File)。

这里我们不需要类型断言，因为任意的值包括w必然实现了空接口。但是把一个值从Reader转移到Writer就必须用类型断言，因为Writer的方法不是Reader的子集。

另外，一个重要的事实是一个接口包含的二元组总是(value,concrete type)，而不会是(value,interface type)。一个接口并不持有接口值，换句话说，接口本身并没有值。

## 反射要知道的第一件事：从接口的值到反射对象的反射

从最基本的层次而言，反射只是一种机制，用来检查一个保存着一个接口对象内部的类型和值对。

我们从reflect包中的两个类型Type和Value，以及两个函数reflect.TypeOf和reflect.ValueOf开始。

通过reflect.Type和reflect.Value我们可以得到接口变量的内容，而这两个函数可以分别从一个接口变量获取到reflect.Type和reflect.Value。当然，从reflect.Value也很容易得到reflect.Type。

我们从Typeof开始

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```

运行结果是

```bash
type: float64
```

不要奇怪，这里传进去的是一个float64而不是一个接口，因为TypeOf的签名是这样的

```go
// TypeOf returns the reflection Type of the value in the interface{}.
func TypeOf(i interface{}) Type
```

我们已经知道，一个接口变量会保存其具体值的二元组(value,type)。当我们调用reflect.TypeOf(x)时，x就首先保存到一个空接口中，然后作为参数传入，reflect.TypeOf再解开这个接口并恢复其类型信息。

而reflect.ValueOf函数则会恢复其值。

```go
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x).String())
fmt.Println("value:", reflect.ValueOf(x))
```

结果是

```bash
value: <float64 Value>
value: 3.4
```

多打一行的原因是避免产生一些不必要的疑问。调用String的原因是fmt包会深入到reflect.Value的内部挖出具体的值3.14，而String不会。

Value有一个Type方法，返回一个reflect.Value的Type。

Value和Type都有一个Kind方法，返回一个常量，表示其保存数据项的是什么：Uint, Float64, Slice等等。

而Value的Int和Float等方法则可以让我们得到其内部保存的数据项的值。

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

运行结果是

```bash
type: float64
kind is float64: true
value: 3.4
```

为简单记，Value的getter和setter方法总是使用位数最长的类型，比方说所有的有符号整型都用int64。也就是说Value的Int方法返回一个int64，SetInt使用一个int64，所以必要时需要类型转换

```go
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())                                       // v.Uint returns a uint64.
```

如果上面最后不做类型转换，便会有个编译错误。

另外就是Kind表示的是一个反射对象的底层数据类型而不是在变量声明体现的静态类型。

```go
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind:", v.Kind())
```

运行结果是

```bash
type: main.MyInt
kind: int
```

v.Kind()仍然是reflect.Int，而不是MyInt，或者说，Kind不能区分MyInt和int，但Type可以。


## 反射要知道的第二件事：从反射对象到接口的值的反射

给定一个reflect.Value，使用Interface方法可以恢复一个接口的值，实际上，这个方法是把类型和值的信息打包进到一个接口，然后返回这个接口，注意这里说的值和类型都是reflect对象，而不是某个变量的值和类型，当然这个变量的值和类型的信息是分别包含在作为reflect对象的值和信息里边的。概念上这里是需要明晰的。

```go
// Interface returns v's value as an interface{}.
func (v Value) Interface() interface{}
```

比如

```go
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

实际上，像fmt.Println, fmt.Printf这些函数的参数是空接口，在fmt内部会解包，所以实际上只要这样就可以了

```go
fmt.Println(v.Interface())
```

空接口的值包含了类型信息，Printf可以获取到。

所以，Interface方法是ValueOf函数的反函数。

## 反射要知道的第三件事：要改变一个反射对象，其值必须是可设置的

先看个错误的例子

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```

错误信息是这样的

```bash
panic: reflect.Value.SetFloat using unaddressable value
```

这里的问题在于不是7.1这个值不可寻址，而是v不可设置。可设置性(Settability)是反射Value的属性，但不是所有的反射Value都有这个属性。

可以用CanSet方法来测试一个Value的可设置性

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

将会输出

```bash
settability of v: false
```

可设置性有一点像可寻址性，但严格些。 它是这样一个属性，表明一个反射对象能够修改其所指向的实际存储空间。可设置性取决于该反射对象是否持有原始数据项。

当我们说

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
```

时给reflect.ValueOf传递的是x的一个副本，因此作为ValueOf参数的接口值是根据x的这个副本创建的，而不是x本身。因此，如果`v.SetFloat(7.1)`是合法的，那么它修改的也只是这个副本，而不是x本身，为简便记就定为非法了，引入可设置性也是为了避免这种情况。想象一下C语言中函数参数的传递方式，当然go语言也是同样的道理。

所以我们需要这样

```go
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```

输出为

```bash
type of p: *float64
settability of p: false
```

p虽然是个指针，但也是不可设置，道理跟上面说的是一样的。另外一方面，我们也不是要修改p，而是修改p所指向的地方，也就是*p，如果很好的理解了C或者就是go本身的指针，这里也是很好理解的，所以我们需要使用Value的Elem方法来得到这个地方

```go
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```

注意，v不是指针变量，而是通过指针间接访问得到的x的一个别名，它是可设置的。因此

```go
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```

输出自然就是

```bashe
7.1
7.1
```

### 结构 Strut

前面所说的机制特别适用于对结构的操作

```go
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
p := reflect.ValueOf(&t)
s := p.Elem()
typeOfT := s.Type()
fmt.Println(typeOfT)
fmt.Println(p)
fmt.Println(p.Type())
fmt.Println(s)
fmt.Println(typeOfT)
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: (%s %s) %s = %v\n", i,
			typeOfT.Field(i).Name, typeOfT.Field(i).Type,
			f.Type(), f.Interface())
}
```

输出为

```bash
&{23 skidoo}
*main.T
{23 skidoo}
main.T
0: (A int) int = 23
1: (B string) string = skidoo
```

注意T的字段名都是大写的，这是有意的，因为只有大写首字母的标识符才是可设置的，当然我们知道这时候这样的标识符才是可导出的，这当然不是偶然或者说巧合。

这个例子我们可以得出这样的信息，当然具体的方法需要参看reflect包。

- typeOfT是一个Type对象，描述的是T这个结构本身，其Field方法返回其包含的域的信息，比方说Name，Type
- s是一个Value对象，其Field方法返回的是代表其包含的域的Value对象，所以可以通过Type()和Interface()方法分别获得这个域的类型和值。
- f是一个Value对象，其Type方法返回这个域的Type，其Interface方法返回这个域的Value。

最后，我们可以修改这个结构的域了，当然指的是一个实际的结构变量，而不是这个结构的定义，要记得一个结构定义是分配存储空间因此必然也是不可寻址的，当然也就无从修改了。

```go
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```

输出

```bash
t is now {77 Sunset Strip}
```

注意对不同类型的域要用不同Set方法，毕竟既然语言已经提供了反射机制，想骗过类型系统那是几乎不可能的了。

## 结论

没有了，就上面三条注意的事项，或者说要知道的事情，翻译成法则其实并不恰当。