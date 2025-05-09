
## 强引用改弱引用

防止World无法正常析构。
![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Pasted%20image%2020230907200536.png)

## 基类的析构函数

不写成虚函数，会导致子类析构函数无法执行，资源与内存未正确释放。
```
-    ~FrameBuffer();
+    virtual ~FrameBuffer();
```

在 C++ 中，我们可以使用基类指针来指向派生类对象。例如：
```
CPPCopyclass Base {
public:
    virtual ~Base() {}
};

class Derived : public Base {
public:
    ~Derived() {}
};

int main() {
    Base* ptr = new Derived();
    delete ptr;
    return 0;
}
```

在上面的示例中，我们定义了一个基类 `Base` 和一个派生类 `Derived`，然后在 `main` 函数中，创建了一个 `Derived` 类型的对象，并使用基类指针 `ptr` 指向它。最后，我们通过 `delete` 关键字删除了 `ptr`，这会调用 `Base` 类的析构函数。由于 `Base` 类的析构函数是虚函数，因此会**先调用 `Derived` 类的析构函数，然后再调用 `Base` 类的析构函数**。

这种使用基类指针来指向派生类对象的方式是多态的一种体现，它允许我们在运行时动态地确定对象的类型，并调用正确的方法。但是，如果基类没有定义虚析构函数，那么在删除基类指针时，只会调用基类的析构函数，而不会调用派生类的析构函数，这可能会导致派生类的资源没有被正确地释放，从而产生内存泄漏或资源泄漏等问题。

因此，使用一个指向派生类对象的基类指针来删除对象，需要确保基类定义了虚析构函数，以确保在删除对象时会调用正确的析构函数。

## C++非const引用问题 

`error: cannot bind non-const lvalue reference of type` 如果把一个临时变量当成非const引用传入函数，由于临时变量的特殊性，无法对临时变量进行操作，同时临时变量随时可能消失，修改临时变量也毫无意义。因此，临时变量不能作为非const引用。

比如，当有以下操作时：
```
std::shared_ptr<MapIndex<Vector3ui>> ptrWallIndexs = std::make_shared<MapIndex<Vector3ui>>(UINTX3);

mpWorld->getUnityMeshDataUploader()->SetIncrementIndexBuffer(meshIDWall, ptrWallIndexs);
```

函数 `SetIncrementIndexBuffer` 的第二个参数必须用const修饰，否则会有上述错误。
```
void UnityMeshDataUploader::SetIncrementIndexBuffer(long meshID, const shared_ptr<tencentmap::BaseVertex> &IndexBuffer)
```


## 回调中删除自身导致ANR

在当前Overlay的onDraw回调中删除该Overlay对象，如果该Overlay有一个锁，OverlayManager也有一个锁，那么很容易ANR。

## 析构中get自身是危险操作

在一个shared_ptr的析构函数里再去get这个shared_ptr，因为还没有置空所以能够get到，但此时该shared_ptr已被破坏，所以很容易导致崩溃，不要这样用。

## to_string生成的对象没有被引用会被释放

```
-      const char* id = std::to_string(universal_id_).c_str();  
-      snprintf(buffer, buffer_size, "%p@%s", p, id);

+      std::string tempStr = std::to_string(uniqueId);
+      snprintf(buffer, buffer_size, "%p@%s", p, tempStr.c_str());
```

由于to_string生成的对象没有被引用，所以会直接释放，导致id为空。需要一个临时的string对象来承接to_string生成的对象。

## reserve不会影响size

`reserve`不影响size，所以`memset`分配不到空间。需要改成`resize`。
```
std::vector<char> vtTmp;

-      vtTmp.reserve(indexBufferSize);
+      vtTmp.resize(indexBufferSize);

memset(&vtTmp[0], 0, vtTmp.size());
```

## this 指针

this 指针是指向当前对象的地址。this指针主要用于在类的成员函数中访问当前对象的成员变量和成员函数。

当一个对象调用自己的成员函数时，编译器会隐式地将对象的地址传递给成员函数，作为一个隐藏的参数，这个隐藏的参数就是this指针。通过this指针，成员函数可以访问和操作当前对象的成员变量和成员函数。

this指针只能在非静态成员函数中使用，因为静态成员函数没有this指针，它们不属于任何具体的对象。

## 清理指针

`Example` 类的析构函数中释放了 `mpValue` 指向的内存，并将 `mpValue` 置为 `nullptr`。这样做可以确保在对象销毁后，`mpValue` 不会指向一个已经释放的内存地址。

```cpp
#include <iostream>
class Example {
public:
    void* mpValue;
    
    Example() : mpValue(nullptr) {} // 初始化为 nullptr
    
    ~Example() {
        // 清理内存
        delete (int*)mpValue; // 假设 mpValue 指向 int 类型
        mpValue = nullptr;     // 将指针置为 nullptr
    }
    
    template <typename T>
    void setValue(const T &value) {
        *(T*)mpValue = value; // 将 value 赋值给 mpValue 指向的内存
    }
};

int main() {
    Example ex;
    // 为 mpValue 分配内存
    ex.mpValue = new int;
    // 设置值
    ex.setValue(42);
    // 输出结果
    std::cout << *(int*)ex.mpValue << std::endl; // 输出 42
    // 当 Example 对象超出作用域时，自动调用析构函数，清理内存
    return 0;
}
```

# 类的初始化

详细内容可见 [[C++类的初始化]]。

## 类成员变量的初始化

**==类成员变量的初始化不能直接在类定义中进行，除非它们是静态常量成员。==**

C++的编译模型是基于翻译单元的，每个源文件（.cpp文件）被单独编译成目标文件（.o或.obj文件），然后链接在一起。类定义通常放在头文件中，而头文件可能会被多个源文件包含。**如果在类定义中直接初始化非静态成员变量，那么每个包含该头文件的源文件都会有一份该成员变量的初始化代码，这会导致重复定义的问题。**

在C++中，构造函数的主要职责之一就是初始化类的成员变量。通过在构造函数中初始化成员变量，可以确保在对象创建时，所有成员变量都被正确初始化。这种方式也使得初始化逻辑更加集中和清晰。

静态常量成员变量在类的所有实例中共享，并且它们的值在编译时就已经确定。因此，可以在类定义中直接初始化静态常量成员变量。这种初始化方式不会导致重复定义的问题，因为静态成员变量在类的所有实例中只有一份。

非法：
```C++
class BaseFunction {
    // 构造函数
    BaseFunction();
    
    // *** 会报错 ***
    Camera cameraEntity(glm::vec3(0.0f, 0.0f, 3.0f));
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

---
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

---
**如果成员变量没有默认构造函数，必须通过初始化列表来提供必要的参数进行初始化。**

当编译器生成构造函数代码时，它会按照以下步骤进行：
1. **调用基类构造函数**（如果有）。
2. **调用成员变量的构造函数**。
3. **执行构造函数体**。

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
