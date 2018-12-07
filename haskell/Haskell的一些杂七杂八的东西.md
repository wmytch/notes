# Haskell的一些杂七杂八的东西

[TOC]
## 随看随记的东西

### 普通函数、中缀函数(运算符)

#### 转化

```haskell
`普通函数` => 中缀函数
(中缀函数) => 普通函数
```

#### 优先级

- 普通函数优先级高于中缀函数

- 中缀函数优先级0-9，9最高

- 没有结合性的中缀函数不能连写，必须加括号，比如

  ```haskell
  x `elem` xs `elem` ys是不对的，必须加上括号：
  （x `elem` xs） `elem` ys
  或者
  x `elem` （xs `elem` ys）
  ```
- ::优先级最低

### 命名风格

- 构造函数，用CamelName
- 其他函数，用camelName
- 自定义类型，CamelName

### `x -> y -> z`

表示两个参数*x*，*y*，返回*z*，也就是最后一个`->`后面的那个标志符是返回值，前面的`->`只是把参数分隔开，如果只有一个参数，就不需要`->`了

### `=>`

表示一个约束，`A a => [a]`表示*a*具有*A*的性质。

### 数据申明 data 
*Type = MakeType arg1 arg2*
基本上类似于

```C++
class Type{
    Type MakeType(arg1,arg2);
}
```
而
*data Type = Type arg1 arg2*
也类似于

```C++
class Type{
    Type(arg1,arg2)
}
```

当然不能机械套用，只是说明一下类型是类型，函数是函数，构造函数是允许跟类型同名的，而C++里面这是必须的。

另外还可以

*data Type = arg1 :op arg2*，于是`:op`就是*Type*的构造运算符，可以这样用

```haskell
1.5 :op 2
(:op) 1.5 2
```

### 模式匹配

来看这段代码

```haskell
module Test where
data Position = MakePosition Double Double

distance :: Position -> Position -> Double
distance p1 p2 =
    case p1 of
        MakePosition x1 y1 ->
            case p2 of
                MakePosition x2 y2 -> sqrt((x1 - x2)^2 + (y1 - y2)^2)
                
pointA :: Position
pointA = MakePosition 0 0

pointB :: Position
pointB = MakePosition 3 4
```

显然，如前述，*`data Position = MakePosition Double Double`*定义了一个类型Position，以及这个类型的构造函数。姑且就先算是定义吧，毕竟这里的构造函数如果理解为返回了一个Position，其实也没什么问题。

接下来*`distance :: Position -> Position -> Double`*不如理解为一个约束，事实上，申明在任何一门语言当中都可以认为是个约束，规定了这个函数的参数个数和返回值以及它们各自的数据类型，而`->`也可以理解为***满足前一个条件则往下***。对*distance*做了一个约束之后，就可以开始定义模板了：显然*distance*有两个参数，*case p1 of  MakePosition x1 y1*判断*p1*满足是*Position*类型这么个条件，满足则下一个，也就是判断*p2*的类型，继续满足，于是计算*sqrt*。

接下来对*pointA*和*pointB*的处理就通常理解好了，不外乎申明了一个变量，并赋了值。

实际上，这段模式匹配以及所谓的绑定，在许多其他语言中是由编译器处理的。也就是说，判断实际参数的类型，并将其与名义参数绑定，是由编译器做的。这样一看，应该就不会对所谓的模式匹配不明觉厉了。

所以，必然有个普通样式的函数定义：

```haskell
distance :: Position -> Position -> Double
distance (MakePosition x1 y1) (MakePosition x2 y2) =
	sqrt((x1 - x2)^2 + (y1 - y2)^2)
```

以及一个略有些怪异的版本：

```haskell
distance p1 p2 =
	let MakePosition x1 y1 = p1
		MakePosition x2 y2 = p2
	in sqrt((x1 - x2)^2 + (y1 - y2)^2)
```

进而有

```haskell
distance p1@(MakePosition x1 y1) p2@(MakePosition x2 y2) =
	sqrt((x1 - x2)^2 + (y1 - y2)^2)
```

但是不管怎么定义，调用的时候总是用*distance p1 p2*的形式。

### 列表`[]`

```haskell
data [a] = a : [a] | []
```

定义了一个类型`[]`，这个类型通常称为列表，以及列表的构造函数`:`和`[]`，这个`[]`表示构造一个空表。

列表要这样用：

```haskell
lista = [1,2,3,4]
listb = 3 : lista
```



