
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