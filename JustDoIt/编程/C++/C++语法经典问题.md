
## `std::shared_ptr` 循环引用

`std::shared_ptr` 确实存在循环引用（也称为循环依赖）的风险。循环引用发生在两个或多个对象相互引用对方，形成一个环，这会导致引用计数永远不会变为零，从而导致内存泄漏。

```C++
#include <iostream>  
#include <memory>  
  
class B; // 前向声明  
  
class A {  
public:  
	std::shared_ptr<B> b_ptr;  
	~A() { std::cout << "A destroyed" << std::endl; }  
};  
  
class B {  
public:  
	std::shared_ptr<A> a_ptr;  
	~B() { std::cout << "B destroyed" << std::endl; }  
};  
  
int main() {  
	auto a = std::make_shared<A>();  
	auto b = std::make_shared<B>();  
	  
	a->b_ptr = b;  
	b->a_ptr = a;  
	  
	// 此时，a 和 b 互相引用，形成循环引用  
	return 0;  
}
```

在这个示例中，`A` 和 `B` 互相引用对方，形成了一个循环引用。即使在 `main` 函数结束时，`a` 和 `b` 超出了作用域，它们的引用计数也不会变为零，因此它们不会被销毁，导致内存泄漏。

为了避免循环引用，可以使用 `std::weak_ptr`。`std::weak_ptr` 是一种不增加引用计数的智能指针，它可以安全地打破循环引用。

```C++
class B {  
public:  
	std::weak_ptr<A> a_ptr; // 使用 std::weak_ptr 打破循环引用  
	~B() { std::cout << "B destroyed" << std::endl; }  
};
```

## 指针解引用

==一个 Object 对象的指针 Object* 解引用得到的是对象的引用 Object& ref。==

```C++
std::aligned_storage_t<sizeof(Object), alignof(Object)> objectStorage; 
Object& object() { return *reinterpret_cast<Object*>(&objectStorage); }
```

代码解释
1. **`std::aligned_storage_t<sizeof(Object), alignof(Object)> objectStorage;`**:
    - `objectStorage` 是一个未初始化的存储空间，大小和对齐方式与 `Object` 类型相同。
    
2. **`reinterpret_cast<Object*>(&objectStorage)`**:
    - `&objectStorage` 获取 `objectStorage` 的地址，类型为 `void*`。
    - `reinterpret_cast<Object*>(&objectStorage)` 将这个地址转换为 `Object*` 类型的指针。
    
3. **`*reinterpret_cast<Object*>(&objectStorage)`**:
    - 解引用 `Object*` 类型的指针，得到该指针所指向的对象。
    - 由于解引用操作返回的是对象的引用，所以 `*reinterpret_cast<Object*>(&objectStorage)` 的类型是 `Object&`。

简单示例

```C++
struct Object {  
	int data;  
	Object(int d) : data(d) {}  
};  
  
int main() {  
	Object obj(42); // 创建一个 Object 对象  
	Object* ptr = &obj; // 获取对象的指针  
	  
	// 解引用指针，得到对象的引用  
	Object& ref = *ptr;  
	  
	// 通过引用访问对象的成员  
	std::cout << "Object data: " << ref.data << std::endl;  
	  
	// 修改引用所指向的对象  
	ref.data = 100;  
	std::cout << "Modified object data: " << obj.data << std::endl;  
	  
	return 0;  
}
```

## `std::function`

`std::function` 是 C++11 引入的一个通用多态函数封装器。它可以存储、复制和调用任何可调用目标（如函数、lambda 表达式、绑定表达式或其他函数对象），其行为类似于函数指针，但`std::function` 可以存储状态（例如，lambda 表达式捕获的变量），这使得它比普通函数指针更强大。

`std::function<void*()>` 是一个特化的 `std::function` 类型，它表示一个可调用对象，该对象不接受任何参数并返回一个 `void*` 类型的指针。
- `void*`：表示返回类型是一个指向任意类型的指针。
- `()`：表示该函数不接受任何参数。

**注意事项**
- `std::function` 会引入一些额外的开销，因为它需要进行类型擦除和动态分配内存。因此，在性能关键的代码中，可能需要权衡其灵活性和性能。
- 使用lambda函数时要尤其注意，不能将函数内临时变量的地址传递到外部。除非这个临时变量是个static静态变量（不建议用）。


**使用普通函数**

```C++
#include <iostream> 
#include <functional>

void* MyFunction() {
	static int value = 42;
	return &value; 
}  

int main() {
	std::function<void*()> func = MyFunction;
	void* result = func();
	std::cout << "Result: " << *static_cast<int*>(result) << std::endl;
	return 0;
}
```

**使用 lambda 表达式**

```C++
#include <iostream> 
#include <functional>

int main() {
	std::function<void*()> func = []() -> void* {
		static int value = 42;
		return &value;
	};
	
	void* result = func();
	std::cout << "Result: " << *static_cast<int*>(result) << std::endl;
	return 0;
}
```

**使用 std::bind**

```C++
#include <iostream> 
#include <functional>  
void* MyFunction() {
	static int value = 42;
	return &value;
}  

int main() {
	auto boundFunc = std::bind(MyFunction);
	std::function<void*()> func = boundFunc;
	void* result = func();
	std::cout << "Result: " << *static_cast<int*>(result) << std::endl;
	return 0; 
}
```

**适用场景**

`std::function<void*()>` 可以用于以下场景：

1. **回调函数**：当你需要传递一个回调函数，并且该回调函数不接受参数但返回一个 `void*` 指针时，可以使用 `std::function<void*()>`。
2. **多态函数对象**：当你需要存储和调用不同类型的可调用对象（如函数、lambda 表达式、函数对象等），并且这些对象具有相同的签名时，可以使用 `std::function`。
3. **事件处理**：在事件驱动的编程中，可以使用 `std::function` 来存储和调用事件处理函数。


## 链式调用

``` C++
world->GetFogUniformSetter()-> SetGlobalFogVertexUniforms(vs_ubo).SetGlobalFogFragmentUniforms(fs_ubo);
```

链式调用（method chaining）的编程风格允许在一行代码中连续调用多个成员函数，从而使代码更加简洁和易读。

链式调用的关键在于每个成员函数返回一个对象的引用（通常是 `*this`），这样就可以在返回的对象上继续调用其他成员函数。

在 `FogUniformSetter` 类中，`SetGlobalFogVertexUniforms` 和 `SetGlobalFogFragmentUniforms` 函数都返回了 `*this`，即当前对象的引用。这使得可以在调用 `SetGlobalFogVertexUniforms` 后，继续调用 `SetGlobalFogFragmentUniforms`，如下所示：

``` C++
FogUniformSetter & SetGlobalFogVertexUniforms(ubo_class & ubo) {
    // ...
    return *this; // 返回当前对象的引用
}

FogUniformSetter & SetGlobalFogFragmentUniforms(ubo_class & ubo) {
    // ...
    return *this; // 返回当前对象的引用
}
```

## extern

**extern**
`extern` 是 `C/C++` 语言中表明函数和全局变量作用范围（可见性）的关键字，该关键字告诉编译器，其声明的函数和变量可以在本模块或其它模块中使用。

注意，语句 `extern int a;` 仅仅是对变量的**声明**，而并不是在定义变量 `a` ，声明变量并未为 `a` 分配内存空间。定义语句形式为 `int a;` ，变量 `a` 在所有模块中作为一种全局变量只能被定义一次，否则会出现连接错误。

在引用全局变量和函数之前必须要有声明或定义。通常，在模块的头文件中对本模块提供给其它模块引用的函数和全局变量以关键字 `extern` 声明。与 `extern` 对应的关键字是 `static` ，被它修饰的全局变量和函数只能在本模块中使用。因此，一个函数或变量只可能被本模块使用时，其不可能被 extern "C" 修饰。

**extern "C"**
被 `extern "C"` 修饰的变量和函数是按照 `C` 语言方式编译和连接的。 假设某个函数的原型为：

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

## &

在C++和类似的语言中，`&`符号的含义取决于它的上下文。

1. 在变量声明中，`&`表示引用。例如：
	```cpp
	int x = 10; 
	int& y = x; // y是x的引用
	```
	在这个例子中，`y`是`x`的引用，这意味着`y`和`x`指向的是同一块内存。如果你修改`y`的值，`x`的值也会改变，反之亦然。

2. 在表达式中，`&`表示取地址。例如：
	```cpp
	int x = 10; 
	int* y = &x; // y是x的地址
	```
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

## const 修饰函数

```cpp
const int& fun(int& a); //修饰返回值
int& fun(const int& a); //修饰形参
int& fun(int& a) const{} //const成员函数
```

**const返回值**
避免返回值被修改的情况。
需要指出的是，如果函数的返回类型是内置类型，比如 int char 等，修改返回值本身就是不合法的！所以 const 返回值是处理返回类型为用户定义类型的情况，多是修饰返回引用类型的情况下使用。

**const 修饰形参**
有的时候我们不希望改变函数参数的值，就要加上const关键字。

**const成员函数**
如果成员函数同时具有 const 和 non-const 两个版本的话， const 对象只能调用const成员函数， non-const 对象只能调用 non-const 成员函数。

## 左值/右值/通用引用

熟悉右值引用的话，就会知道它是通过`&&`来声明的。可能看见`&&`的第一反应就是右值引用。可惜，有些情况下竟然不是。例如`std::move`的源代码：
```
template <typename T>  
typename remove_reference<T>::type&& move(T&& t) noexcept  
{  
  return static_cast<typename remove_reference<T>::type&&>(t);  
}
```

这里`std::move`的参数`t`，就不是右值引用。若是右值引用的话，你怎么能传一个左值给它当参数呢？它的作用可是把左值转换成右值啊。所以，`t`必须能匹配左值。另一方面，这样的代码也不会出错：
```
Foo f("123");  
Foo&& r1 = std::move(std::move(f));
Foo&& r2 = std::move(Foo("456"));
```

也就是说，`t`也能够匹配右值。实际上，参数`t`就是一个**通用引用**。通用引用，可以使用左值表达式初始化，也可以使用右值表达式初始化。匹配左值表达式时，它表现的像一个左值引用；匹配右值表达式时，它表现的像一个右值引用。


**构成通用引用的条件**
出现`&&`的时候，如何判断它是右值引用，还是通用引用呢？通用引用有两个条件：
1. 形如`T&&`；
2. T的类型需要推导；
先看一些例子：
```
int lvalue = 3;  
auto&& r1 = lvalue;  
  
auto&& r2 = 4;  
  
int&& r3 = 5;  
auto&& r4 = r3; //等价于int& r4 = r3;  
  
std::vector<int> vect;  
auto&& r5 = vect[0];  
  
template<typename T>  
void func(T&& p);  
  
func(lvalue);  
func(10);
```
这里`r3`是右值引用，因为没有类型推导；相反，`r1, r2, r4, r5`和`p`都是通用引用。


**左值引用和右值引用**
```
Foo func()  
{  
  Foo f1("AAA");  
  return f1;  
}  
```
编译后运行，输出如下：
```
1 Foo(const char * s)    0x7ffc4a430330/0x21b0010/AAA  
2 Foo(Foo && other)    0x7ffc4a430330/0x21b0010/AAA --> 0x7ffc4a430380/0x21b0010/AAA
3 ~Foo()    0x7ffc4a430330/0/NULL  
```
- 输出第1行：构造func2中的局部变量f1；
- 输出第2行：构造临时对象。f1的data被move到临时对象（通过移动拷贝构造函数）；它被下面的r2引用；
- 输出第3行：析构f1。data已经被move，故为0/NULL；

```
int main()  
{ 
  // r1, r2, r3, r4 均为右值
  int && r1 = 3;  
  Foo && r2 = func();  
  Foo && r3 = Foo("BBB");
  r2.dump();  
  r3.dump(); 
  
  Foo f("CCC");  
  Foo && r4 = std::move(f);  
  
  r2 = Foo("DDD");
  
  Foo f3("EEE");  
  r3 = f3;  
  
  return 0;  
}
```
编译：
```
# g++ -fno-elide-constructors --std=c++11 test.cpp
```
运行，输出如下：
```
1 Foo(const char * s)    0x7ffc4a430390/0x21b0030/BBB  
2 0x7ffc4a430380/0x21b0010/AAA  
3 0x7ffc4a430390/0x21b0030/BBB

4 Foo(const char * s)    0x7ffc4a430360/0x21b0050/CCC  
5 Foo(const char * s)    0x7ffc4a4303a0/0x21b0070/DDD  
6 Foo& operator= (Foo && other) 0x7ffc4a4303a0/0x21b0070/DDD --> 0x7ffc4a430380/0x21b0070/DDD  
7 ~Foo()    0x7ffc4a4303a0/0/NULL

8 Foo(const char * s)    0x7ffc4a430350/0x21b0010/EEE  
9 Foo& operator= (const Foo & other) 0x7ffc4a430350/0x21b0010/EEE --> 0x7ffc4a430390/0x21b0030/EEE

10 ~Foo()    0x7ffc4a430350/0x21b0010/EEE  
11 ~Foo()    0x7ffc4a430360/0x21b0050/CCC  
12 ~Foo()    0x7ffc4a430390/0x21b0030/EEE  
13 ~Foo()    0x7ffc4a430380/0x21b0070/DDD
```
- 输出第1行：构造临时对象Foo(“BBB”)；它被r3引用；
- 输出第2行：r2.dump()；
- 输出第3行：r3.dump()；
- 输出第4行：构造main中的局部变量f；
- 输出第5行：构造临时对象Foo(“DDD”);
- 输出第6行：临时对象Foo(“DDD”)被赋值到r2（通过移动赋值函数）;
- 输出第7行：析构临时对象Foo(“DDD”)。data已经被move，故为0/NULL；
- 输出第8行：构造main中的局部变量f3；
- 输出第9行：f3被赋值到r3。通过赋值函数(非移动赋值)，因为f3是左值；
- 输出第10行：析构f3。它的data没有被move。
- 输出第11行：析构main中的局部变量f；
- 输出第12行：析构r3引用的临时对象(匿名变量)；
- 输出第13行：析构r2引用的临时对象(函数返回值)；

我们发现 r2 和 r3 可以出现在`=`的左边，特别是r3，这时候可以直接赋给它一个左值。r2 初始化完成后就一直持有着临时对象(函数返回值) `0x7ffc4a430380` ，被移动赋值后也只是DDD被传到了 `0x7ffc4a430380` 这个地址中，原地址随后被析构。直到最后函数return才释放了这个地址。这不是左值才有的特征吗？

是的！**右值引用类型的变量，r2和r3，就是左值！**。再说一遍，**右值引用类型的变量是左值！** 只不过：

- 这个左值的类型是**右值引用类型**；
- 这个左值只能**通过右值来初始化**；

右值引用和左值引用比起来，**不同的地方在于初始化那一刻：右值引用必须通过右值来初始化。** 初始化完成之后，就没有什么特殊的了。

相似之处：

- 右值引用类型的变量和左值引用类型的变量，本身都是左值；
- 右值引用类型的变量和左值引用类型的变量，都是引用另一个变量，本身不占内存；
- 对右值引用类型的变量和左值引用类型的变量进行&运算，得到的都是被引用对象的地址；
- 和返回局部变量的左值引用是非法的一样，返回局部变量的右值引用也是非法的；道理也相同；
- 和左值引用一样，父类的右值引用，也能够引用子类的右值；

不同之处：

- 初始化不同：右值引用类型的变量只能用右值进行初始化，而左值引用类型的变量必须用左值(这里指非const情形)。

## `#pragma once` & `#ifndef`

`#pragma once` 是一个非标准但被广泛支持的预处理符号， 其主要作用是防止文件重复引入问题。 在头文件中，可以定义 `#pragma once` 或者 `#ifndef`， 本文比较以下这两者区别。

```c
#pragma once

#ifndef __ARCH_ARM_SRC_ARTOSYN_AR_UART_H
#define __ARCH_ARM_SRC_ARTOSYN_AR_UART_H

#endif
```

**共同点：防止文件重复 include**

在以前的一些编译系统中，为了提高编译的效率，编译系统各自开发了 `#pragma once` 来防止文件重复 include。（非标准但被广泛支持！！）随着后来的开发，编译器层面对 `#ifndef` 进行了[优化](https://gcc.gnu.org/onlinedocs/gcc-2.95.3/cpp_1.html#SEC8), 目前的编译速度上两者并没有差别。　

**不同点**

1. `#pragma once` 不可用于 gcc 3.4 之前版本。
2. `#ifndef` 在于需要定义一个宏，如 `__ARCH_ARM_SRC_ARTOSYN_AR_UART_H`，一般这种宏以 `_前缀_文件名_H` 形式，如果文件名做了更改，那么需要一并更改这个宏。
3. 如果在不同的地方存在同名的文件，文件里面使用 `#ifndef` 定义的宏是一样的，链接编译的时候会收到一个警告， 使用 `#pragma once` 没有任何异常。

```c
.
├── hello.c
├── hello.h
├── main.c
└── Src
    ├── hello.c
    └── hello.h
```


## `std::map`, `std::unordered_map` 

`std::map` 和 `std::unordered_map` 都有各自的优点和缺点，选择哪一个取决于你的具体需求。

**`std::map`**

优点：

1. 元素总是按照特定的顺序排序。这是因为 `std::map` 使用红黑树来存储元素，红黑树是一种自平衡的二叉搜索树，它保证了元素的有序性。
2. 查找、插入和删除操作的时间复杂度都是 O(log n)。
3. 如果你需要频繁地查找元素，并且元素的顺序很重要，那么 `std::map` 是一个很好的选择。

缺点：

1. 由于红黑树的特性，`std::map` 的内存占用比 `std::unordered_map` 要大。
2. 如果你需要频繁地插入和删除元素，那么 `std::map` 可能不是一个好的选择，因为这可能会导致树的频繁调整，影响性能。

**`std::unordered_map`**

优点：

1. `std::unordered_map` 的内存占用比 `std::map` 要小。
2. 查找、插入和删除操作的时间复杂度都是平均 O(1)，在元素数量较多时，性能会比 `std::map` 更好。
3. 如果你需要频繁地插入和删除元素，并且对元素的顺序没有要求，那么 `std::unordered_map` 是一个很好的选择。

缺点：

1. 元素没有特定的顺序。如果你需要按照特定的顺序遍历元素，那么 `std::unordered_map` 可能不是一个好的选择。
2. 如果元素的数量很大，并且负载因子（元素数量/桶数量）很高，那么 `std::unordered_map` 的性能可能会下降，因为这可能导致大量的冲突和 ⚠️ **重新哈希**。
3. hash表的效率取决于hash算法和冲突解决方法（一般是拉链法，hash桶），以及数据分布，如果负载因子高，就会降低命中率，为了提高命中率，就需要扩容，重新hash，而重新hash是很慢的，相当于卡一下。而红黑树有更好的平均复杂度，所以如果数据量不是特别大，map是胜任的。

总的来说，如果你需要频繁地查找元素，并且元素的顺序很重要，那么 `std::map` 是一个很好的选择。如果你需要频繁地插入和删除元素，并且对元素的顺序没有要求，那么 `std::unordered_map` 是一个很好的选择。


## 结构化绑定

C++17 引入的结构化绑定（structured bindings）语法。这种语法允许 `for` 循环中直接解构一个容器中的元素，特别是当容器的元素是一个键值对时。

``` C++
for (const auto& [sid, plugin] : instances) {
    // 这里可以使用 _sid_ 和 _plugin_
}
```

- `instances` 是一个容器，通常是一个 `std::map`、`std::unordered_map` 或其他类似的容器。
- `const auto&` 表示我们希望以常量引用的方式访问容器中的元素，以避免不必要的拷贝。
- `[sid, plugin]` 是结构化绑定的部分，它将容器中的每个键值对解构为两个变量：`sid` 和 `plugin`。`sid` 将会是键，`plugin` 将会是值。

示例
```C++
#include <iostream>
#include <map>
#include <string>
  
int main() {
    std::map<int, std::string> instances = {
        {1, "PluginA"},
        {2, "PluginB"},
        {3, "PluginC"}
    };

    for (const auto& [sid, plugin] : instances) {
        std::cout << "SID: " << sid << ", Plugin: " << plugin << std::endl;
    }
    return 0;
}
```


## ## 使用 emplace/try_emplace/emplace_back

`emplace` 系列接口可以实现原地对象构造，消除掉拷贝构造、移动构造等操作。

反例：
```C++
std::unordered_map<std::string, Demo> demo_group;
demo_group.insert(std::make_pair(key, Demo(value, i)));  // 案例一
demo_group.emplace(std::make_pair(key, Demo(value, i)));  // 案例二
```

上述例子中的两个插入元素举例，都是不高效的写法，其背后执行相同的计算流程： 首先构造 `Demo` 临时对象，然后用 `std::make_pair` 生成 `pair` 临时对象，最后 `pair` 临时对象移进 `demo_group`，这里在多个技术点上有性能浪费。

首先，有多个临时对象构造、移动构造带来的浪费。`Demo` 临时对象构造完之后，会执行两次 `move` 构造，第一次是生成 `pair` 对象时，第二次是进入 `demo_group` 时。`pair` 临时对象构造后也会执行一次移动构造。

然后，键值对插入失败时，也有临时对象构造的消耗。上述代码会先生成 `pair` 对象，然后在 `pair` 对象插入 `demo_group` 时才去判断是否可插入。

修正：
```C++
std::unordered_map<std::string, Demo> demo_group;
demo_group.try_emplace(key, value, i);
```

如果插入不成功，则所有临时对象都不会构造；如果插入成功，则只会原地构造一次，不会触发移动构造。上述例子中，如果 Demo 对象不需要在插入时实时构造，则可以改为 `demo_group.emplace(key, demo)` 或者 `demo_group.emplace(key, std::move(demo))`。

反例：
```C++
std::vector<Demo> demo_group;
demo_group.push_back(Demo(value, 1));  // 案例一
demo_group.emplace_back(Demo(value, 1));  // 案例二
```

上述例子中的两个案例，都是不高效的写法，其背后执行相同的计算流程：首先构造 Demo 临时对象，然后将临时对象移进 demo_group。

修正：
```C++
std::vector<Demo> demo_group;
demo_group.emplace_back(value, 1);
```

# 类的初始化

1. 类成员变量初始化
	- 非静态成员
	- 静态成员
	- const和引用成员
	- 类内初始化（C++11）
	- 初始化列表的使用
2. 构造函数相关
	- 默认构造函数
	- 拷贝/移动构造函数
	- 委托构造函数
	- 继承构造函数（C++11）
3. 派生类初始化
	- 基类构造顺序
	- 虚基类初始化
	- 成员初始化顺序
4. 特殊成员初始化
	- 数组成员
	- 聚合类初始化
	- 位域初始化
5. 静态成员初始化
	- 静态常量
	- 静态非常量
	- inline静态成员（C++17）
6. 函数默认参数与重载
	- 默认参数规则
	- 重载冲突问题
7. 异常处理

## 类成员变量的初始化

**==类成员变量的初始化不能直接在类定义中进行，除非它们是静态常量成员。==**

C++的编译模型是基于翻译单元的，每个源文件（.cpp文件）被单独编译成目标文件（.o或.obj文件），然后链接在一起。类定义通常放在头文件中，而头文件可能会被多个源文件包含。如果在类定义中直接初始化非静态成员变量，那么每个包含该头文件的源文件都会有一份该成员变量的初始化代码，这会导致重复定义的问题。

在C++中，构造函数的主要职责之一就是初始化类的成员变量。通过在构造函数中初始化成员变量，可以确保在对象创建时，所有成员变量都被正确初始化。这种方式也使得初始化逻辑更加集中和清晰。

静态常量成员变量在类的所有实例中共享，并且它们的值在编译时就已经确定。因此，可以在类定义中直接初始化静态常量成员变量。这种初始化方式不会导致重复定义的问题，因为静态成员变量在类的所有实例中只有一份。

非法：
```C++
class BaseFunction {
    // 构造函数
    BaseFunction();
    
    // camera
    Camera cameraEntity(glm::vec3(0.0f, 0.0f, 3.0f)); // *** 会报错 ***
```

**==常量成员（`const`）和引用成员（`&`）必须在初始化列表中进行初始化，因为它们在对象的生命周期内不能被赋值。==**

```C++
class Example { 
public: 
    Example(int value) : constValue(value), refValue(value) {} 
    
private: 
    const int constValue; 
    int& refValue; 
};
```

==**使用初始化列表可以避免不必要的默认构造和赋值操作，从而提高性能。**== 对于类类型的成员变量，初始化列表可以直接调用其构造函数，而在构造函数体内进行赋值则会先调用默认构造函数，然后再调用赋值运算符，这可能会导致不必要的性能开销。


## 函数默认参数

在C++中，函数重载是允许的，但有一些规则需要遵循。函数重载的基本规则是函数名相同，但参数列表必须不同。参数列表的不同可以是参数的类型、参数的数量，或者参数的顺序。

然而，**默认参数**在函数重载中可能会引起一些问题。具体来说，如果两个重载函数的参数列表在考虑默认参数后变得相同，编译器将无法区分它们，从而导致编译错误。比如：

```C++
GLFWwindow* CreateWindowContext();

GLFWwindow* CreateWindowContext(const unsigned int SCR_WIDTH = 800, const unsigned int SCR_HEIGHT = 600, const char* windowName = "Hello window");
```

在这种情况下，编译器无法区分调用 `CreateWindowContext()` 时应该使用哪个函数。因此，这种重载是不允许的。

1. **默认参数**（如 `position = glm::vec3(0.0f)`）
    - **==默认参数必须写在 头文件 的函数/构造函数声明中**==，否则，如果分散在cpp文件中，可能会有一个文件给了默认值但另一个问题不知道的问题。
    
2. **成员初始化列表** 属于构造函数的**实现细节**，因此：
	- 如果构造函数在头文件中**内联定义**（直接在类声明中实现），成员初始化列表写在头文件。
	- 如果构造函数在源文件（.cpp）中**外部定义**，成员初始化列表写在源文件。

## 派生类的构造

**基类的构造函数应该在派生类的构造函数的成员初始化列表中调用。**

我们先来看一串代码：**注意第15、16行**

```C++
class TestBase
{
public:
	TestBase() {}
	TestBase(TestBase*) {}
	virtual ~TestBase() {}
};

class Test : public TestBase {
public:
	Test(TestBase*);
	~Test() {}
};

Test::Test(TestBase* xxx) {
	TestBase(xxx); // ***** 这里有问题 *****
}

int main() {
	auto x = new Test(nullptr);
	return 0;
}
```

它能编译成功吗？
……
No，无法编译通过！

我们来看看原因：
这里的错误是因为在 Test 类的构造函数实现中，`TestBase(xxx);` 这一行代码试图调用 TestBase 类的构造函数来初始化 Test 类的基类部分。然而，这种写法是错误的，因为基类的构造函数应该在派生类的构造函数的成员初始化列表中调用。

在 C++ 中，当一个派生类的对象被创建时，基类的构造函数会在派生类的构造函数之前被调用。因此，为了正确地初始化基类部分，需要在派生类的构造函数的成员初始化列表中调用基类的构造函数。在这个例子中，需要将 `TestBase(xxx);` 改为 `: TestBase(xxx)`，并将其放在 Test 类的构造函数的定义之前。

以下是正确的写法：

```
Test::Test(TestBase* xxx) : TestBase(xxx) {
}
```

这里，我们在 Test 类的构造函数的定义之前添加了 `: TestBase(xxx)`，这样就可以正确地调用 TestBase 类的构造函数，将 xxx 传递给它。这种写法满足了 C++ 的规则，因此代码可以正确编译和运行。

在C++中，类的继承是通过在派生类名后面跟随一个冒号(:)，然后是基类的名称来定义的。这个基类可以是任何有效的类类型，包括其他类。

在派生类的构造函数中，基类的构造函数可以通过初始化列表来调用。初始化列表的语法是冒号后面的基类名称，然后是括号括起来的参数列表。
```
class Base {
public:
    Base(int value) : value_(value) {}
private:
    int value_;
};

class Derived : public Base {
public:
    Derived(int value) : Base(value) {}
};
```
在这个例子中，`Derived`类继承自`Base`类，并且在它的构造函数中调用了`Base`类的构造函数。这个构造函数的参数是`value`，它被传递给`Base`类的构造函数。


**基类的移动构造函数也可以通过初始化列表来调用**

假如class B重载了移动构造函数，class D 继承 class B。为了确保移动语义适用于D从B继承来的那部分，我们需要为D提供移动构造函数。这样是不对的：
```
D(D&& other) : B(other)  
{  
  ...  
}
```

因为other是一个左值，B(other)会调用B(const B&)函数，而不是移动构造函数。正确的写法是：
```
D(D&& other) : B(std::move(other))  
{  
  ...  
}
```

