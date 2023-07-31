
# 内存分布

根据地址起点与类型长度取出对应的数据。

```cpp
union data{
	int n;
	char ch;
	short m;
};
```

要想理解union的输出结果，弄清成员之间究竟是如何相互影响的，就得了解各个成员在内存中的分布。以上面的 data 为例，各个成员在内存中的分布如下：  

![](http://c.biancheng.net/uploads/allimg/190118/152553G12-0.jpg)  

成员 n、ch、m 在内存中“对齐”到一头，对 ch 赋值修改的是前一个字节，对 m 赋值修改的是前两个字节，对 n 赋值修改的是全部字节。也就是说，ch、m 会影响到 n 的一部分数据，而 n 会影响到 ch、m 的全部数据，**会造成ch和m中数据损坏**。
  
上图是在绝大多数 PC 机上的内存分布情况，如果是 51 单片机，情况就会有所不同：  

![](http://c.biancheng.net/uploads/allimg/190118/1525532T7-1.jpg)  

为什么不同的机器会有不同的分布情况呢？这跟机器大端小端存储模式有关。

同时可以看到，上述内存空间的存储顺序和我们的定义顺序是不同的。因此，在许多情况下我们会将 `union` 中的成员变量定义为 `struct`，这是因为 `struct` 中的成员变量是按照定义顺序依次存储的，这样可以确保 `union` 中不同类型的成员变量的访问顺序是一致的，从而避免了 undefined behavior。



## 实验

***实验一：使用struct***

```cpp
#include <iostream>

union MyUnion {
    struct {
        int x;
        int y;
    } point;
    struct {
        int width;
        int height;
    } size;
};

int main() {
    MyUnion myUnion;
    myUnion.point.x = 10;
    myUnion.point.y = 20;
    std::cout << "Point: (" << myUnion.point.x << ", " << myUnion.point.y << ")" << std::endl;

    myUnion.size.width = 100;
    myUnion.size.height = 200;
    std::cout << "Size: (" << myUnion.size.width << ", " << myUnion.size.height << ")" << std::endl;

    std::cout << "Point: (" << myUnion.point.x << ", " << myUnion.point.y << ")" << std::endl;

    return 0;
}
```

输出结果为

```
Point: (10, 20)
Size: (100, 200)
Point: (100, 200)
```


***实验二：不使用struct***

```cpp
#include <iostream>

union MyUnion {
    int x;
    int y;
    int width;
    int height;
};

int main() {
    MyUnion myUnion;
    myUnion.x = 10;
    myUnion.y = 20;
    std::cout << "Point: (" << myUnion.x << ", " << myUnion.y << ")" << std::endl;

    myUnion.width = 100;
    myUnion.height = 200;
    std::cout << "Size: (" << myUnion.width << ", " << myUnion.height << ")" << std::endl;

    std::cout << "Point: (" << myUnion.x << ", " << myUnion.y << ")" << std::endl;

    return 0;
}
```

输出结果为

```
Point: (20, 20)
Size: (200, 200)
Point: (200, 200)
```



# 注意⚠️

### ***1. Union中使用非POD需要定义 non-trivial ctor&dtor***

如果 `union` 中的成员变量是非 `POD type` ([[POD]])，则需要定义相应的构造函数和析构函数，即non-trivial ctor&dtor ([[Non-trivial]]) 来确保正确的初始化和清理。因为 `union` 中的成员变量共享同一块内存空间，如果没有正确的构造函数和析构函数，可能会导致成员变量的数据被错误地初始化或清理，从而导致 undefined behavior。

```cpp
#include <iostream>
#include <string>

union MyUnion {
    int intValue;
    std::string stringValue;
};

int main() {
    MyUnion myUnion;
    myUnion.intValue = 42;
    std::cout << myUnion.intValue << std::endl;

    myUnion.stringValue = "Hello, world!";
    std::cout << myUnion.stringValue << std::endl;

    return 0;
}
```

XCode报错提示为

```
Call to implicitly-deleted default constructor of 'MyUnion'
```
太贴心了，看到我们用非POD的成员变量 `stringValue`，直接把默认构造函数隐式删除了，强迫我们写构造函数。如果我们没有定义 `MyUnion` 的构造函数和析构函数，`stringValue` 成员变量将会使用默认构造函数进行初始化，这可能会导致 `stringValue` 中的数据被初始化为一个未知的值。同样的，当 `MyUnion` 的作用域结束时，将会使用默认析构函数进行清理，这可能会导致 `stringValue` 中的数据被错误地清理，从而产生 undefined behavior。


正确定义为

```cpp
union MyUnion {
    int intValue;
    std::string stringValue;
    
    MyUnion() {}
    ~MyUnion() {}
};
```