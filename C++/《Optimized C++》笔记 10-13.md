[TOC]
# Chapter 10 Optimize Data Structures

## Get to Know the Standard Library Containers

出于优化的目的，C++标准库容器众多特性中有一些是特别重要的：

- 插入和删除的大O性能保证
- 线性容器添加元素时的常量时间要求
- 容器要能很好的控制动态内存分配

虽然看起来STL容器都有类似的接口，但实际上内部语义往往不同，简单的互相替代对于性能的影响是需要实测的。此外，程序员需要很好地了解容器的细节，以便能够选择合适的容器来满足性能上的需求。

### Sequence Containers

线性容器包括`std::string`，`std::vector`，`std::deque`，`std::list`，`std::forward_list`等。

- 这些容器中的项按照插入的顺序保存
- 每一个容器都有一个前端和一个后端，或者说是有前后方向的
- 所有容器都有插入项的方法
- 除了`std::forward`之外，其它容器都有常量时间的把项push到尾端的成员函数
- 只有`std::deque`，`std::list`，`std::forward_list`能够高效的把项push到前端
- `std::string`，`std::vector`，`std::deque`中的项按照0到`size-1`编号，从而可以高效通过下标获取项
- `std::list`和`std::forward`没有下标操作符
- `std::string`，`std::vector`，`std::deque`建立于一个类数组的内部基干上，因此插入项时会有将原有项后移的操作
    - 除了在尾端之外的任意位置插入项都有`O(n)`的时间
    - 插入操作可能会使得内部数组重新分配空间，从而使得所有的迭代器和指针失效
- `std::list`和`std::forward_list`，只有在移除元素时才会使迭代器和指针失效
- `std::list`和`std::forward_list`可以拆分或者归并，并且此时可以保证迭代器不失效
- 向`std::list`或者`std::forward_list`中间插入项的操作可以是常量时间，前提是已经有迭代器指向了插入位置。实际上，找到插入位置的时间是与这两个容器的内部存储相关的，并不一定是线性的。

### Associative Containers

关联容器按照项的某种顺序关系来存储插入的项，而不是按照插入的顺序，或者说关联容器的插入操作隐含了排序操作，这是与线性容器的一个区别。所以，对于关联容器元素的访问时间是低于线性时间的。

- map把key和value分别存储，并且提供了高效的从key到value的映射

- set存储唯一并且有序的value，并且提供了高效的检测一个value是否存在的方法

- multimap和multiset分别与map和set的区别只是在于允许插入多个**比较时**为相等的项，注意这里指的是比较时为相等，而不是说相等

上述四种关联容器也称为有序关联容器，都需要定义一个顺序关系，比如`operator<()`，对于map是key的顺序，set则是对值本身的顺序。理论上这些容器的实现是可以不同的，但是实际上，它们都可能只是对同一个平衡二叉树数据结构的修饰(facade)
- 有序关联容器是通过平衡二叉树实现的，因此不需要对它们进行排序操作。
- 对容器进行迭代是按照容器本身的顺序关系生成的一个项的序列
- 插入或者移除项的平摊时间为$$O(log~2~n)$$

C++11引入了4种无序关联容器：`std::unordered_map`，`std::unordered_multimap`， `std::unordered_set`，`std::unordered_multiset`。

- 无序关联列表只需要一个相等关系
- 无序关联列表是通过哈希表来实现的
- 对无序关联列表进行迭代生成的列表是没有预定义顺序的
- 插入和移除项的操作平均时间为常量时间，最坏情况为`O(n)`

对于查找表来说，关联容器是很显然的选择。程序员当然也可以给一个序列容器定义一个顺序，对容器进行排序之后，也可以得到相当高效的$$O(log~2~n)$$的查找算法。

### Experimenting with the Standard Library Containers

经过大量的数据实测，作者发现大O性能并不能完全说明问题，比方说，同样的`O(1)`的算法，实测时间上某些容器可能比其它容器快几倍，而即便如此，用undorderd_map和map比较，同样是`O(1)`的查找，实测时间上前者要比后者快，但是并没有想象中那么大的差距，并且为此所花费的内存空间是很显著的。

大多数容器对插入都提供了多种实现，不同实现之间时间差距有可能相差甚远，原因未明。向一个容器插入数据包括两部分工作，一个是分配存储空间，一个是通过拷贝构造函数拷贝数据。分配存储空间对于指定数量的项来说耗费是固定的，但是拷贝数据就各有不同，很多时候拷贝构造函数的耗费决定了构建一个容器的耗费。不过stl的容器大体上都是一致的。

大多数容器对迭代也提供了多种实现，各种实现同样也存在原因未明的性能差距。不过有趣的是，不同容器的迭代耗费差距并没有想象的那么大。

## std::vector and std::string

### Performance Consequences of Reallocation

### Inserting and Deleting in std::vector

### Iterating in std::vector

### Sorting std::vector

### Lookup with std::vector

## std::deque

### Inserting and Deleting in std::deque

### Iterating in std::deque

### Sorting std::deque

### Lookup with std::deque

## std::list

### Inserting and Deleting in std::list

### Iterating in std::list

### Sorting std::list

### Lookup wih std::list

## std::forward_list

### Inserting and Deleting in std::forward_list

### Iterating in std::forward_list

### Sorting std::forward_list

### Lookup wih std::forward_list
## std::map and std::multimap

### Inserting and Deleting in std::map

### Iterating in std::map

### Sorting std::map

### Lookup wih std::map

## std::set and std::multiset

## std::unordered_map and std::unordered_multimap

### Inserting and Deleting in std::unordered_map

### Iterating in std::unordered_map

### Lookup wih std::unordered_map 

## Other Data Structures

## Summary

