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

## 以上是看一本中文书时的笔记，再次验证中文书就是不靠谱，一边看一边还得自己脑补，看得累，所以还是老老实实上官网看文档。暂时就不删了，如果跟将来有什么不同或者混淆的地方，以将来的说法为准，这里保留的目的是为了证明前面说的中文书就是不靠谱不是瞎说的。接下来要看的文档是 :

## [Learning Haskell](http://learn.hfm.io/index.html "by Gabriele Keller and Manuel M T Chakravarty") 

### First Step

#### Values 

似乎对应于其他语言的*字面量*(*literal*)，比方说，`5`(整型数值)，`”hello world!“`(字符串)，`3.141`(浮点数)。

#### 常用类型

- `Int` = {…, `-3`, `-2`, `-1`, `0`, `1`, `2`, `3`, …}
- `Float` = {…, `-1232.42342`, …, `1.0`, `3.141`, …}
- `Double` = {… , `-1232.42342`, …, `1.0`, `3.141`, …}
- `Char` = {…, `'a'`, `'A'`, `'b'`, `'B'`, …`'1'`, …, `'@'`, `'#'`, …}
- `String` = {`""`, `"a"`, `"b"`, …, `"Hi"` ,`"3423#"`, …}
- `Bool` = {`False`, `True`}

这个没什么好解释的

#### `::`

`1 :: Int`可以读作“`1` *的类型是* `Int`“,

#### 函数定义 

一个函数定义包括一个*head*和一个*body*以及一个`=`。比方说：![inc x = x + 1](intro_canned.png)

#### 嵌套调用

以上面函数为例，

> `inc (inc 5)`  ⇒  `inc (5 + 1)`  ⇒  `inc 6`  ⇒  `6 + 1`  ⇒  `7`

#### 命名规则

- 以小写字母或者`_`开头，这条不是强制而是惯例
- 只能包含字母，数字，`_`，`’`
- 不能重名，函数跟函数不能重名，变量跟变量不能重名，函数跟变量也不能重名

#### 函数是映射

函数是个映射，唔，这才是函数本来的奥义呀，比方说：

![](intro_incPic.png)

#### Type Signatures

一个函数的定义是这样的：

```haskell
inc :: Int -> Int     -- type signature
inc x = x + 1         -- function equation
```

类型签名是可选的，不过它的主要作用一个是给程序员看，另一个是帮助haskell系统检查类型错误，所以还是有必要的。

#### Multiple Arguments 及柯里化

```haskell
average :: Float -> Float -> Float
average a b  = (a + b) / 2.0
```
`Float -> Float -> Float`可以理解为`Float -> (Float -> Float)`，因为`->`是右结合的，意思就是`Float -> Float -> Float -> Float`可以理解为`Float -> (Float -> (Float -> Float))`，当然这里说的是函数签名，而不是调用过程，不要理解为会先计算最右边的那个表达式。

函数调用语句或者说表达式是左结合的，比方说

`average 3.0 4.0`

就等价于

`(average 3.0) 4.0`。

这里并没有矛盾，因为`(average 3.0)`实际返回了一个新的函数，比方说叫`average_with_3_0 `，其定义为：

```haskell
average_with_3_0 :: Float -> Float
average_with_3_0 x = (x+3.0)/2
```

可以从另一个角度来考虑，这里本质上就是:
$$
average:Float \to average' \\
average':Float \to Float
$$
$average$和$average’$分别是两个数学意义上的映射。或者

```haskell
average':Float -> Float
average:Float -> average'
```

如果不支持LaTeX的话。

这种每次绑定一个参数，然后返回一个新函数的处理称之为柯里化，haskell中所有的函数都是这样处理的，缺省情况下。

#### 中缀和前缀

- (中缀)就变成前缀：
```haskell
(+) 1 5
```
等价于 
```haskell
1 + 5
```
- \`前缀\`就变成中缀：
```haskell
6.9 `average` 7.25
```
等价于

```haskell
average 6.9 7.25
```

#### 二元运算符左结合性

```haskell
6.9  `average` 7.25 `average` 3.4
```

等价于

```haskell
(6.9  `average` 7.25) `average` 3.4
```

#### `=>`

只是在函数的类型签名中用到这个符号，表示对值的一个约束。比如：

```haskell
(+) :: Num a => a -> a -> a
```

表示`a`是一个`Num`类型的值

#### 常用*type classes*和重载函数

- Typeclass `Show`
  - 函数： `show :: Show a => a -> String` 。
  - 成员类型：几乎所有的预定义类型，除了函数类型。
- Typeclass `Eq`
  - 函数：`(==), (/=) :: Eq a => a -> a -> Bool` 
  - 成员类型： 几乎所有的预定义类型，除了函数类型。
- Typeclass `Ord`
  - 函数： `(<), (>), (<=), (>=) :: Ord a => a -> a-> Bool`
  - 成员类型： 几乎所有的预定义类型，除了函数类型。
  - 所有属于`Ord`的类型都已经属于`Eq`, 所以如果要对同一个值使用 `==` 或者 `<` ，这个值属于`Ord`就可以了。
- Typeclass `Num`
  - 函数：`(+), (-), (*) :: Num a => a -> a -> a`
  - 成员类型：`Float`, `Double`, `Int`, `Integer`
- Typeclass `Integral`
  - 函数：`div, mod :: Integral a => a -> a -> a`
  - 成员类型：`Int` (固定精度), `Integer` (可变精度)
- Typeclass `Fractional`
  - 函数： `(/) :: Fractional a => a -> a -> a`
  - 成员类型：`Float`, `Double`
- Typeclass `Floating`
  - 函数： `sin, cos, tan, exp, sqrt,… :: Floating a => a -> a`
  - 成员类型：`Float`, `Double`

总结一下就是

- 除了函数类型，其它所有的类型都属于Show，Eq，Ord，也就是可以转换成字符串，可以比较，可以排序。注意

  `show “hello，world”`的结果会是`“\”hello,world\””`。

- 只有Float、Double、Int、Integer可以进行`+, -, *`运算。

- 只有Int、Integer可以做`div, mod`运算。

- 只有Float、Double可以做`/`运算以及数学函数运算。

### Fundamentals

