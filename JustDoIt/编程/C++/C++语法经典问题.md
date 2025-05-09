
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


## emplace/try_emplace/emplace_back

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

