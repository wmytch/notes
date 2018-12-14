
# [Learning Haskell](http://learn.hfm.io/index.html "by Gabriele Keller and Manuel M T Chakravarty") 笔记
[TOC]
## First Step

### Values 

似乎对应于其他语言的*字面量*(*literal*)，比方说，`5`(整型数值)，`”hello world!“`(字符串)，`3.141`(浮点数)。

### 常用类型

- `Int` = {…, `-3`, `-2`, `-1`, `0`, `1`, `2`, `3`, …}
- `Float` = {…, `-1232.42342`, …, `1.0`, `3.141`, …}
- `Double` = {… , `-1232.42342`, …, `1.0`, `3.141`, …}
- `Char` = {…, `'a'`, `'A'`, `'b'`, `'B'`, …`'1'`, …, `'@'`, `'#'`, …}
- `String` = {`""`, `"a"`, `"b"`, …, `"Hi"` ,`"3423#"`, …}
- `Bool` = {`False`, `True`}

这个没什么好解释的

### `::`

`1 :: Int`可以读作“`1` *的类型是* `Int`“,

### 函数定义 

一个函数定义包括一个*head*和一个*body*以及一个`=`。比方说：![inc x = x + 1](intro_canned.png)

### 嵌套调用

以上面函数为例，

> `inc (inc 5)`  ⇒  `inc (5 + 1)`  ⇒  `inc 6`  ⇒  `6 + 1`  ⇒  `7`

### 命名规则

- 以小写字母或者`_`开头，这条不是强制而是惯例
- 只能包含字母，数字，`_`，`’`
- 不能重名，函数跟函数不能重名，变量跟变量不能重名，函数跟变量也不能重名

### 函数是映射

函数是个映射，唔，这才是函数本来的奥义呀，比方说：

![](intro_incPic.png)

### Type Signatures

一个函数的定义是这样的：

```haskell
inc :: Int -> Int     -- type signature
inc x = x + 1         -- function equation
```

类型签名是可选的，不过它的主要作用一个是给程序员看，另一个是帮助haskell系统检查类型错误，所以还是有必要的。

### Multiple Arguments 及柯里化

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

### 中缀和前缀

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

### 二元运算符左结合性

```haskell
6.9  `average` 7.25 `average` 3.4
```

等价于

```haskell
(6.9  `average` 7.25) `average` 3.4
```

### `=>`

只是在函数的类型签名中用到这个符号，表示对值的一个约束。比如：

```haskell
(+) :: Num a => a -> a -> a
```

表示`a`是一个`Num`类型的值

### 常用*type classes*和重载函数

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

- typeclass必要时可以组合使用，比如：

  ```go
  signum :: (Ord a, Num a) => a -> Int
  signum x = if x < 0 then -1 else if x == 0 then 0 else 1
  ```

  `a -> Int`改成`a -> a`也是可以的，不过这个函数要注意下面的问题：

  ```haskell
  *Test> signum (-1)
  -1
  *Test> signum (0-1)
  -1
  *Test> signum 0-1
  -1
  *Test> signum 2-5
  -4
  *Test> signum (2-5)
  -1
  *Test> signum -5
  
  <interactive>:46:1: error:
  ```

  这里主要是提醒注意signum是函数，`-`也是，而且`-`不但是二元减法运算符，也是一元取负运算符，并且他们仨的优先级相同，而函数调用是左结合的，这样就能理解上面的执行结果了。

- 句法上，Typeclass所在的位置必须是Typeclass，不能是具体的类型名字，比如上面的`(Ord a, Num a) `改成`(Ord a, Int a) `是不行的。

## Fundamentals

### module 

句法：

```haskell
module Simple
where
--module body
```

`where`放第一行也是可以的。module名字首字母必须**大写**。

### 分支

也就是*if .. then .. else ..*及其各种变形

```haskell
signum :: (Ord a, Num a) => a -> Int
signum x = if x < 0 then -1 else if x == 0 then 0 else 1

signum :: (Ord a, Num a) => a -> Int
signum x | x <  0  = -1
         | x == 0  = 0
         | x >  0  = 1
         
signum :: (Ord a, Num a) => a -> Int
signum x | x <  0     = -1
         | x == 0     = 0
         | otherwise  = 1
```

以上3个*signum*是等价的，也同样在`x<0`的时会有一些问题。

### Binders

所谓绑定，理解为固定赋值就好了，与赋值不同的是绑定之后就不能再变了，所以叫绑定而不是赋值。

可以把一个值绑定到一个标识符，这个通常叫常量了

```haskell
pi :: Floating a => a
pi = 3.141592653589793
```

这是所谓全局绑定，还有局部绑定

```haskell
circleArea' :: Floating a => a -> a
circleArea' diameter  = pi * radius * radius
  where
    radius = diameter / 2.0       -- local binding
```

这里circleArea’是个全局绑定的名字，而radius则只限于这个函数当中。

对于绑定本身没有什么特别的，只是要注意作用范围内，绑定不可更改。

### (形参)多态函数

看例子

```haskell
fst :: (a, b) -> a
fst (x, y)  = x

snd :: (a, b) -> b
snd (x, y)  = y
```

这里没有类型签名，因为，fst和snd都不对参数进行运算，所以不用管参数的类型是什么，a，b的类型甚至可以不一样。这种函数叫**(形参)多态函数**

### 元组

还是看例子

```haskell
type Point = (Int, Int)
origin :: Point
origin  = (0, 0)

moveRight :: Point -> Int -> Point
moveRight (x, y) distance  = (x + distance, y)

moveUp :: Point -> Int -> Point
moveUp (x, y) distance  = (x, y + distance)
```
注意一下*type*的定义方式，以及*origin*的绑定方式，也就是直接赋了一个值。

以及

```haskell
type Colour = String
type ColourPoint = (Int, Int, Colour)
origin :: Colour -> ColourPoint
origin colour  = (0, 0, colour)

move :: ColourPoint -> Int -> Int -> ColourPoint
move (x, y, colour) xDistance yDistance  
  = (x + xDistance, y + yDistance, colour)

distance :: ColourPoint -> ColourPoint -> Float
distance (x1, y1, colour1) (x2, y2, colour2) 
  = sqrt (fromIntegral (dx * dx + dy * dy))
  where
    dx = x2 - x1
    dy = y2 - y1
```
这里*origin*是个函数，而不是前面那个例子的常量或者说变量。*where*后面是局部绑定，不是*SQL*的条件，更不是*while*。

### error

*Prelude*包中有个*error*函数：

```haskell
error :: String -> a
```

那个*a*表示什么类型都有可能，因为不用对*a*进行运算，所以不需要知道其类型，也就不用类型签名了。

这个函数可以理解为抛出异常并打印一段提示：

```haskell
error "encountered a fatal error"  ⇒  ** Exception: encountered a fatal error
```



### 递归

计算

```haskell
natSum :: Num a => a -> a
natSum 0  = 0                    
natSum n  = n + natSum (n - 1) 
```

和

```haskell
natSum :: Num a => a -> a
natSum n = if n == 0 
              then 0 
              else n + natSum (n - 1)
```

是一样的。不过这两种定义，在n<0时是不会终止的，也就是说这是个*partial*函数。

所以需要这样定义

```haskell
natSum :: (Num a, Ord a) => a -> a
natSum 0              = 0
natSum n  | n > 0     = n + natSum (n - 1) 
          | otherwise = error "natSum: Input value too small!"
```

## List

```haskell
repeatN :: Int -> a -> [a]
repeatN 0 x  = []
repeatN n x  = x : repeatN (n - 1) x
```

这里递归生成了一个所有元素都相同的列表。

```haskell
suffixes :: String -> [String]
suffixes ""  = []
suffixes str = str : suffixes (tail str)
```

比方说：

> ```haskell
> suffixes "Hello"`  ⇒  `["Hello", "ello", "llo", "lo", "o"]
> ```

### *nil* 和 *cons*

*nil*：`[]`

*cons*: `:`

这两个运算符就可以构造任意的列表了。

比方说

```haskell
[x1, x2,… , xn] = (x1 : (x2 : ⋯ : (xn : [])⋯)
```

实际上，后面的`()`里边的递归形式的表示方式是列表的原始方式，`[]`表达方式只是一种便利。

所以我们可以这样表示两个函数

```haskell
head :: [a] -> a
head (x:xs) = x

tail :: [a] -> [a]
tail (x:xs) = xs
```

当然我们知道这两个函数是*partial functions*，因为没有处理空列表的情况。

### 映射：操作列表元素

```haskell
allSquares :: Num a => [a] -> [a]
allSquares []       = []
allSquares (x : xs) = x * x : allSquares xs
```

> ```haskell
> allSquares [x1, x2,… , xn]` = `[x1 * x1, x2 * x2,… , xn * xn]
> ```

```haskell
import Data.Char 

allToUpper :: String -> String
allToUpper []                 = []
allToUpper (chr : restString) = toUpper chr : allToUpper restString
```

> ```haskell
> allToUpper "can you hear me now?`  ⇒  `"CAN YOU HEAR ME NOW?"
> ```

要记得列表的`(x:xs)`这种表示方式。以及，*String*本身是个列表。

此外，最关键的是这种模式：

```haskell
recursiveFunction []       = []
recursiveFunction (x : xs) = doSomethingWith x : recursiveFunction xs
```

再看一个例子

```haskell
distancesFromPoint :: ColourPoint -> [ColourPoint] -> [Float]
distancesFromPoint point []
  = []
distancesFromPoint point (p : ps)
  = distance point p : distancesFromPoint point ps
```

计算一个点与一组点之间的距离。*ColourPoint*和*distance*的定义在前面。

### 过滤：移除列表元素

```haskell
import Data.Char

extractDigits :: String -> String
extractDigits []
  = []
extractDigits (chr : restString)
  | isDigit chr = chr : extractDigits restString
  | otherwise   =       extractDigits restString
```

> ```haskell
> extractDigits "das43 dffe 23 5 def45" => "4323545"
> ```

以及

```haskell
inRadius :: ColourPoint -> Float -> [ColourPoint] -> [ColourPoint]
inRadius point radius []
  = []
inRadius point radius (p : ps)
  | distance point p <= radius = p : inRadius point radius ps
  | otherwise                  =     inRadius point radius ps
```

相当于指定一个圆心为point半径为radius的圆，找出列表中所有位于圆内或者圆上的点。

### Reductions: 合并列表的元素

不确定标题该怎么翻译，意思就是通过对列表元素的一些组合或者合并操作实现缩减。

比如

```haskell
product :: Num a => [a] -> a
product []     = 1
product (x:xs) = x * product xs
```

当然这是递归，只不过与前面不同的是在空表是返回1，而不是0。

## Higher-order Functions
### map/filter

所谓*higer-order functions*就是那种用函数作为参数的函数，以及返回函数的函数，其实是很普通的概念。

```haskell
import Prelude hiding (map)

map :: (a -> b) -> [a] -> [b]
map f []       = []
map f (x : xs) = f x : map f xs
```

注意第一行的hiding (map)。

这里定义的map函数，有两个参数：*f* 和*(x:xs)*。从签名看，*f*代表一个函数*(a->b)*，*xs*是个元素为a类型的List，而结果则是一个元素类型为b的List。要记得一个List的*(x:xs)*这种表达方式，并不是说会自动把*xs*拆分成*(x:xs)*，而是*xs*本身就是*(x:xs’)*，体会一下区别。递归的基本部分我们忽略，只看递归部分，这里就是说map的第一个参数，函数*f*，处理第二个参数List *xs*的队首，*xs*的队尾则继续递归。更进一步，就是*xs*的队首是函数*f*的参数，一个a类型的元素，经过*f*处理后返回一个b类型的元素，并且与剩下的递归部分通过`:`运算生成一个b类型的List。

总之，map定义了一个递归函数的模板，将一个函数，也就是其第一个参数*f*，映射到待处理的数据，一个List(x:xs)上，最终返回经过*f*对数据处理的结果。

注意的是这里用了`:`来生成一个List，这也是map的签名规定了的，如果我们要求的结果不是一个List，而是其他的一个什么，比方说累加值，这个时候map的签名中返回值就不应该是[b]，而是b，或者a，当然b也可能就是a。

实际上，最后的返回值也不一定要跟a或者b有什么关系，当然此时这个函数就不叫map，而是一个别的什么名字了，比方说**filter**

```haskell
filter :: (a -> Bool) -> [a] -> [a]
filter p []  = []
filter p (x : xs)
  | p x       = x : filter p xs
  | otherwise = filter p xs
```
我们来看第一个例子
```haskell
allSquares :: Num a => [a] -> [a]
allSquares xs = map square xs
  where
    square x = x * x
```
于是*square*以*xs*的队首元素为参数并返回其平方，而*xs*的队尾继续递归。

第二个例子
```haskell
allToUpper :: String -> String
allToUpper string = map toUpper string
```
于是*toUpper*以string的第一个字符为参数并将其转换成大写，string剩下的字符继续递归。

再看一个复杂一些的
```haskell
distancesFromPoint :: ColourPoint -> [ColourPoint] -> [Float]
distancesFromPoint point points = map distanceP points
  where
    distanceP :: ColourPoint -> Float
    distanceP p = distance point p
```
这里就不说map的语义了。说复杂，并不是说map变得复杂，而是*distancesFromPoint*带了两个参数，第一个参数*point*并没有在map里边出现，而是出现在了*where*子句中*distanceP*的定义中，并且在其后的递归中保持不变，会变的只是map的第二个参数，那个**List**。

### 匿名函数

也叫闭包，或者别的一些什么名字，闭包是个比较通用的叫法了。

```haskell
functionName a1 a2 ⋯ an = body
```

与

```haskell
\a1 a2 ⋯ an -> body
```

是等价的，也就是说这个匿名函数，在参数列表前用`\`代替了函数名，同时把绑定符号`=`用`->`代替，其它的就没有了。当然匿名函数终究要符合闭包的使用条件。比方说

```haskell
nRadius :: ColourPoint -> Float -> [ColourPoint] -> [ColourPoint]
inRadius point radius points = filter (\p -> distance point p <= radius) points

llSquares :: Num a => [a] -> [a]
allSquares xs = map (\x -> x * x) xs
```

map和filter的定义如前，只不过这里使用了匿名函数。

### Point-free notation and partial application

使用map的函数定义并不检查参数列表，所以我们可以

```haskell
allSquares :: Num a => [a] -> [a]
allSquares = map (\x -> x * x)
```

考虑map的类型

```haskell
map :: (a -> b) -> [a] -> [b]
```

于是map就生成了一个新的函数

```haskell
allSquares = map (\x -> x * x)
```

怎么理解呢，初看起来这就像是个宏替换，不过在haskell中有个概念叫柯里化，也就是说，从map的类型来看，`map (\x -> x * x)`会得到一个新函数，这个函数以`[a]`为参数，而这里由`map (\x -> x * x)`得到的这个函数绑定到了*allSquares*上。其实我们在写C代码的时候也经常会使用宏替换来实现这样的功能，不过那个需要仔细调试，毕竟编译器在预编译的时候不会做任何检查，只是简单替换。

其他的所谓*high-order*函数都可以类似处理，也就是在绑定函数定义的时候不加参数列表，只要生成一个函数即可，因为这个参数列表也只是传递到新生成的这个函数。

### Reductions

上面看到的map和filter的共同点是都是只带两个参数，一个是函数，一个是数据。为方便记忆，再把他们的定义列一下

```haskell
map :: (a -> b) -> [a] -> [b]
map f []       = []
map f (x : xs) = f x : map f xs

filter :: (a -> Bool) -> [a] -> [a]
filter p []  = []
filter p (x : xs)
  | p x       = x : filter p xs
  | otherwise = filter p xs
```

可以看到区别是map要用f操作x，而filter则只是简单的取出或者丢弃x。然而，取出或者丢弃本身也是一种操作，因此，filter完全可以归结到map上。

我们看另外一种hige-order函数

```haskell
foldr :: (a -> b -> b) -> b -> [a] -> b
foldr op n []     = n
foldr op n (x:xs) = x `op` foldr n op xs
```

带了3个参数，一个是两个参数的函数，一个可以成为边界或者guard的数，另外一个是列表。

这里的语义就是取出队首元素，队尾进行递归，最终返回值和队首元素作为函数的操作数进行运算。

比方说

```haskell
minList :: [Int] -> Int
minList xs = foldr min maxBound xs
-- =x min foldr min maxBound xs'
-- foldr min maxBound [] = maxBound
-- xs=(x:xs'),下同

sum :: Num a => [a] -> a
sum xs = foldr (+) 0 xs
-- =x (+) foldr (+) 0 xs'
-- foldr (+) 0 [] = 0
```

当然，我们也可以定义point-free 风格的函数

```haskell
minList :: [Int] -> Int
minList = foldr min maxBound 

sum :: Num a => [a] -> a
sum = foldr (+) 0 
```

以及匿名函数版本的

```haskell
concat :: [[a]] -> [a]
concat = foldr (++) []

reverse :: [a] -> [a]
reverse = foldr (\x xs -> xs ++ [x]) []
```

实际上，我们还有左结合版本的fold

```haskell
foldl :: (b -> a -> b) -> b -> [a] -> b
foldl op acc []     = acc
foldl op acc (x:xs) = foldl op (acc `op` x) xs
```

就是先操作队首元素，然后操作结果和队尾一起进入递归，直到列表为空，这时候的返回值就是最后的返回值，其实这更像是递推，而不是递归了。

比如

```haskell
stringToInt :: String -> Int
stringToInt = foldl (\acc chr -> 10 * acc + digitToInt chr) 0

fastReverse :: [a] -> [a]
fastReverse = foldl (\accList x -> x : accList) []
```

所有以上这些函数定义，我们必须要注意的是**边界条件在high-order函数的基本部分检查，而在使用high-order函数进行定义的函数中不需要检查**。

最后我们看这么个例子

```haskell
sumOfSquareRoots :: (Ord a, Floating a) => [a] -> a
sumOfSquareRoots xs = sum (map sqrt (filter (> 0) xs))
```

其中的`(>0)`等价于`\x -> x>0`。于是

```haskell
sumOfSquareRoots xs = sum (x:xs')
					= sum (map sqrt (x':xs''))
					= sum (map sqrt (filter (\x -> x>0) (x'':xs''')))
					= sum (map sqrt (|x>0 = x : filter (\x -> x>0) xs'''
									 |otherwise = filter (\x -> x>0) xs'''))
					= sum (map sqrt (x':x''))
					= sum (x:xs')
					= a
```

###  **$** 和 **.**

```haskell
infixr 0 $
($) :: (a -> b) -> a -> b
f $ x = f x

(.) :: (b -> c) -> (a -> b) -> (a -> c)
(f . g) x = f (g x)
```

`$`等价于加了括号

```haskell
sumOfSquareRoots xs = sum $ map sqrt $ filter (> 0) xs
```

注意看括号加到哪里了。

所以如下定义是不对的

```haskell
sumOfSquareRoots = sum $ map sqrt $ filter (> 0) 
```

因为这时候

```haskell
sumOfSquareRoots xs = sum (map sqrt (filter (> 0))) xs
```

于是我们就需要`.`

```hasekll
sumOfSquareRoots = sum . map sqrt . filter (> 0) 
```

## Algebraic Data Types

### Enumeration Types

```haskell
data <TypeName> = <DataConstructorName1> 
				| ⋯ 
				| <DataConstructorNameN>
                deriving <Classes>
```

比如

```haskell
data Day
  = Sunday
  | Monday
  | Tuesday
  | Wednesday
  | Thursday
  | Friday
  | Saturday
  deriving (Enum)
```

*deriving (Enum)*表示新类型Day继承自标准类型Enum，也就意味着定义在Enum上的操作自动的也能应用到Day上。因此我们可以

```haskell
[Monday .. Friday]`   ⇒   `[Monday, Tuesday, Wednesday, Thursday, Friday]
```

通常来说deriving(Show)会是个好主意。

### Pattern matching and case expressions

模式匹配

```haskell
isWeekday :: Day -> Bool
isWeekday Sunday   = False
isWeekday Saturday = False
isWeekday _        = True
```

case表达式

```haskell
isWeekday :: Day -> Bool
isWeekday day = case day of
                  Sunday   -> False
                  Saturday -> False
                  _        -> True
```

要注意的是，如果case表达式中找不到匹配项，将会抛出异常，这跟C或者别的一些语言不同，所以最后一个`_`是必须的，除非你能包括所有的情况，不过现代语言在编译时通常也会提出警告或者就作为一个error。

### layout

除了上面的样式之外，还可以这样

```haskell
isWeekday :: Day -> Bool
isWeekday day = case day of
                { Sunday   -> False
                ; Saturday -> False
                ; _        -> True
                }
```

或者

```haskell
isWeekday :: Day -> Bool
isWeekday day = case day of {Sunday -> False ; Saturday -> False; _ -> True}
```

也就是说使用`{}`和`;`的话，就不需要顾虑缩进了，否则，就要通过缩进来表示代码的嵌套性。

把`;`放在句首是种很古怪的表达方式，所以我宁愿选择缩进。

### 为什么要使用构造函数

比方说我们有这样的定义

```haskell
type Point   = (Float, Float)
type Vector  = (Float, Float)
type Line    = (Point, Point)
type Colour  = (Int, Int, Int, Int) 
```

有这样的函数

```haskell
movePointN :: Float -> Vector -> Point -> Point
movePointN n (vx, vy) (x, y) = (n * vx + x, n * vy + y)
```

然后我们这样调用`movePointN 5 point vector`，显然haskell并不能检查出这里的错误。

所以我们需要构造函数

```haskell
data <Type> = <Constructor> <Type1> ⋯ <TypeN>
            deriving <Classes>
```

比如

```haskell
data Point = Point Float Float
           deriving (Show, Eq)
data Vector = Vector Float Float
            deriving (Show, Eq)
```

并且

```haskell
movePointN :: Float -> Vector -> Point -> Point
movePointN n (Vector vx vy) (Point x y) 
  = Point (n * vx + x) (n * vy + y)
```

于是

> ```haskell
> movePointN 5 (Vector 1 3) (Point 0 0)   ⇒   Point (5 * 1 + 0) (5 * 3 + 0)   ⇒   Point 5 15
> ```

当然，我们也是可以这样用的

```haskell
data Colour = Colour Int Int Int Int  -- red, green, blue, and opacity component
            deriving (Show, Eq)
red :: Colour
red = Colour 255 0 0 255
```

### 枚举和构造函数

用data自定义类型的时候，可以用枚举，也可以用构造函数，换句话说就是分别对应c++里enum和class/struct。为方便记忆，把上面的内容在下面复制了一份

```haskell
data Day
  = Sunday
  | Monday
  | Tuesday
  | Wednesday
  | Thursday
  | Friday
  | Saturday
  deriving (Enum)
  
data Vector = Vector Float Float
            deriving (Show, Eq)
```

使用上也是一样的，一个变量被申明为Day类型:`day :: Day`，那么这个变量就可以被赋值为Day中的文字量，如果一个变量被申明为Vector类型:`vec :: Vector`，那么这个变量就可以通过构造函数赋给一个值，如前面的red，当然在需要一个Vector的地方，可以直接把构造函数带入。

实际上还可以组合起来使用

```haskell
data PictureObject 
  = Path    [Point]                   Colour LineStyle 
  | Circle  Point   Float             Colour LineStyle FillStyle 
  | Ellipse Point   Float Float Float Colour LineStyle FillStyle 
  | Polygon [Point]                   Colour LineStyle FillStyle
  deriving (Show, Eq)
```

其中的Path，Circle，Ellipse，Polygon都是已定义的类型的构造函数。于是，PictureObject就相当于一个抽象类，并且实际上限定了其有哪些具体类。