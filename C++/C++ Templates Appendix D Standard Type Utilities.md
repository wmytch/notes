# Appendix D Standard Type Utilities

[TOC]

## D.1 Using Type Traits

一般来说，使用类型特性，需要包含头文件type_traits，然后就可以使用了：

- 产生类型的特性，可以

    ~~~c++
    typename std::trait<...>::type
    ~~~

    或者

    ```c++
    std::trait_t<...>   //从c++14开始
    ```

- 产生值的特性，可以

    ```c++
    std::trait<...>::value
    std::trait<...>()  //依模板参数表达式初始化一个值，类型与value的类型相同，或者说做了次隐含转换,
    std::trait_v<...>  //从c++17开始
    ```

    看例子

    ```c++
    #include <iostream>
    #include <type_traits>
    
    int main()
    {
        int i=42;
        std::add_const<int>::type c=i;  //c是int const
        std::add_const_t<int> c14=i;   	//从C++14开始
        static_assert(std::is_const<decltype(c)>::value,"c should be const");
    
        std::cout<<std::boolalpha;
        std::cout<<std::is_same<decltype(c),int const>::value<<std::endl; 	//true
        std::cout<<std::is_same_v<decltype(c),int const><<std::endl;		//C++17
        if(std::is_same<decltype(c),int const>{})	//隐含转换成bool
            std::cout<<"same\n";
    
    }
    ```

    