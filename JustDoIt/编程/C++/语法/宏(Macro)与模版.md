
## 宏

### 带参宏

错误写法：
```cpp
#define MUL(M_a, M_b) M_a * M_b ​ 

MUL(3, 4); // 3 * 4

MUL(1 + 2, 3 + 4); // 1 + 2 * 3 + 4 
//此时的运行结果则已经是1 + 6 + 4 = 11了
```

正确的写法应该怎么样呢？我想你已经想到了：添加括号，保证优先级

```cpp
#define MUL(M_a, M_b) (M_a) * (M_b)

MUL(1 + 2, 3 + 4)
//经过预处理，变为了 （1 + 2） * (3 + 4)
//这回终于没问题了
```

### 使用宏替换函数

一定要小心多出来的分号《；》。

### 宏展开

当每个宏出现在另一个宏的定义中时，它会展开，但当它间接出现在自己的定义中，则不会展开。

```cpp
#define A(x) 1 
#define B(x) A(x) 
#define C(x) A(x) + B(x) 

C(1)//1 + 1
```

```cpp
#define A(x) 3
#define B(x) A(x) + B(x) + C(x)
#define C(x) A(x) + B(x) 

C(1)//3 + 3 + B(1) + C(1)
```

可以看到，宏参数中的宏会优先被展开：

```cpp
#define A(x) 3 
#define B(x) A(x) + B(x) + C(x) 
#define C(x) A(x) + B(x) 

C(A(3))//3 + 3 + B(3) + C(3)
```

问题：？？？？
```
#define A(x) 1 
#define B(x) A(x) 
#define C(x) A(x) + B(x) 

B(2) = ??
```

### 宏参字符串化

在宏里面我们可以使用 # 把随后的宏参数字符串化, 也就是把宏参数加上双引号，变成一个字符串字面量。

```cpp
#define xstr(s) str(s) 
#define str(s) #s 
#define foo 4 

str(foo) // "foo" 
//可以看到，即使参数是一个宏名，单层调用并不会使得参数宏展开 

xstr(foo) // "4" 
//想要参数宏展开转为字符串，需要使用两层的宏嵌套 
//第一层宏主要是是将宏参数中的宏展开，其次再展开展开式中的宏，才能获得想要的结果
```

有时候想生成如下代码，他们的名字前缀可能为prefix_prefix_prefix_，然后后缀是序号。于是又想到了用宏，迅速写下了:

```cpp
#define PREFIX(type, x) type prefix_prefix_prefix_ x

PREFIX(int, 1)//int prefix_prefix_prefix_ 1
//诶不对，怎么连接这两个部分呢
```

你发现这样并不能连接程序中的Token，那么如何在宏中连接两个Token呢？使用## ,

```cpp
#define PREFIX(type, x) type prefix_prefix_prefix_ ## x

PREFIX(int, 1)//int prefix_prefix_prefix_1
```

在C预处理器中，`##` 是一种特殊的预处理运算符，用于在宏定义中连接两个标记(tokens)。它的作用是将前一个标记和后一个标记合并成一个标记。


## 模板

### 函数模板

```cpp
template <typename T>
T add(T a, T b){
    return a + b;
}
```

### 类模板

```javascript
template <typename T> 
class AddAdapter{
public:
    T add(T a, T b){
        return a + b;
    }
};

AddAdapter<int> int_ap;
AddAdapter<float> float_ap;
```

### 模板特化

模板特化（Template Specialization）是 C++ 中模板的一个重要概念，它允许你为特定的数据类型或情况提供定制的实现。模板特化允许你在一般模板的基础上，为特定的类型或条件提供特定的行为或实现，从而实现更加精细化的控制。下面来看个例子：
```javascript
template<typename T>
void func(){
    std::cout << "func1\n";
}

template<>
void func<int>(){
    std::cout << "func2\n";
}

template<int N>
void func(){
    std::cout << N << std::endl;
}

class A {};

func<int>();//func2
func<float>();//func1
func<char>();//func1
func<A>();//func1
func<114514>();//114514
```

有两种类型的模板特化：完全特化(Full Specialization)和部分特化(Partial Specialization)。

- 完全特化(Full Specialization)： 完全特化是为特定的模板参数提供精确的实现。这种特化适用于给定参数的特定类型。
```cpp
// 完全特化的例子：针对 int 类型的模板参数
template <>
void printValue<int>(int value) {
    std::cout << "This is an integer: " << value << std::endl;
}
```

- 部分特化或者叫偏特化(Partial Specialization)： 部分特化是在给定部分参数时为模板提供特定的实现。这种特化适用于特定条件下的不同情况。
```cpp
// 类模板偏特化
template <typename T, typename U>
class Temp {
public:
    Temp() {
        std::cout << "Normal version." << std::endl;
    }
};

template <typename T>
class Temp<T, int> {
public:
    Temp() {
        std::cout << "Partial version." << std::endl;
    }
};

Temp<int, int> v1;//Partial version.
Temp<int, float> v2;//Normal version.
```
