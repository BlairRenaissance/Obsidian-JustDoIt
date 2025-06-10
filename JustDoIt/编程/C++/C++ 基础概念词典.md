==`&`==：在变量声明中，`&`表示引用。`int& y = x;` 在表达式中，`&`表示取地址。`int* y = &x;`

==`const`==：表示变量是常量，值不可修改。[[3.2 Const and Constexpr]]

==`static`==：表示变量的**链接性**和**存储期**。[[4.2 Static]]
- 在类中表示静态成员；
- 在函数内用于声明静态局部变量，使其在函数调用之间保持状态；
- 在函数外表示内部链接（只在当前翻译单元可见）。

==`extern`==： `C/C++` 语言中表明函数和全局变量作用范围（可见性）的关键字，该关键字告诉编译器，其声明的函数和变量具有**外部链接**，可以在本模块或其它模块中使用。

==`extern "C"`==：`extern "C"` 修饰的变量和函数是按照 `C` 语言方式编译和连接的。 假设某个函数的原型为：

```text
void foo( int x, int y );
```

该函数被 `C` 编译器编译后在符号库中的名字为 `_foo` ，而 `C++` 编译器则会产生像 `_foo_int_int` 之类的名字（不同编译器生成的名字(`mangled name`)可能不同，但都采用了相同的机制）。`_foo_int_int` 这样的名字包含了函数名、函数参数数量及类型信息，`C++` 就是靠这种机制实的函数重载。

编译：如果一个头文件中有 `extern "C"` ，则编译后会提供C风格的符号。
链接：如果 `C++` 调用一个 `C` 语言编写的 `.DLL` ，在包含 `.DLL` 的头文件或声明接口函数时，应加 `extern "C" {　}` 。

```c
/* c语言头文件：cExample.h */
#ifndef C_EXAMPLE_H
#define C_EXAMPLE_H
extern int add(int x,int y);
#endif

/* c语言实现文件：cExample.c */
#include "cExample.h"
int add( int x, int y )
{
    return x + y;
}
```

```c++
// c++实现文件，调用add：cppFile.cpp
extern "C"
{
    #include "cExample.h"
}
int main(int argc, char* argv[])
{
    add(2,3);
    return 0;
}
```

---
## auto, auto&, const auto&

**auto**
for(auto x : range)。这一种用法将会为range的每一种元素都创建一份拷贝，所以当不想改变range里面的元素而想拷贝时，使用这一种用法。这一种用法有两个特殊的情况：
1. 用于`vector<bool>`时，可能结果会出乎你的意料。
	当使用`for(auto x : vector<bool>)`时，得到的x是一个`__bit_reference proxy class`，而这个class在进行赋值操作符=时，是返回的引用，于是对x操作时，会改变`vector<bool>`本身的元素。这样的情况，最好直接使用`for(bool x : vector<bool>)`。
2. 不能用于含有std::unique_ptr等只有move语意的容器。
	auto是在进行拷贝，而std::unique_ptr没有，会有编译错误。

**auto&**
for(auto& x : range)。当想要修改range里面的元素时，使用auto&，但遇到`vector<bool>`时会失效。因为`vector<bool>`返回的是`__bit_reference proxy class`临时对象，而这个临时对象不能绑定在non-const l-value reference中。

**auto&&**
for(auto && x : range)，这就是两全其美的办法。当`vector<bool>`返回的是临时对象，即右值时，我们也能正确的使用右值引用捕获住。
1. 当被左值初始化时，auto&&是左值引用(l-value reference)
2. 当被右值初始化时，auto&&是右值引用(r-value reference)

**const auto&**
for(const auto& x : range)，**当只想要读取range里面的元素时，使用const auto&**。

**总结**
1. 当想要拷贝range的元素时，使用for(auto x : range)。
2. 当想要修改range的元素时，使用for(auto & x : range) or for(auto && x : range)。
3. 当想要只读range的元素时，使用for(const auto & x : range)。

---
## auto, auto*

auto* 会指定推断出的类型为指针。

当使用 `auto` 和 `auto*` 进行类型推断时，通常是：

1. 使用 `auto` 进行类型推断：
```
auto a = 10; // 推断为 int 类型 
auto b = 3.14; // 推断为 double 类型 
auto c = "Hello"; // 推断为 const char* 类型 
auto d = std::string("World"); // 推断为 std::string 类型 
auto e = {1, 2, 3}; // 推断为 std::initializer_list<int> 类型
```
在上面的例子中，`auto` 根据初始化表达式的类型进行推断，并将变量的类型设置为相应的类型。

2. 使用 `auto*` 进行指针类型推断：
```
int* ptr = new int(5); 
auto* p = ptr;  // 推断为 int* 类型  

const char* str = "Hello"; 
auto* s = str;  // 推断为 const char* 类型的指针  

std::vector<int>* vec = new std::vector<int>{1, 2, 3}; 
auto* v = vec;  // 推断为 std::vector<int>* 类型的指针
```
在上面的例子中，`auto*` 根据初始化表达式的类型推断指针的类型，并将指针指向相应的对象类型。

那么如果我写这么写呢：
```
 const char* str = "Hello"; 
 auto s = str; 
```
在这个例子中，`auto* s = str;`和`auto s = str;`的效果是一样的，因为`str`的类型是`const char*`，所以`auto`和`auto*`都会推断出`s`的类型为`const char*`。

然而，在一些其他情况下，`auto`和`auto*`的行为是有区别的。例如，当初始化表达式的类型是一个对象（而不是一个指针）时，`auto`会推断出对象的类型，而`auto*`会尝试推断出一个指针类型，这时编译器会报错。例如：
```
int i = 0; 
auto a = i;  // a的类型是int 
auto* b = i; // 编译错误，不能将int转换为int*
```

```
const int* ci = nullptr; 
auto a = ci;      // a的类型是const int* 
auto* b = ci;     // b的类型是const int* 
auto c = *ci;     // 编译错误，因为ci是一个空指针 
auto* d = *ci;    // 编译错误，ci是空指针且不能将int转换为int*
```

总的来说，当推断类型是指针时，使用`auto*`可以提高代码的可读性和明确性，减少潜在的错误。

---
