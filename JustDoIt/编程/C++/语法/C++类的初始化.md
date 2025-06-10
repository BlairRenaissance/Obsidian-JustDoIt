# 一、类成员变量初始化

## 1.1 非静态成员初始化

(1) 类内初始化（C++11+）

```C++
class Widget {  
	int value = 42; // 合法（C++11起）  
	std::string name{"default"};  
};
```

(2) 构造函数初始化列表（推荐方式）

```C++
class Example {  
public:  
    Example(int x, double y) : m_x(x), m_y(y) {}  
private:  
    int m_x;  
    double m_y;
};
```

(3) 构造函数体内赋值（不推荐）

```C++
class BadPractice {
public:
    BadPractice(int x) {
        m_x = x;  // 先默认构造，再赋值
    }

private:
    int m_x;
};
```

## 1.2 特殊成员初始化

``` C++
// 必须使用初始化列表的情况
class MustUseInitList {
    const int const_member;  // const成员
    std::vector<int>& vec_ref; // 引用成员
    NoDefaultCtorClass obj; // 无默认构造的类
    
public:
    MustUseInitList(int c, std::vector<int>& v, NoDefaultCtorClass o) 
        : const_member(c), vec_ref(v), obj(o) {} // ✅ 必须用初始化列表
};
```

| 成员类型       | 初始化要求                         |
| ---------- | ----------------------------- |
| `const` 成员 | 必须通过初始化列表初始化                  |
| 引用成员       | 必须通过初始化列表初始化                  |
| 无默认构造的类    | 必须通过初始化列表初始化                  |
| 类类型成员      | 推荐使用初始化列表直接构造                 |
| 数组成员       | C++11起支持直接初始化（C风格数组需构造时逐个初始化） |
| 位域         | 必须通过初始化列表初始化                  |
* ==C++标准规定，所有成员变量必须在构造函数体执行前完成初始化。==
* ==初始化列表在构造函数体执行之前被处理。==
* 引用成员必须通过初始化列表进行初始化，这是因为引用必须在其创建时被绑定到一个有效的对象，并且在其生命周期内不能被重新绑定。C++语言规范要求引用在声明时必须被初始化。这是为了确保引用在任何时候都指向一个有效的对象或变量。编译器会强制执行这一规则，以防止引用未初始化或指向无效对象。
- 当类成员没有默认构造函数时，如果不在初始化列表中显式初始化，编译器会尝试调用不存在的默认构造函数，导致编译错误。
* 类内初始化（C++11起）不适用于引用成员，因为引用的初始化需要在构造函数中进行。

```C++
class SpecialMembers {
public:
    SpecialMembers(int cv, int& ref, unsigned value) 
        : constValue(cv), refValue(ref), 
          arr{1,2,3}, bitfield(value) {}

private:
    const int constValue;
    int& refValue;
    int arr[3];
    unsigned bitfield : 4;
};
```

# 二、静态成员初始化

## 2.1 静态常量成员

```C++
class Constants {
public:
    static const int MAX_SIZE = 100;        // 类内声明
    static constexpr double PI = 3.14159;   // C++11起
    static const double staticConstDouble;  // 非整型常量静态成员，不能在类内初始化

};
// 类外定义（当需要取地址时）
const int Constants::MAX_SIZE;
constexpr double Constants::PI;
// 在类外定义非整型常量静态成员 
const double Example::staticConstDouble = 3.14;
```

问：什么时候能类内定义，什么时候必须类外定义？
- 想要能够类内定义，需要编译器可以在编译时确定这些值。
1. **类内初始化**：
	- **C++11之前**：在C++11之前，类内只能为静态整型常量提供初始值。这是因为编译器可以在编译时确定这些值，不需要在类外定义。
	- **C++11及之后**：C++11引入了`constexpr`关键字，并允许类内初始化更广泛的常量类型，包括非整型常量。这意味着你可以在类内为`constexpr`静态成员提供初始值，而不需要在类外定义。
2. **`const`和`constexpr`的区别**：
	- **`const`**：`const`关键字用于声明一个变量为常量，表示其值在初始化后不能被修改。`const`变量的值可以在运行时确定。
	- **`constexpr`**：`constexpr`关键字用于声明一个变量或函数可以在编译时求值。`constexpr`变量的值必须在编译时确定，因此它比`const`更严格。

---
问：“当需要取地址时”指的是什么？
1. **编译时常量优化**：
    - 对于`static const`和`static constexpr`成员，编译器通常会在编译时直接将它们的值内联到使用它们的地方。这意味着在大多数情况下，这些常量不需要实际的存储空间。
2. **取地址的需求**：
    - 当你需要获取一个静态常量成员的地址时，编译器不能再简单地内联其值，因为地址需要指向一个实际的内存位置。
    - 例如，如果你尝试获取`Constants::MAX_SIZE`的地址（`&Constants::MAX_SIZE`），编译器需要为`MAX_SIZE`分配存储空间，以便返回一个有效的地址。
3. **类外定义的必要性**：
    - **编译时优化**：静态常量通常在编译时内联，不需要存储空间。
	- **取地址需求**：当需要获取常量的地址时，必须在类外定义以分配存储空间。
	- **类外定义**：提供一个空定义即可满足取地址的需求。

## 2.2 静态非常量成员

```C++
class Counter {
public:
    static int count;  // 类内声明
};
// 类外定义（必须）
int Counter::count = 0;


// C++17起可用inline
class ModernCounter {
public:
    inline static int count = 0;  // 无需类外定义
};
```

问：为什么静态非常量成员必须类外定义？
1. **静态成员的特性**：
	- **类级别的成员**：静态成员变量属于类本身，而不是类的某个具体对象。它们在内存中只存在一份，无论创建多少个类的实例，静态成员变量都共享同一份数据。
	- **存储分配**：静态成员变量需要在类外定义，以便为其分配存储空间。类内的声明只是告诉编译器这个成员的存在，但不分配存储空间。
2. **链接要求**：
	- **链接器的需求**：在C++中，静态成员变量的定义需要在类外提供，以便链接器能够找到并分配存储空间。类内的声明只是一个声明，而不是定义。
	- **全局可见性**：在类外定义静态成员变量可以确保它在整个程序中是可见的，并且可以被正确地链接。

# 三、构造函数相关

| 类型     | 特点                                 |
| ------ | ---------------------------------- |
| 默认构造函数 | `ClassName() = default;`           |
| 拷贝构造函数 | `ClassName(const ClassName&);`     |
| 移动构造函数 | `ClassName(ClassName&&);` (C++11+) |
| 转换构造函数 | 单参数非explicit构造函数                   |
| 委托构造函数 | 构造函数可以调用同类其他构造函数 (C++11+)          |
| 继承构造函数 | `using Base::Base;` (C++11+)       |

| 场景      | 推荐方案          | 避免方案         |
| ------- | ------------- | ------------ |
| 简单对象初始化 | 直接构造函数        | 过度设计         |
| 多版本构造函数 | 委托构造函数        | 重复初始化代码      |
| 继承体系扩展  | 继承构造函数        | 手动转发所有基类构造函数 |
| 复杂对象创建  | 工厂模式 + 智能指针   | 直接new操作      |
| 菱形继承    | 虚继承 + 最派生类初始化 | 多层非虚继承       |
| 跨模块对象创建 | 抽象工厂 + DLL接口  | 暴露具体类实现      |

## 3.1 委托构造函数

传统方式的问题：

```C++
class Config {
public:
    Config() { 
        init(0, "");  // 重复初始化逻辑
    }
    Config(int v) {
        init(v, "");
    }
    Config(string s) {
        init(0, s);
    }
private:
    void init(int v, string s) { /*...*/ }
};
```

现代解决方案（C++11起）：

```C++
class Config {
public:
    Config() : Config(0, "") {}  // 委托主构造函数
    Config(int v) : Config(v, "") {}
    Config(string s) : Config(0, s) {}

    // 主构造函数
    Config(int v, string s) { 
        /* 统一初始化逻辑 */
    }
};
```

## 3.2 继承构造函数

```C++
class Base {
public:
    explicit Base(int) {}
    Base(double, const char*) {}
};

class Derived : public Base {
public:
    using Base::Base;  // 继承所有Base构造函数
    
    // 可扩展新构造函数
    Derived(const string& s) : Base(s.size()) {}
};
```
- 继承的构造函数不会初始化派生类新增成员。
- 如果基类构造函数有默认参数，继承时会保留默认参数。

## 3.3 异常处理

```C++
class ResourceHolder {
public:
    ResourceHolder() 
        try : resource(new int[1024]) {  // 函数try块
            // 构造函数体
        } catch(...) {
            delete[] resource;
            throw;
        }
private:
    int* resource;
};
```

# 四、继承体系初始化

## 4.1 派生类构造顺序

1. 虚基类构造（按继承顺序）
2. 直接基类构造（按声明顺序）
3. 成员对象构造（按声明顺序）
4. 派生类构造函数体执行

```C++
class VirtualBase {
public:
    VirtualBase() { std::cout << "VirtualBase\n"; }
};

class Base1 : virtual public VirtualBase {
public:
    Base1() { std::cout << "Base1\n"; }
};

class Base2 : virtual public VirtualBase {
public:
    Base2() { std::cout << "Base2\n"; }
};

class Member {
public:
    Member() { std::cout << "Member\n"; }
};

class Final : public Base1, public Base2 {
    Member m;
public:
    Final() { std::cout << "Final\n"; }
};

/* 构造顺序：
VirtualBase → Base1 → Base2 → Member → Final */
```

## 4.2 虚基类构造

```C++
class Base {
public:
    Base(int) { std::cout << "Base\n"; }
};  

class V1 : virtual public Base {
public:
    V1() : Base(1) { std::cout << "V1\n"; }
};

class V2 : virtual public Base {
public:
    V2() : Base(2) { std::cout << "V2\n"; }
};

class Derived : public V1, public V2 {
public:
    // 必须直接初始化虚基类
    Derived() : Base(0), V1(), V2() { 
        std::cout << "Derived\n"; 
    }
};

/* 输出：
Base(0)  // 虚基类只初始化一次
V1
V2
Derived */
```

**虚基类构造规则**：
- 由最派生类直接初始化。
- 中间类的虚基类初始化会被忽略。
- 初始化顺序按继承图深度优先、从左到右。

---
问：为什么菱形继承必须使用虚继承？
问题场景
```C++
     File
    /   \
Input  Output
    \   /
     IO

// 传统继承导致File重复
class File { /*...*/ };
class Input : public File { /*...*/ };
class Output : public File { /*...*/ };
class IO : public Input, public Output {
    // 包含两个File子对象！
};
```

解决方案
```C++
class File { /*...*/ };
class Input : virtual public File { /*...*/ };  // 虚继承
class Output : virtual public File { /*...*/ }; // 虚继承
class IO : public Input, public Output {
    // 只包含一个File子对象
};
```

---
问：为什么要避免在基类的构造函数中调用虚函数？

当创建一个派生类的对象时，基类的构造函数会先被调用，然后是派生类的构造函数。在基类构造函数执行期间，派生类的部分还没有初始化。这时候如果基类构造函数调用了一个虚函数，根据C++的动态绑定机制，这个调用应该指向基类自己的虚函数实现，而不是派生类的。而如果基类的虚函数是纯虚函数，也就是没有实现的话，这时候调用就会导致未定义行为，通常是程序崩溃。

> 标准规定（ISO C++ §15.7）
> ”当从构造函数或析构函数直接或间接调用虚函数时，被调用的函数是构造函数或析构函数自身类或其基类中定义的版本，而不是派生类中的覆盖版本。“

```C++
class Base {
public:
    Base() {
        // 危险
        virtualMethod(); 
    }
    
    virtual void virtualMethod() = 0;
};
```

解决方案1：两阶段初始化模式

```C++
class SafeBase {
public:
    SafeBase() = default;
    
    template<typename T, typename... Args>
    static std::unique_ptr<T> create(Args&&... args) {
        auto obj = std::make_unique<T>(std::forward<Args>(args)...);
        obj->postInit();  // 对象完全构造后调用
        return obj;
    }

protected:
    virtual void postInit() {}  // 空实现
};

class SafeDerived : public SafeBase {
protected:
    void postInit() override {
        // 安全访问派生类成员
    }
};
```

解决方案2：编译期多态（CRTP模式）

```C++
template<typename Derived>
class CRTPBase {
public:
    CRTPBase() {
        // 编译期绑定，无虚函数开销
        static_cast<Derived*>(this)->safeInit();
    }
};

class CRTPDerived : public CRTPBase<CRTPDerived> {
    friend class CRTPBase<CRTPDerived>;
private:
    void safeInit() { /* 安全初始化 */ }
};
```

## 4.3 直接基类构造

```C++
class Base {
public:
    explicit Base(int) {}
};

class Derived : public Base {
public:
    // 必须显式调用基类构造函数
    Derived() : Base(42) {} 
    
    // 错误：Base没有默认构造函数
    // Derived() {}
};
```

## 4.4 成员对象构造

**如果成员变量没有默认构造函数，必须通过初始化列表来提供必要的参数进行初始化。**

当编译器生成构造函数代码时，它会按照以下步骤进行：
1. 调用基类构造函数（如果有）。
2. 调用成员变量的构造函数。
3. 执行构造函数体。

如果成员变量没有默认构造函数，编译器在步骤2中无法完成成员变量的初始化，从而导致编译错误。使用初始化列表可以在步骤2中提供必要的参数，确保成员变量被正确初始化。

```C++
class Database {
public:
    Database(int port) { /*...*/ }
};

class Service {
    Database db;
public:
    // 必须通过初始化列表构造
    Service() : db(3306) {} 

    // 错误：Database没有默认构造函数
    // Service() { db = Database(3306); } 
};
```