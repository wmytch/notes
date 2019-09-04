# C++ Templates Chapter 18 The Polymorphic Power of Templates

[TOC]



## 18.1 Dynamic Polymorphism

```c++
class GeoObj
{
  public:
  	virtual void draw() const=0;
  	virtual Coord center_of_gravity() const=0; //不用管Coord是啥，无关紧要
  	
  	virtual ~GeoObj()=default;
};

class Circle:public GeoObj
{
	public:
		virtual void draw() const override;
		virtual Coord center_of_gravity() const override;
};

class Line:public GeoObj
{
	public:
		virtual void draw() const override;
		virtual Coord center_of_gravity() const override;
};

void myDraw(GeoObj const& obj)
{
  obj.draw();
}

void distance(GeoObj const& x1,GeoObj const& x2)
{
  Coord c=x1.center_of_gravity()-x2.center_of_gravity(); //显然Coord重载了运算符-
  return c.abs();
}

void drawElems(std::vector<GeoObj*> const& elems)
{
  for(std::size_type i=0;i<elems.size();++i)
  {
    elems[i]->draw();
  }
}

int main()
{
  Line l;
  Circle c,c1,c2;
  
  myDraw(l);
  myDraw(c);
  
  distance(c1,c2);
  distance(l,c);
  
  std::vector<GeoObj*> coll;
  coll.push_back(&l);
  coll.push_back(&c);
  drawElems(coll);
}
```

其实这程序也没什么特别好说的，只不过要提醒的是传入接口的是基类指针或者对基类的引用，只有这样，才能在接口中正确的指向正确的对象，如果传入一个实例，如果基类没有实现虚函数，调用函数时就肯定会出错，如果实现了，调用虚函数时就会调用基类的实现，因为这时候实际上发生了一次类型转换并构造了一个临时的基类对象。所以，在使用虚函数实现多态时通过指针或者引用传递参数并不仅仅是为了高效，而更本质的是只有这样才能实现动态绑定。

另外，就是coll这个vector是异构的，也就是说可以存放不同的GeoObj对象(指针、引用)，可以通过动态绑定来实现各自的操作。

## 18.2 Static Polymorphism

```c++
class Circle
{
  public:
  	void draw() const;
  	Coord center_of_gravity() const;
};

class Line
{
  public:
  	void draw() const;
  	Coord center_of_gravity() const;
};

template<typename GeoObj>
void myDraw(GeoObj const& obj)
{
  obj.draw();
}

template<typename GeoObj1,typename GeoObj2>
Coord distance(GeoObj1 const& x1,GeoObj2 const& x2)
{
  Coord c=x1.center_of_gravaity()-x2.center_of_gravity();
}

template<typename GeoObj>
void drawElems(std::vector<GeoObj> const& elems)
{
  for(unsigned i=0;i<elems.size();++i)
  {
    elems[i].draw();
  }
}

int main()
{
  Line l;
  Circle c,c1,c3;
  
  myDraw(l);
  myDraw(c);
  
  distance(c1,c2);
  distance(l,c);
  
  //std::vector<GeoObj*> coll  //Error
  std::vector<Line> coll;   
  coll.push_back(l);
  drawElems(coll);     
  
}
```

可以跟前一节的实现做个比较。这里Line和Circle都是独立的类，并没有一个共同的基类，所以要么是`coll<Line>`，要么是`coll<Circle>`，或者说这个coll是同构的，里边必然是单一类型，当然用指针或者引用也是可以的，只不过由于drawElems实现的限制，coll里边不能是指针，这样也有更好的类型安全性。

当然，这里也可以给Line和Circle提供一个模板基类，不过那是另外一个问题了。

不管用哪种方式，Line和Circle都要各自实现一系列成员函数，不论是虚函数还是自己的成员函数。

但是对于不同的对象，使用模板可能会有好几个接口实现，而使用虚函数只需要一个。

## 18.3 Dynamic versus Static Polymophism

### Terminology

- 通过继承实现的多态称为绑定的和动态的
  - **绑定的**意味着多态接口是由共同基类的设计预先决定的，也被称为invasive和intrusive，也就是侵入的意思。
  - **动态**意味着接口的绑定是在运行时。
- 通过模板实现的多态称为非绑定的和静态的
  - **非绑定的**意味着接口的多态行为是没有预先决定的，也被称为noninvasive和nonintrusive。
  - **静态**意味着接口的绑定是在编译时。

### Strengths and Weaknesses

动态多态优点是

- 可以优雅的处理异构对象集合
- 可执行代码相对小，因为只需要一个多态函数，而模板可能会生成多个实例。
- 代码可以完全编译，因此不需要公布实现源码，而使用模板的应用可能需要获取模板库的源码。

静态多态的优点是

- 内置类型对象的集合很容易实现，也就是说，不需要为这些类型提供一个公共的基类
- 代码执行可能会更快些，因为不需要解析指针或者访问虚表
- 可以使用只公开了部分接口的具体类型

### Combining Both Forms

如前面所说，两种形式是可以混合使用的。

所谓奇异递归模板模式，也就是curiously recurring template pattern，即CRTP。

## 18.5 New Forms of Design Patterns

本来使用指针或者引用然后通过虚函数实现的操作的可以使用模板参数传入一个类型，这同样属于组合。以下关于Bridge模式的示例，不要在意返回值的细节，只是因为markdown不能画类图，又懒得另外找工具了。

```c++
class Implementation
{
  public:
  	virtual operationA()=0;
  	virtual operationB()=0;
  	virtual operationC()=0;
};

class Interface
{
  public:
  	Implementation *body;
  	
  	operationA()
    {
      body->operationA();
    }
  	
  	operationB()
    {
      body->operationB();
      body->operationC();
    }
};

class ImplementationA:public Implementation
{
	public:
		virtual operationA();
		virtual operationB();
		virtual operationC();
};

class ImplementationB:public Implementation
{
	public:
		virtual operationA();
		virtual operationB();
		virtual operationC();
};

ImplementationA implA;
ImplementationB implB;
Interface itf;  //示例，不要在意这样使用的细节
itf.body=&implA;
...
itf.body=&implB;
...
```

或者

```c++
template<typename Impl>
class Interface
{
  public:
  	Impl body;
  	
  	operationA()
    {
      body.operationA();
    }
  	
  	operationB()
    {
      body.operationB();
      body.operationC();
    }
  
};

class ImplementationA
{
	public:
		operationA();
		operationB();
		operationC();
};

class ImplementationB
{
	public:
		operationA();
		operationB();
		operationC();
};

Interface<ImplementationA> implA;
Interface<ImplementationB> implB;
```

显见的区别就是使用继承需要两个Implementation实例，一个Interface实例，而使用模板，实际上生成了两个Interface实例以及各自包含的一个ImplementationA实例和一个ImplementationB实例。

## 18.6 Generic Programming

泛型编程的典范自然是STL，STL主要包含容器和算法，算法和容器都是模板，关键在于算法不是容器的成员函数，两者只是通过迭代器，也就是各种iterator联系起来。