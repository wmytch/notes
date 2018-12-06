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

### 数据申明data 
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

当然不能机械套用，只是说明一下类型是类型，函数是函数，构造函数是允许跟类型同名的，而C++里面是必须的。

另外还可以

*data Type = arg1 :op arg2*，于是`:op`就是*Type*的构造运算符，可以这样用

```haskell
1.5 :op 2
(:op) 1.5 2
```

