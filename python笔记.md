# Python笔记  

[TOC]



### 2.1.1. Argument Passing 

`python3 -c 'import sys;print(sys.argv[0]);print(sys.argv[1]),print(sys.argv[2])' 1 2`
输出为:   

```python 
-c
1
2  
```
可见`argv[0]:-c` `argv[1]:1` `argv[2]:2`....
总之从argv[1]开始才是传入并且由command或者module或者file处理的参数，而argv[0]则要看具体传递的形式，或者是“-c”本身或者是个文件名等等。    

### 2.2.1. Source Code Encoding 以及shell脚本
`# -*- coding: cp1252 -*-`  

或者

```  bash
#!/usr/bin/env python3
# -*- coding: cp1252 -*-  
```
第二个例子还说明了用python写shell脚本的方式  
### 3.1.1. Numbers 幂运算  

```python  
>>> 5 ** 2  # 5 squared
25
>>> 2 ** 7  # 2 to the power of 7
128  
```
### 3.1.1. Numbers 变量`_`
```python  
>>> tax = 12.5 / 100
>>> price = 100.50
>>> price *tax
12.5625
>>> price +_
113.0625
>>> round(_, 2)
113.06  
```
互动模式下，`_`代表上一条刚打印的表达式，这是个只读变量  
### 3.1.2. Strings escape以及raw string  
作为参数调用print时escape才起作用  

```python 
>>> '"Isn\'t," she said.'

'"Isn\'t," she said.'`

>>> print('"Isn\'t," she said.')

"Isn't," she said.

>>> s = 'First line.\nSecond line.'  # \n means newline
>>> s  # without print(), \n is included in the output

'First line.\nSecond line.'

>>> print(s)  # with print(), \n produces a new line

First line.
Second line.
```
raw string  

```python
>>> print(r'C:\some\name')  # note the r before the quote
C:\some\name
```
### 3.1.2. Strings 多行输入以及\的作用  
一个string分成多行输入时，\可以取消编辑时输入的换行符  

```python
>>> print("""\
... Usage: thingy [OPTIONS]
...      -h                        Display this usage message
...      -H hostname               Hostname to connect to
... "”")
```
```python
Usage: thingy [OPTIONS]
     -h                        Display this usage message
     -H hostname               Hostname to connect to
```
```python
>>> print("""
... Usage: thingy [OPTIONS]
...      -h                        Display this usage message
...      -H hostname               Hostname to connect to
... "”")
```
```python

Usage: thingy [OPTIONS]
     -h                        Display this usage message
     -H hostname               Hostname to connect to
```
```python
>>> print("""\
... Usage: thingy [OPTIONS]\
...      -h                        Display this usage message\
...      -H hostname               Hostname to connect to\
...  "””)
```
```python
Usage: thingy [OPTIONS]     -h                        Display this usage message     -H hostname               Hostname to connect to
```
### 3.1.2. Strings index以及slice  
```python
>>> text='python'
>>> text[3]
‘h'  
```
是可以的  

`‘python'[0]`  

是不允许的  

```python  
|P |y |t |h |o |n |
 0  1  2  3  4  5  6
-6 -5 -4 -3 -2 -1
```
所以  

```python  
text[0]==text[-6]==‘P’  
text[-1]=text[5]=’n’
```

因为-0==0，负索引从-1开始，也就是说最后一个字符索引是-1,第一个字符是-len(string)

slice的索引范围是个半开区间，也就是[b,e)这样的区间`text[-2:]==‘on’==text[4:]`  

```python  
>>> text[4:30]
'on'
>>> text[6:]
''
```
### 3.1.2. Strings 连接  

```python
>>> 'un'*3
'ununun'
>>> 3*'un'
'ununun'  
```
下面连接方式只适用于字符串文本，不能用于连接变量和表达式，要连接变量或者表达式时要用+号

```python
>>> 'py''thon'
'python'
>>> text=('py'
... 'thon')
>>> text
'python'
```
### 3.1.3. Lists append/replace/remove/clear  
list是mutable的，string是immutable的  

```python
>>> cubes =[1, 8, 27, 65, 125]  # something's wrong here
>>> cubes[3] = 64  # replace the wrong value
>>> cubes
[1, 8, 27, 64, 125]  
```
可以append  

```python
>>> cubes.append(216)  # add the cube of 6
>>> cubes.append(7 ** 3)  # and the cube of 7
>>> cubes
[1, 8, 27, 64, 125, 216, 343]  
```
*******
```python
>>> letters =['a', 'b', 'c', 'd', 'e', 'f', 'g']
>>> letters
['a', 'b', 'c', 'd', 'e', 'f', 'g']  
```
*******
可以replace一些值  

```python
>>> letters[2:5] =['C', 'D', 'E']
>>> letters
['a', 'b', 'C', 'D', 'E', 'f', 'g']  
```

可以remove一些值  

```python
>>> letters[2:5] =[]
>>> letters
['a', 'b', 'f', 'g']  
```

可以清空一个list  

```python
>>> letters[:] =[]
>>> letters
[]  
```
### 3.1.3. Lists 嵌套  

```python
>>> a =['a', 'b', 'c']
>>> n =[1, 2, 3]
>>> x =[a,n]
>>> x
[['a', 'b', 'c'], [1, 2, 3]]
>>> x[0]
['a', 'b', 'c']
>>> x[0][1]
'b'   
```
### 4.2. for Statements range的copy及改变  
```python
>>> for w in words:
...     print(w, len(w))
...  
cat 3
window 6
defenestrate 12
```
如果要在迭代中修改words，则需要：  

```python
>>> for w in words[:]:  # Loop over a slice copy of the entire list.
...     if len(w)>6:
...             words.insert(0, w)
... 
>>> words
['defenestrate', 'cat', 'window', 'defenestrate']
```
words[:]是words的一个copy，而不是words本身，所以在words中insert并不会改变这个copy，否则：  

```python
>>> for w in words:  # Loop over a slice copy of the entire list.
...     if len(w)>6:
...             words.insert(0, w)
```
会无限循环下去，不停的在前面插入‘defenestrate'
### 4.3. The range() Function  
```python
>>> print(range(10))
range(0, 10)
>>> x=range(10)
>>> print(x)
range(0, 10)
```
可见range返回一个可迭代的对象，但不是list本身：
可以用for循环迭代，也可以使用list()函数生成一个list  

```python
>>> list(range(5))
[0, 1, 2, 3, 4]
```
### 4.4. break and continue Statements, and else Clauses on Loops  

```python
>>> forn in range(2, 10):
…     for x in range(2,n):
…         if n % x == 0:
...             print(n, 'equals',x, '*',n//x)
...             break
...       else:
...         # loop fell through without finding a factor
...             print(n, 'is a prime number')
...
2 is a prime number
3 is a prime number
4 equals 2 * 2
5 is a prime number
6 equals 2 * 3
7 is a prime number
8 equals 2 * 4
9 equals 3 * 3
```
break和continue与c语言或者其他什么语言的语义是一样的，而这里的else会在内层循环正常迭代结束之后执行，但是break出来之后不会执行else  
### 4.6. Defining Functions None以及函数对象  
任何一个函数都会有一个返回值，即使没有return语句，也会返回一个None。  
函数是一个对象，所以可以  

```python
f=func
f(100)
```
也可以  

```python
f100=func(100)
f100
```
### 4.7.1. Default Argument Values 及in和is None  

```python
def ask_ok(prompt,retries=4,reminder='Please try again!'):
    while True:
        ok = input(prompt)
        if ok in('y', 'ye', 'yes'):
            return True
        if ok in ('n', 'no', 'nop', 'nope'):
            return False
        retries =retries - 1
        if retries < 0:
            raise ValueError('invalid user response')
        print(reminder)  

```
有三种调用方式  

1. `ask_ok('Do you really want to quit?')`
2. `ask_ok('OK to overwrite the file?', 2)  #retries=2`
3. `ask_ok('OK to overwrite the file?', 2, 'Come on, only yesor no!')`

in用来检查一个序列中是否包含某个值

缺省参数在函数定义的地方计算，只计算一次，而不是在调用时计算  

```python
>>> i=5
>>> def f(arg=i):
...     print(arg)
... 
>>> i=6
>>> f()
5  
```
在f的定义处，arg已经被初始化为5，之后与i已经没有关系了。
而  

```python
>>> def f(a, L=[]):
...     L.append(a)
...     return L
... 
>>> print(f(1))
[1]
>>> print(f(2))
[1, 2]
>>> print(f(3))
[1, 2, 3]  
```
在定义处初始化的这个list是共享的，这对于mutable的对象都是如此，比如dictionary和一些class。这也说明每次调用函数时，最初指向的都是同一个函数对象，而不是马上产生一个函数副本。  

如果不希望被共享，可以：  

```python
def f(a,L=None):
    if L is None:
        L =[]
    L.append(a)
    return  L  
```
另外还可以看到None是用is来比较，而不是==
### 4.7.2. Keyword Arguments  
keyword实参[^参数]指在调用函数时使用arg=value的形式传入的参数，而在函数定义形式参数列表中arg=value形式的参数arg称为可选参数，value称为缺省值  

~~~python
def parrot(voltage, state='a stiff', action='voom', type='Norwegian Blue'):
    print("-- This parrot wouldn't", action, end=' ')
    print("if you put", voltage, "volts through it.")
    print("-- Lovely plumage, the", type)
    print("-- It's", state, "!")  
~~~
这个函数接受一个required实参(voltage),三个可选实参(state,action,type)。keyword参数出现的顺序任意的，毕竟已经提供参数名字了，还要求顺序对程序员来说显得略多余。  
下面这样调用都是非法的：  

~~~python
parrot()                     # 缺少参数
parrot(voltage=5.0, 'dead')  # 在keyword参数之后不允许再出现非keyword参数
parrot(110, voltage=220)     # voltage重复了
parrot(actor='John Cleese')  # actor未知参数
~~~

[^参数]: 这里用参数还是实参犹豫了一阵子，虽然parameter和argument意义是不同的，前者指函数定义处的参数，也就是形式参数或者形参，后者指函数调用时的参数，也就是实际参数或者实参，不过在不会混淆的情况下还是会使用参数  
***
~~~python
def cheeseshop(kind, *arguments, **keywords):
    print("-- Do you have any", kind, "?")
    print("-- I'm sorry, we're all out of", kind)
    for arg in arguments:
        print(arg)
    print("-" * 40)
    for kw in keywords:
        print(kw, ":", keywords[kw])
~~~
复习一下`print("-" * 40)`这种字符串生成方式，以及注意一下arguments和keywords在函数中的使用方式。  
这个函数可以这样调用  

~~~python
cheeseshop("Limburger", "It's very runny, sir.",
           "It's really very, VERY runny, sir.",
           shopkeeper="Michael Palin",
           client="John Cleese",
           sketch="Cheese Shop Sketch")  
~~~
`*arguments`是个元组:  
`("It's very runny, sir.","It's really very, VERY runny, sir.")`
`**keywords"`则是个字典:  
`{"shopkeeper":"Michael Palin","client":"John Cleese",      "sketch":"Cheese Shop Sketch"}`  
***_注意并且理解参数列表中的顺序。_***

### 4.7.4. Unpacking Argument Lists `*`和`**`  

~~~python
>>> list(range(3, 6))            # normal call with separate arguments
[3, 4, 5]
>>> args = [3, 6]
>>> list(range(*args))            # call with arguments unpacked from a list
[3, 4, 5]  
~~~
args是个list,用元组也是可以的，但是args必须加上`*`号  

~~~python
>>> def parrot(voltage, state='a stiff', action='voom'):
...     print("-- This parrot wouldn't", action, end=' ')
...     print("if you put", voltage, "volts through it.", end=' ')
...     print("E's", state, "!")
...
>>> d = {"voltage": "four million", "state": "bleedin' demised", "action": "VOOM"}
>>> parrot(**d)
-- This parrot wouldn't VOOM if you put four million volts through it. E's bleedin' demised !  
~~~
同样的，d也必须加上`**`号
可以这么认为,`*`解开一个list或者元组,`**`解开一个dict.  
实际上`*`把list或者元组外面的括号去掉,使之成为一个序列.而`**`则要复杂一些,解开之外,还要给相应的形参赋值.

```python
>>> m=[1,2,[3,4]]
>>> u=*m
  File "<stdin>", line 1
SyntaxError: can't use starred expression here
>>> n=[[1,2]]
>>> >>> u=*n,
>>> u
([1, 2],)
>>> u=*n,1
>>> u
([1, 2], 1)
>>> u=**n,1
  File "<stdin>", line 1
    u=**n,1
       ^
SyntaxError: invalid syntax
>>> u=*(*n),1
  File "<stdin>", line 1
SyntaxError: can't use starred expression here
```
***注意这里的赋值事实上是个元组packing***
### 4.7.5. Lambda Expressions  

```python
>>> def make_incrementor(n):
...     return lambda x: x + n
...
>>> f = make_incrementor(42)
>>> f(0)
42
>>> f(1)
43
```
第4行生成了一个函数对象**f**:x->x+42[^函数]，接下来第5和第7行是对这个函数**f**的调用，而不是对函数make_incrementor的调用。  

[^函数]: 好吧，这个函数的写法是数学意义上的函数的写法，理解就行  

```python
>>> pairs = [(1, 'one'), (2, 'two'), (3, 'three'), (4, 'four')]
>>> pairs.sort(key=lambda pair: pair[1])
>>> pairs
[(4, 'four'), (1, 'one'), (3, 'three'), (2, 'two')]
```

lamda表达式作为参数传入了sort函数，需要注意的是必须使用keyword实参的形式传入。  
**注意**：这里只是说明了在函数的语境中lamda表达式的使用：作为返回值或者参数。  
不过我们可以看另外一个例子，可能更好理解一些。

```python
>>> def printPairs(key):
...     pairs = [(1, 'one'), (2, 'two'), (3, 'three'), (4, 'four')]
...     for tup in pairs:
...             print(key(tup))

>>> printPairs(key=lambda pair: pair[1])
one
two
three
four
>>> printPairs(key=lambda pair: pair[0])
1
2
3
4
```
### 4.7.6. Documentation Strings:`__doc__`  

```python
>>> def my_function():
...     """Do nothing, but document it.
...
...     No, really, it doesn't do anything.
...     """
...     pass
...
>>> print(my_function.__doc__)
Do nothing, but document it.

    No, really, it doesn't do anything.  

>>> def my_function():
...     """\
...     Do nothing, but document it.
...
...     No, really, it doesn't do anything.
...     """
...     pass
... 
>>> print(my_function.__doc__)
	Do nothing, but document it.
	
	No, really, it doesn't do anything.

```
第一个例子是惯常的函数文档的写法，第二个只是复习一下相关内容。

### 4.7.7. Function Annotations: `__annotations__`  
```python
>>> def f(ham: str, eggs: str = 'eggs') -> str:
...     print("Annotations:", f.__annotations__)
...     print("Arguments:", ham, eggs)
...     return ham + ' and ' + eggs
...
>>> f('spam')
Annotations: {'ham': <class 'str'>, 'return': <class 'str'>, 'eggs': <class 'str'>}
Arguments: spam eggs
'spam and eggs'

>>> def f(ham, eggs= 'eggs'):
...     print("Annotations:", f.__annotations__)
...     print("Arguments:", ham, eggs)
...     return ham + ' and ' + eggs
... 
>>> f('spam')
Annotations: {}
Arguments: spam eggs
'spam and eggs'
```
似乎不需要解释什么了,只是看一下一个完整的函数签名应该是什么样的.跟swift很类似,或者说这种签名形式是目前的潮流.
### 4.8. Intermezzo: Coding Style  
***`CamelCase` for classes and `lower_case_with_underscores` for functions and methods***
### 5.1. More on Lists  
所有list的方法都在这了：  

- `list.append(x)`  
* `list.extend(iterable)`
- `list.insert(i, x)`
- `list.remove(x)`
- `list.pop([i])`
- `list.clear()`
- `list.index(x[, start[, end]])`
- `list.count(x)`
- `list.sort(key=None, reverse=False)`
- `list.reverse()`
- `list.copy()`  

```python
>>> fruits = ['orange', 'apple', 'pear', 'banana', 'kiwi', 'apple', 'banana']
>>> fruits.count('apple')
2
>>> fruits.count('tangerine')
0
>>> fruits.index('banana')
3
>>> fruits.index('banana', 4)  # Find next banana starting a position 4
6
>>> fruits.reverse()
>>> fruits
['banana', 'apple', 'kiwi', 'banana', 'pear', 'apple', 'orange']
>>> fruits.append('grape')
>>> fruits
['banana', 'apple', 'kiwi', 'banana', 'pear', 'apple', 'orange', 'grape']
>>> fruits.sort()
>>> fruits
['apple', 'apple', 'banana', 'banana', 'grape', 'kiwi', 'orange', 'pear']
>>> fruits.pop()
'pear'
```
`insert`, `remove` 和 `sort`返回`None`
### 5.1.1. Using Lists as Stacks

```python
>>> stack = [3, 4, 5]
>>> stack.append(6)
>>> stack.append(7)
>>> stack
[3, 4, 5, 6, 7]
>>> stack.pop()
7
>>> stack
[3, 4, 5, 6]
>>> stack.pop()
6
>>> stack.pop()
5
>>> stack
[3, 4]
```
### 5.1.2. Using Lists as Queues
由于list存储方式的原因，直接使用list做为队列效率并不高，可以用 `collections.deque`:

```python
>>> from collections import deque
>>> queue = deque(["Eric", "John", "Michael"])
>>> queue.append("Terry")           # Terry arrives
>>> queue.append("Graham")          # Graham arrives
>>> queue.popleft()                 # The first to arrive now leaves
'Eric'
>>> queue.popleft()                 # The second to arrive now leaves
'John'
>>> queue                           # Remaining queue in order of arrival
deque(['Michael', 'Terry', 'Graham'])
```
### 5.1.3. List Comprehensions
不确定Comprehensions该怎么翻译，看例子：

```
[x**2 for x in range(10)]

[(x, y) for x in [1,2,3] for y in [3,1,4] if x != y]
```
一个list comprehension包括一对方括号`[]`,在方括号里边有一个表达式，比如上面两个例子里的`x**2`和`(x,y)`, 这个表达式后面跟着一个`for`语句, 然后是0个或者多个`for`或者`if`语句. 
上面两个例子分别等价于：

```python
>>> squares = []
>>> for x in range(10):
...     squares.append(x**2)
...
>>> squares
[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```
和

```python
>>> combs = []
>>> for x in [1,2,3]:
...     for y in [3,1,4]:
...         if x != y:
...             combs.append((x, y))
...
>>> combs
[(1, 3), (1, 4), (2, 3), (2, 1), (2, 4), (3, 1), (3, 4)]
```
对于第一个例子还有等价的做法，不过这是另外一个问题了:

```python
squares = list(map(lambda x: x**2, range(10)))
```

更多的例子：

```python
>>> vec = [-4, -2, 0, 2, 4]
>>> # create a new list with the values doubled

>>> [x*2 for x in vec]
[-8, -4, 0, 4, 8]

>>> # filter the list to exclude negative numbers
>>> [x for x in vec if x >= 0]
[0, 2, 4]

>>> # apply a function to all the elements
>>> [abs(x) for x in vec]
[4, 2, 0, 2, 4]

>>> # call a method on each element
>>> freshfruit = ['  banana', '  loganberry ', 'passion fruit  ']
>>> [weapon.strip() for weapon in freshfruit]
['banana', 'loganberry', 'passion fruit']

>>> # create a list of 2-tuples like (number, square)
>>> [(x, x**2) for x in range(6)]
[(0, 0), (1, 1), (2, 4), (3, 9), (4, 16), (5, 25)]

>>> # the tuple must be parenthesized, otherwise an error is raised
>>> [x, x**2 for x in range(6)]
  File "<stdin>", line 1, in <module>
    [x, x**2 for x in range(6)]
               ^
SyntaxError: invalid syntax

>>> # flatten a list using a listcomp with two 'for'
>>> vec = [[1,2,3], [4,5,6], [7,8,9]]
>>> [num for elem in vec for num in elem]
[1, 2, 3, 4, 5, 6, 7, 8, 9]

>>> from math import pi
>>> [str(round(pi, i)) for i in range(1, 6)]
['3.1', '3.14', '3.142', '3.1416', '3.14159']
```
***注意第21和第24行两个例子的区别.当然,`()`改成`[]`也是正确的,只不过生成的是个嵌套的list***

### 5.1.4. Nested List Comprehensions 矩阵转置

```python
>>> matrix = [
...     [1, 2, 3, 4],
...     [5, 6, 7, 8],
...     [9, 10, 11, 12],
... ]
```
我们要把这个矩阵转置，不过为了好理解，首先看这个例子:

```python
>>> [row[i] for row in matrix for i in range(4)]
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
```
比较下：

```python
>>> [[row[i] for row in matrix] for i in range(4)]
[[1, 5, 9], [2, 6, 10], [3, 7, 11], [4, 8, 12]
```
第一个例子等价于：

```python
>>> transposed = []
>>> for row in matrix:
...     for i in range(4):
...         transposed.append(row[i])
...
```
相应的第二个例子就等价于:

```python
>>> transposed = []
>>> for i in range(4):
...     # the following 3 lines implement the nested listcomp
...     transposed_row = []
...     for row in matrix:
...         transposed_row.append(row[i])
...     transposed.append(transposed_row)
...
>>> transposed
[[1, 5, 9], [2, 6, 10], [3, 7, 11], [4, 8, 12]]
```
也就是

```python
>>> transposed = []
>>> for i in range(4):
...     transposed.append([row[i] for row in matrix])
...
>>> transposed
[[1, 5, 9], [2, 6, 10], [3, 7, 11], [4, 8, 12]]
```
当然实现矩阵转置可以调用内置函数`zip`:

```python
>>> list(zip(*matrix))
[(1, 5, 9), (2, 6, 10), (3, 7, 11), (4, 8, 12)]
```
当然我们要注意的是list函数返回的list里的元素是元组，我们也可以猜测zip返回的实际上是个可迭代的对象，跟range一样。
### 5.2. The del statement 删除对象  

```python
>>> a=[1,2,3]
>>> del a
>>> a
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'a' is not defined
```
### 5.3. Tuples and Sequences 空元组和只有一个元素的元组 

```python
>>> empty = ()
>>> singleton = 'hello',    # <-- note trailing comma
>>> len(empty)
0
>>> empty
()
>>> len(singleton)
1
>>> singleton
('hello',)
```
单个元素的元组要注意后面跟着的逗号

```python
>>> ss=(1)
>>> ss
1
>>> ss[0]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'int' object is not subscriptable
>>> ss=(1,)
>>> ss
(1,)
>>> ss[0]
1
```
### 5.3. Tuples and Sequences 嵌套

```python
>>> t = 12345, 54321, 'hello!'
>>> u = t, (1, 2, 3, 4, 5)
>>> u
((12345, 54321, 'hello!'), (1, 2, 3, 4, 5))
>>> u = t, 1, 2, 3, 4, 5
>>> u
((12345, 54321, 'hello!'), 1, 2, 3, 4, 5)
>>> u=*t,1,2,3,4,5
>>> u
(12345, 54321, 'hello!', 1, 2, 3,4,5)
```
复习下`*`的作用,符合大多数语言中的语义,`*`就是个解析运算符,把外层括号去掉
### 5.3. Tuples and Sequences 元组的pack和unpack

```python
>>> t = 12345, 54321, 'hello!'
>>> >>> t
(12345, 54321, 'hello!')
>>> x, y, z = t
>>> >>> x
12345
>>> y
54321
>>> z
'hello!'
```
`t = 12345, 54321, 'hello!'`称为tuple packing，`x, y, z = t`则称为sequence unpacking.既然叫做sequence unpacking而不是tuple unpacking，当然是有道理的：

```python
>>> v=([3, 4, 5], [4, 3, 2])
>>> a,b=v
>>> a
[3, 4, 5]
>>> b
[4, 3, 2] 
```
而`x,y,z=1,2,3`这样所谓的多重赋值则实际上是tuple packing和sequence unpacking的组合。
### 5.3. Tuples and Sequences immutable和mutable  
tuple是不可修改的：

```python
>>> t = 12345, 54321, 'hello!'
>>> t[0]
12345
>>> t
(12345, 54321, 'hello!')
>>> t[0] = 88888
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
```
但是可以包含可修改的对象：

```python
>>> v = ([1, 2, 3], [3, 2, 1])
>>> v
([1, 2, 3], [3, 2, 1])
>>> v[0][1]
2
>>> v[0][1]=3
>>> v[0][1]
3
>>> v
([1, 3, 3], [3, 2, 1])
>>> v[0]=[2,3,4]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
```
比较：

```python
>>> v=([2,3,4],[3,2,1])
>>> for t in v:
...     for i in range(len(t)):
...             t[i]=t[i]+1
... 
>>> v
([3, 4, 5], [4, 3, 2])
```
### 5.3. Tuples and Sequences 元组和list的区别
元组是immutable，其元素通常是异构(heterogeneous)的，通常通过unpacking或者index来访问其元素。  
list是mutable的，其元素通常是同构的(homogeneous),其元素通常通过迭代访问。

### 5.4. Sets 创建

```python
>>> basket = {'apple', 'orange', 'apple', 'pear', 'orange', 'banana'}
>>> print(basket)   # show that duplicates have been removed,the result will be the same even if not using the print function             
{'orange', 'banana', 'pear', 'apple'}
>>> a = set('abracadabra')
>>> a
{'a', 'r', 'b', 'c', 'd'}
```
### 5.4. Sets 空set
要创建一个空set，必须用`set()`，而不是`{}`，`{}`将创建一个空dictionary
### 5.4. Sets 运算 `in`,`-`,`|`,`&`,`^`

```python
>>> basket = {'apple', 'orange', 'apple', 'pear', 'orange', 'banana'}
>>> basket                      
{'orange', 'banana', 'pear', 'apple'}
>>> 'orange' in basket                 
True
>>> 'crabgrass' in basket
False
>>> a = set('abracadabra')
>>> b = set('alacazam')
>>> a                                  # unique letters in a
{'a', 'r', 'b', 'c', 'd'}
>>> a - b                              # letters in a but not in b
{'r', 'd', 'b'}
>>> a | b                              # letters in a or b or both
{'a', 'c', 'r', 'd', 'b', 'm', 'z', 'l'}
>>> a & b                              # letters in both a and b
{'a', 'c'}
>>> a ^ b                              # letters in a or b but not both
{'r', 'd', 'b', 'm', 'z', 'l'}
```
***注意没有`+`***
### 5.4. Sets set comprehensions

```python
>>> a = {x for x in 'abracadabra' if x not in 'abc'}
>>> a
{'r', 'd'}  
```
### 5.5. Dictionaries key
key必须是不可修改的，比如字符串和数字，元组也可以，但是元组只能包含字符串和数字或者元组，不可直接或者间接包含可修改的元素。  
list是不能做key的。
### 5.5. Dictionaries 运算 `[]`,`del` `keys()`,`in`

```python
>>> tel = {'jack': 4098, 'sape': 4139}
>>> tel['guido'] = 4127
>>> tel
{'sape': 4139, 'guido': 4127, 'jack': 4098}
>>> tel['jack']
4098
>>> del tel['sape']
>>> tel['irv'] = 4127
>>> tel
{'guido': 4127, 'irv': 4127, 'jack': 4098}
>>> list(tel.keys())
['irv', 'guido', 'jack']
>>> sorted(tel.keys())
['guido', 'irv', 'jack']
>>> 'guido' in tel
True
>>> 'jack' not in tel
False
```
### 5.5. Dictionaries dict()

```python
>>> dict([('sape', 4139), ('guido', 4127), ('jack', 4098)])
{'sape': 4139, 'jack': 4098, 'guido': 4127}
```
或者

```python
>>> dict(sape=4139, guido=4127, jack=4098)
{'sape': 4139, 'jack': 4098, 'guido': 4127}
```
### 5.5. Dictionaries dict comprehensions

```python
>>> {x: x**2 for x in (2, 4, 6)}
{2: 4, 4: 16, 6: 36}
```
### 5.6. Looping Techniques items():同时获取dict的key和value

```python
>>> knights = {'gallahad': 'the pure', 'robin': 'the brave'}
>>> for k, v in knights.items():
...     print(k, v)
...
gallahad the pure
robin the brave
```
### 5.6. Looping Techniques  enumerate():同时获取sequence的索引和相应的值

```python
>>> for i, v in enumerate(['tic', 'tac', 'toe']):
...     print(i, v)
...
0 tic
1 tac
2 toe
```
### 5.6. Looping Techniques zip():同时在多个sequence上循环

```python
>>> questions = ['name', 'quest', 'favorite color']
>>> answers = ['lancelot', 'the holy grail', 'blue']
>>> for q, a in zip(questions, answers):
...     print('What is your {0}?  It is {1}.'.format(q, a))
...
What is your name?  It is lancelot.
What is your quest?  It is the holy grail.
What is your favorite color?  It is blue.
```
### 5.6. Looping Techniques reversed():反向访问一个sequence

```python
>>> for i in reversed(range(1, 10, 2)):
...     print(i)
...
9
7
5
3
1
```
### 5.6. Looping Techniques sorted():访问的时候排序

```python
>>> basket = ['apple', 'orange', 'apple', 'pear', 'orange', 'banana']
>>> for f in sorted(set(basket)):
...     print(f)
...
apple
banana
orange
pear
```
### 5.7. More on Conditions `in/not in` 和 `is/not is`
`in`或者`not in`检查一个序列中存在或者不存在一个值。  
`is`或者`not is`检查两个对象是否同一对象，应该说是两个object reference是否指向同一个object
### 5.7. More on Conditions chain

```python
>>> a=1
>>> b=2
>>> c=3
>>> a<b==c
False
>>> a<b<c
True
>>> a<b>c
False
>>> a>b<c
False
>>> a>b>c
False
```
分别等价于

```python
>>> a<b and b==c
False
>>> a<b and b<c
True
>>> a>b and b<c
False
>>> a>b and b>c
False
```
### 5.7. More on Conditions short-circuit以及表达式返回值

```python
>>> string1, string2, string3 = '', 'Trondheim', 'Hammer Dance'
>>> non_null = string1 or string2 or string3
>>> non_null
'Trondheim'
>>> non_null = string1 or string3 or string2
>>> non_null
'Hammer Dance'
>>> non_null=True or string1 or string3 or string2 
>>> non_null
True
>>> non_null=False or string1 or string3 or string2 
>>> non_null
'Hammer Dance'
```
这里可以看到所谓short-circuit，以及表达式返回的是比较停止的地方的表达式的值，而这个表达式的值可能就是一个变量的值，不必是个bool值或者转化成bool值
### 5.7. More on Conditions `=`和`==`
python是不允许在表达式内部进行赋值的。

```python
>>> if a=3 and a>4:
  File "<stdin>", line 1
    if a=3 and a>4:
        ^
SyntaxError: invalid syntax
```
这样对于C程序员来说，虽然某些情况下不太方便，但是至少避免了在需要`==`时打成了`=`这样的错误。
### 5.8. Comparing Sequences and Other Types

```python
(1, 2, 3)              < (1, 2, 4)
[1, 2, 3]              < [1, 2, 4]
'ABC' < 'C' < 'Pascal' < 'Python'
(1, 2, 3, 4)           < (1, 2, 4)
(1, 2)                 < (1, 2, -1)
(1, 2, 3)             == (1.0, 2.0, 3.0)
(1, 2, ('aa', 'ab'))   < (1, 2, ('abc', 'a'), 4)
```
不同类型的对象是可以比较的，但是这些对象必须提供相应的比较方法，比方说上面的`(1, 2, 3)             == (1.0, 2.0, 3.0)`,也可以理解为必须是相容的对象。比方说：

```python
>>> 1 < 'a'
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: '<' not supported between instances of 'int' and 'str'
>>> 1 < 1.2
True
```
### 6. Modules  `__name__`
一个module必须是一个文件,其文件名以`.py`做后缀.在一个module里面,`__name__`就代表了这个module的名字.  
比方说我们创建了这么一个module:`fibo.py`,其文件内容如下:  

```python
# Fibonacci numbers module

def fib(n):    # write Fibonacci series up to n
    a, b = 0, 1
    while a < n:
        print(a, end=' ')
        a, b = b, a+b
    print()

def fib2(n):   # return Fibonacci series up to n
    result = []
    a, b = 0, 1
    while a < n:
        result.append(a)
        a, b = b, a+b
    return result
```
我们可以这样使用:

```python
>>> import fibo
>>> fibo.fib(1000)
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
>>> fibo.fib2(100)
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
>>> fibo.__name__
'fibo'
```
当然也可以这样:

```python
>>> fib = fibo.fib
>>> fib(500)
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377
```
### 6.1. More on Modules `import`和`as`以及`reload`

```python
>>> from fibo import fib, fib2
>>> fib(500)
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377

>>> from fibo import *
>>> fib(500)
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377

>>> import fibo as fib
>>> fib.fib(500)
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377

>>> from fibo import fib as fibonacci
>>> fibonacci(500)
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377
```

```python
import importlib; 
importlib.reload(modulename)
```
在互动模式下,如果期间修改了某个之前已经import过的module,而又不打算退出环境重新进入,就可以调用这个方法.
### 6.1.1. Executing modules as scripts `"__main__"`
在上面的`fibo.py`文件的末尾加上:

```python
if __name__ == "__main__":
    import sys
    fib(int(sys.argv[1]))
```
就可以这样执行这个脚本了:

```python
$ python fibo.py 50
0 1 1 2 3 5 8 13 21 34
```
这样并不影响import
### 6.1.2. The Module Search Path
假设要import一个叫spam的module,首先会寻找是否存在一个叫spam的系统内建module,如果没有,则通过`sys.path`来查找是否存在这么一个文件.`sys.path`包含这些内容:

- 脚本所在目录,或者是当前目录`./`,或者是启动时带的目录名如`path/script.py`,那么path就会包含在`sys.path`里.
- PYTHONPATH 环境变量指定的目录,这个可以放在profile中
- 安装时的一些缺省的目录.

另外`sys.path`是不包含符号链接的,加入path前符号链接就已经被解析了.  
python程序运行过程中可以修改`sys.path`.  
另外,脚本所在目录是放在`sys.path`最前面的,在`sys.path`中的顺序决定了翻译器搜索的顺序.
### 6.1.3. “Compiled” Python files 
- -o 移除所有的assert
- -oo 移除所有的asset和`__doc__`字符串
- 一个`.pyc`文件并不比一个`.py`文件运行得更快,只是在load时更快些
- ` compileall`可以把一个目录下的所有module编译成`.pyc`文件

### 6.2. Standard Modules 提示符

```python
>>> import sys
>>> sys.ps1
'>>> '
>>> sys.ps2
'... '
>>> sys.ps1 = 'C> '
C> print('Yuck!')
Yuck!
C>
```
### 6.2. Standard Modules sys.path
如前所述,`sys.path`是翻译器寻找module的路径,是可以动态更改的,比如:

```python
>>> import sys
>>> sys.path.append('/ufs/guido/lib/python')
```
### 6.3. The dir() Function
`dir()`函数返回一个模块中所定义的所有名字或者说符号的名字,包括变量,模块,函数名等等.

```python
>>> import fibo, sys
>>> dir(fibo)
['__name__', 'fib', 'fib2']
>>> dir(sys)  
['__displayhook__', '__doc__', '__excepthook__', '__loader__', '__name__',
 '__package__', '__stderr__', '__stdin__', '__stdout__',
 '_clear_type_cache', '_current_frames', '_debugmallocstats', '_getframe',
 '_home', '_mercurial', '_xoptions', 'abiflags', 'api_version', 'argv',
 'base_exec_prefix', 'base_prefix', 'builtin_module_names', 'byteorder',
 'call_tracing', 'callstats', 'copyright', 'displayhook',
 'dont_write_bytecode', 'exc_info', 'excepthook', 'exec_prefix',
 'executable', 'exit', 'flags', 'float_info', 'float_repr_style',
 'getcheckinterval', 'getdefaultencoding', 'getdlopenflags',
 'getfilesystemencoding', 'getobjects', 'getprofile', 'getrecursionlimit',
 'getrefcount', 'getsizeof', 'getswitchinterval', 'gettotalrefcount',
 'gettrace', 'hash_info', 'hexversion', 'implementation', 'int_info',
 'intern', 'maxsize', 'maxunicode', 'meta_path', 'modules', 'path',
 'path_hooks', 'path_importer_cache', 'platform', 'prefix', 'ps1',
 'setcheckinterval', 'setdlopenflags', 'setprofile', 'setrecursionlimit',
 'setswitchinterval', 'settrace', 'stderr', 'stdin', 'stdout',
 'thread_info', 'version', 'version_info', 'warnoptions']
```
 如果调用`dir()`时没有提供参数,则返回到调用处为止之前所有的名字

```python
>>> a = [1, 2, 3, 4, 5]
>>> import fibo,sys
>>> fib = fibo.fib
>>> dir()
['__builtins__', '__name__', 'a', 'fib', 'fibo', 'sys']
```
### 6.4. Packages `__init__.py`
假设有这么一个package目录结构:

```python
sound/                          Top-level package
      __init__.py               Initialize the sound package
      formats/                  Subpackage for file format conversions
              __init__.py
              wavread.py
              wavwrite.py
              aiffread.py
              aiffwrite.py
              auread.py
              auwrite.py
              ...
      effects/                  Subpackage for sound effects
              __init__.py
              echo.py
              surround.py
              reverse.py
              ...
      filters/                  Subpackage for filters
              __init__.py
              equalizer.py
              vocoder.py
              karaoke.py
              ...
```
***注意当要导入这个package时,python是从`sys.path`所指路径中寻找module的,所以当翻译器抱怨说找不到模块的时候,记得把相关路径加入到`sys.path`中***  
`__init__.py`对于一个package是必须的,可以避免一些符号隐藏的问题,当然这是python搜索策略的原因.这个文件可以是一个空文件,也可以在其中做一些初始化的工作.  

### 6.4. Packages import以及from
当使用`from package import item`时,其中的`item`可以是子模块,或者子包,或者在这个包中定义的其他名字,比如类,函数,变量等.  
反过来,当使用`import item.subitem.subsubitem`时,除了最后一项外,其余的都必须是package的名字,最后一项可以是个module也可以是个package,但不能是类或者函数或者变量.  
我们可以看到使用上的区别:

```python
import sound.effects.echo
sound.effects.echo.echofilter(input, output, delay=0.7, atten=4)

from sound.effects import echo
echo.echofilter(input, output, delay=0.7, atten=4)

from sound.effects.echo import echofilter
echofilter(input, output, delay=0.7, atten=4)
```

### 6.4.1. Importing * From a Package `__all__`  
如果`__init__.py`里定义了一个叫做`__all__`的列表,那么当使用`from package import *`时只会导入该列表中出现的module.例如在`sound/effects/__init__.py`中如果包含了
`__all__ = ["echo", "surround", "reverse"]` ,那么`from sound.effects import *`将会导入这三个子module中的名字. 
如果没有定义`__all__`,则只导入在`__init__.py`中定义的名字,以及之前已经显式导入的module中的名字.比如:  

```python
import sound.effects.echo
import sound.effects.surround
from sound.effects import *
```
这里的意思就是,我们将来使用echo或者surround中的名字的时候,就不需要再加上`sound.effects.echo`或者`sound.effects.surround`之类的前缀了.换句话说,前面两个import等价于在`__all__`中的定义.  
最后,尽可能使用`from Package import specific_submodule`,而不是`from package import *`,只是要注意下重名的问题.
### 6.4.2. Intra-package References 相对/绝对导入
- 绝对导入  
比如`sound.filters.vocoder`模块需要使用`sound.effects`中的`echo`模块,则可以`from sound.effects import echo`.  
- 相对导入
比如在`surround`模块中可以

```python
from . import echo
from .. import formats
from ..filters import equalizer
```
*注意最后一句当中filters前面是`..`,而不是`...`或者其他的什么.*  
**最后,一个python应用的主模块,必须使用绝对导入来导入其他模块.**
### 6.4.2. Intra-package References 主模块导入
一个python应用的主模块,必须使用绝对导入来导入其他模块.因为,我们知道,一个module的名字对应了文件系统中的一个文件,而相对导入也就对应了通过相对路径查找文件的过程,而一个主模块的名字总是`__main__`,,如果在主模块中使用相对导入,就会从`__main__`出发形成一个相对路径,而`__main__`不能是个package.
例如这样的一个目录结构:

```
a.py
b.py
__main__/
    b.py
```
a.py内容是:

```python
from .b import bc

if __name__=='__main__':
    bc() 
```
两个b.py的内容都不影响这里的讨论,就不列出了.  
于是有:

```
$ python3 a.py
Traceback (most recent call last):
  File "a.py", line 1, in <module>
    from .b import bc
ModuleNotFoundError: No module named '__main__.b'; '__main__' is not a package
```
如果把`__main__`目录删掉,结果是一样的,因为python解释器已经禁止使用`__main__`作为一个package的名字.  
但是如果我们修改a.py的导入语句为:
`from b import bc`,那么:  

```
$ python3 a.py 
bc
```
也就是说,只有看到`.`或者`..`的时候解释器才会以当前模块的名字为出发点组织一个相对导入。这一点对任何module都是一样的，所以导入当前目录的module时，不要画蛇添足的使用`from . import module`。  
**因此在主模块中可以从当前目录导入一个模块而不使用绝对导入,记得不要加`.`。**
### 6.4 关于import的一个正确的实例
虽然文档上对于导入做了不少说明，然而实际使用的时候各种与import相关的错误仍然层出不穷，并且看文档也似乎解决不了问题。  
总的来说，python的import跟c/c++的include的处理是不大一样的，这里并不打算探讨其内在逻辑，只是给出一个正确但不承诺完备的例子：

- 首先，是一个python项目的目录结构：

```
pyproject/
    main.py
    package/
        __init__.py
        topmoudle1.py
        topmoudle2.py
        package1/
            __init__.py
            p1m1.py
            p1m2.py
            package11/
                __init__.py
                p11m1.py
        package2/
            __init__.py
            p2m1.py
```
- 其次，是各个文件的内容：
    - `main.py`：

    ```python
    #!/usr/bin/env python3
    import package.topmodule1
    from package.package1 import p1m1
    if __name__=='__main__':
        package.topmodule1.func1()
        p1m1.func11()
    ```
    注意这里使用的是绝对导入。下边其余文件里都使用了相对导入，这是必须而不是可选的。
    
    - `package/topmodule1.py`:

    ```python
    from .topmodule2 import func2 
    def func1():
        print(__name__)
        print("func1") 
        func2()
    ```
    - `package/topmodule2.py`:

    ```python3
    def func2():
        print(__name__)
        print("func2") 
    ```
    - `package/package1/p1m1.py`:

    ```python
    from .p1m2 import func12
    from .package11.p11m1 import func111
    from ..package2 import p2m1
    from .. import topmodule1
    def func11():
        print(__name__)
        print("func11")
        func12()
        func111()
        p2m1.func21()
        topmodule1.func1()
    ```
    - `package/package1/p1m2.py`:

    ```python3
    def func12():
        print(__name__)
        print("func12")
    ```
    - `package/package1/package11/p11m1.py`:

    ```python
    def func111():
        print(__name__)
        print("func111") 
    ```
    - `package/package2/p2m1.py`:

    ```python
    from ..package1 import package11
    from .. import topmodule1
    def func21():
        print(__name__)
        print("func21")
        package11.p11m1.func111()
        topmodule1.func1()
    ```
运行结果如下：

```
$ ./main.py 
package.topmodule1
func1
package.topmodule2
func2
package.package1.p1m1
func11
package.package1.p1m2
func12
package.package1.package11.p11m1
func111
package.package2.p2m1
func21
package.package1.package11.p11m1
func111
package.topmodule1
func1
package.topmodule2
func2
package.topmodule1
func1
package.topmodule2
func2
```
另外，使用`....`的导入方式是不存在的，可自行验证，这也是跟c/c++不同的地方，或者说`..`和`.`并不能完全视为路径。
### 7.2. Reading and Writing Files with和for

```python
>>> with open('workfile') as f:
...     read_data = f.read()
>>> f=open('workfile')
>>> for line in f:
...     print(line, end='')
```
### 9. Classes

- 普通成员(函数，数据)都是public的
- 函数都是virtual的
- 成员函数定义时，其第一个参数是显式提供的一个对象指针，而调用时这个参数是隐含的，也就是不需要显式提供这个指针对象的。
- 类本身就是对象，因此可以改名以及导入
- 内置类型可以作为基类
- 可以重载大多数内置运算符  

### 9.1. A Word About Names and Objects
一个对象可以有多个名字，这些名字可以理解为别名或者引用。

### 9.2. Python Scopes and Namespaces namespace
namespace是从名字到对象的映射。看看c++的namespace的定义或者申明方式:

```c++
namespace xxx{/*各种东西*/  } //一个名字空间：xxx
```
可以通过python的namespace的创建时机和生存期来体会其中的区别：

- python翻译器启动时就创建了一个内置名字的名字空间，并且其中的名字不会被移除。
- 当读入一个module时，这个module的全局名字空间就创建了，并且持续存在到翻译器退出。
- 翻译器直接调用的语句，不论是从脚本读入还是从交互界面输入，都被认为有个独立的名字空间。
- 函数被调用是创建了属于这个函数的local namespace，函数退出时被移除，包括异常时。递归调用的函数总是各自有各自的local namespace。

**总之，python的namespace自然而成。**
### 9.2. Python Scopes and Namespaces scope
scope是python程序中的一块文本，在该区域内一个名字空间可以直接访问，也就是不用加限定就可以访问。这是一个静态的描述。  
在执行的时候，一个名字空间可以直接访问的至少有三个scope，这三个scope是嵌套的，查找名字的时候从里到外，体会下这里名字空间与c/c++名字空间概念上的差异：  

- 最内层，包括局部名字
- 函数的非局部但也非全局的名字，以及module的全局的名字
- 最外层，系统内置的名字。

熟悉的变量隐藏这里也是适用的，虽然说法不一样。

9.2.1. Scopes and Namespaces Example

```python
def scope_test():
    def do_local():
        spam = "local spam"

    def do_nonlocal():
        nonlocal spam
        spam = "nonlocal spam"

    def do_global():
        global spam
        spam = "global spam"

    spam = "test spam"
    do_local()
    print("After local assignment:", spam)
    do_nonlocal()
    print("After nonlocal assignment:", spam)
    do_global()
    print("After global assignment:", spam)

scope_test()
print("In global scope:", spam)
```
运行结果：

```
After local assignment: test spam  #do_local里的赋值没有改变scope_test中spam的对象绑定
After nonlocal assignment: nonlocal spam #使用nonlocal将do_nonlocal中的spam绑定至外面的spam，也就是scope_test中的那个spam，要注意nonlocal的名字必须是显式存在的非local的名字，之前之后没有关系。
After global assignment: nonlocal spam #使用global表示这里的赋值操作指向global中的spam，而不是在scope_test中的这个spam，即便在global中还没有spam存在。
In global scope: global spam #之前并不存在的spam，也就是说在一个函数中定义了一个global的名字，并且在global中可以直接访问。
```
体会下nonlocal和global的使用。  
**总之，global必然是global的，而nonlocal则是非local的，也就是在当前的外面一层,但不能是global的。**

### 9.3.5. Class and Instance Variables

```python
class Dog:

    kind = 'canine'         # class variable shared by all instances

    def __init__(self, name):
        self.name = name    # instance variable unique to each instance

>>> d = Dog('Fido')
>>> e = Dog('Buddy')
>>> d.kind                  # shared by all dogs
'canine'
>>> e.kind                  # shared by all dogs
'canine'
>>> d.name                  # unique to d
'Fido'
>>> e.name                  # unique to e
'Buddy'
```
```python
class Dog:

    tricks = []             # mistaken use of a class variable

    def __init__(self, name):
        self.name = name

    def add_trick(self, trick):
        self.tricks.append(trick)

>>> d = Dog('Fido')
>>> e = Dog('Buddy')
>>> d.add_trick('roll over')
>>> e.add_trick('play dead')
>>> d.tricks                # unexpectedly shared by all dogs
['roll over', 'play dead']
```
```python
class Dog:

    def __init__(self, name):
        self.name = name
        self.tricks = []    # creates a new empty list for each dog

    def add_trick(self, trick):
        self.tricks.append(trick)

>>> d = Dog('Fido')
>>> e = Dog('Buddy')
>>> d.add_trick('roll over')
>>> e.add_trick('play dead')
>>> d.tricks
['roll over']
>>> e.tricks
['play dead']
```
### 9.7. Odds and Ends `struct`
如果需要一个类似于c的struct的数据结构，可以：

```python
class Employee:
    pass

john = Employee()  # Create an empty employee record

# Fill the fields of the record
john.name = 'John Doe'
john.dept = 'computer lab'
john.salary = 1000
```

### 9.7. Odds and Ends 多态
因为python中所有数据类型都是对象，所以实现多态是很自然的，或者说，通过鸭子类型来实现了多态。如果某个函数，可以传进去一个类，这个类本身是一个对象，其中定义了对某种数据类型的操作，在这个函数中我们就可以通过这个类的引用操作具体的某种数据类型。
### 9.8. Iterators `for` `iter` `next`

```python
for element in [1, 2, 3]:
    print(element)
for element in (1, 2, 3):
    print(element)
for key in {'one':1, 'two':2}:
    print(key)
for char in "123":
    print(char)
for line in open("myfile.txt"):
    print(line, end='')
```
`for`语句实际上调用了`iter()`函数，这个函数返回一个容器对象的迭代器对象，这个迭代器对象定义了一个方法：`__next()__`，于是就可以通过这个方法来访问容器对象的元素：

```python
>>> s = 'abc'
>>> it = iter(s)
>>> it
<iterator object at 0x00A1DB50>
>>> next(it)
'a'
>>> next(it)
'b'
>>> next(it)
'c'
>>> next(it)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
    next(it)
StopIteration
```
类似的我们可以对自定义对象定义迭代器：

```python
class Reverse:
    """Iterator for looping over a sequence backwards."""
    def __init__(self, data):
        self.data = data
        self.index = len(data)

    def __iter__(self):
        return self

    def __next__(self):
        if self.index == 0:
            raise StopIteration
        self.index = self.index - 1
        return self.data[self.index]
```
运行一下：

```python
>>> rev = Reverse('spam')
>>> for char in rev:
...     print(char)
...
m
a
p
s
```
### 9.8. Iterators `__func__()`与`func()`
在上面我们已经看到了`__next__()`与`next()`以及`__iter__()`与`iter()`之间各自的关系，具体的说就是，如果一个对象obj定义了`__func__(self)`这样的方法，那么就可以用`func(obj)`来调用这个obj的`__func__(self)`方法。这也是所谓**鸭子类型**的一个实例，同时也是前面提到的多态的一个例子，当然多态并不仅仅如此。
### 9.9. Generators
Generator是用来生成迭代器的：

```python
def reverse(data):
    for index in range(len(data)-1, -1, -1):
        yield data[index]
>>> for char in reverse('golf'):
...     print(char)
...
f
l
o
g
```
当然也可以像前面那样定义对象自己的迭代器。generator自动生成`__iter__()`和`__next__()`,而更主要的是能够自动保存局部变量和运行状态，比方说上面的index和data。
所以，如果我们需要创建一个迭代器，并不需要去修改原来的对象，只需要写这么一个函数就行了。
当然，如果对象本身就已经定义了`__iter__()`和`__next__()`，也就不需要generator了。
### 9.10. Generator Expressions
Generator Expression类似于list或者dict的comprehension

```python
>>> sum(i*i for i in range(10))                 # sum of squares
285

>>> xvec = [10, 20, 30]
>>> yvec = [7, 5, 3]
>>> sum(x*y for x,y in zip(xvec, yvec))         # dot product
260

>>> from math import pi, sin
>>> sine_table = {x: sin(x*pi/180) for x in range(0, 91)}

>>> unique_words = set(word  for line in page  for word in line.split())

>>> valedictorian = max((student.gpa, student.name) for student in graduates)

>>> data = 'golf'
>>> list(data[i] for i in range(len(data)-1, -1, -1))
['f', 'l', 'o', 'g']
```
