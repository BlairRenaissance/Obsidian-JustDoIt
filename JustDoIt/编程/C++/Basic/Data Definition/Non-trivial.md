
最近在读《stl 源码剖析》中 copy 的实现时，文章说，如果一个容器内的元素类型拥有非平凡拷贝赋值函数时，应该不厌其烦的一次次调用拷贝赋值函数进行拷贝。但如果元素类型拥有平凡拷贝赋值函数，直接 memcpy 或者 memmove 对元素进行内存拷贝就行了。

那么问题来了，什么是平凡(trivial) / 非平凡(non-trivial) 构造函数，平凡 / 非平凡拷贝构造函数，平凡 / 非平凡拷贝赋值函数呢？


# 结论

简单来说，“平凡” 意味着这些特殊的成员函数用**很朴素的方式**完成它们的工作。而” 很朴素的方式 “这个说法对不同的函数有不同的意义。

1.  对默认构造函数和析构函数来说，“平凡” 意味着**什么也不做。**
2.  对拷贝构造函数和拷贝赋值函数来说，“平凡” 意味着**只做简单的内存拷贝**。

下面是具体的规则：

***规则一：*** 如果你为类显式定义了一个构造函数，那么它就是非平凡的构造函数，即使这个构造函数里什么都没做。因此，平凡的构造函数一定是你没写构造函数，编译器为你生成的那个函数。这个规则对其他几类函数同样适用。

可以想象，如果一个类同时拥有平凡构造函数，平凡拷贝构造函数，平凡拷贝赋值函数，平凡析构函数，那么这个类应该是非常简单的结构（这些函数咱肯定都没写）。

**_规则二：_** 规则一只是平凡的必要条件，除了不主动实现这些函数之外，“平凡构造函数”不允许在对象创建做任何隐式的初始化，“平凡拷贝构造函数”和 “平凡拷贝赋值函数” 不允许在对象拷贝的过程中做任何额外的操作。

规则二不太好理解，举两个例子：如果类有虚函数，那么相比没有虚函数的类，编译器为你生成的构造函数里需要进行额外的初始化（初始化虚指针等），因此这个类的构造函数不能认为是平凡的。同理，如果一个类有虚基类，这个类的构造函数也不是平凡的，因为这个类在构造的时候也需要进行一些隐式的额外操作来支持虚继承机制。

有虚函数和有虚基类这两种情况下，对象不能由简单的原始内存复制例程（如 memcpy）进行复制，而需要进行额外的操作才能正确地重新初始化副本中的隐藏指针。由于这个原因，上述两种类的拷贝构造函数和拷贝赋值操作符也都不是平凡的。

**_规则三：_**“平凡” 是递归定义的，意味着类的每个组件都要是平凡的。这里说的组件，包括基类，成员变量。因为我们知道，类构造时会先调用基类的构造函数，调用成员变量的构造函数，如果它们不是平凡的，那么自己的构造函数怎么会是平凡的？

上述结论的原文可在这里找到:

[What is a non-trivial constructor in C++?](https://link.zhihu.com/?target=https%3A//stackoverflow.com/questions/3899223/what-is-a-non-trivial-constructor-in-c)




# 实验部分

光听结论还是不够能让人相信，好在c++ stl提供了判断平凡性的类型萃取器，只要#include<type_traits>就可以用了。

**_情况一：不主动实现这三种特殊成员函数，并且类既没有虚函数 ，也没有虚基类_**

```cpp
#include <iostream>
#include <string>
#include <type_traits>

class base
{
private:
    int name;
public:
    void print_name()
    {
        std::cout << name << std::endl;
    }
};

class derived : public base
{
};

int main()
{
    // 判断是否为平凡构造函数
    std::cout << std::is_trivially_constructible<derived>::value << std::endl; 

    // 判断拷贝构造函数是否平凡
    std::cout << std::is_trivially_copyable<derived>::value << std::endl;

    // 判断拷贝赋值函数是否平凡
    std::cout << std::is_trivially_assignable<derived,const derived&>::value << std::endl;
}
```

输出结果为

```text
1
1
1
```

表明derived类的三种函数都是平凡的。

  

**_情况二：有一个显式定义的构造函数_**

```text
class derived : public base
{
    derived(){}
};
```

输出结果为

```text
0
1
1
```

表明即使加的构造函数什么也不做，derived类的构造函数也会变成非平凡的。

  

**_情况三: 有一个虚函数_**

```
class derived : public base
{
    virtual void do_nothing() {}
};
```

输出结果为

```text
0
0
0
```

有虚函数的类三种函数都不是平凡的

  

**_情况四：有虚基类_**

```text
class derived : virtual public base
{
};
```

输出结果为

```text
0
0
0
```

有虚基类的类三种函数都不是平凡的。

  

**_情况五：成员变量的对应函数是非平凡的_**

```text
class derived : public base
{
    string s;
};
```

输出结果为

```text
0
0
0
```

string类型的特殊成员函数是非平凡的，这导致derived类的三类特殊成员函数也是非平凡的。