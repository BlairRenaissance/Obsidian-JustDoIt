## 函数模板

```cpp
template <typename T>
T add(T a, T b){
    return a + b;
}
```

## 类模板

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

## 模板特化

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
