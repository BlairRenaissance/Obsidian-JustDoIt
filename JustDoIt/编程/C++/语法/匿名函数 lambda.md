## 结构解释

lambda函数的语法如下：

`[capture](parameters) -> return_type { body }`

**示例**

```C++
std::function<std::string(int, const std::string&)> func = [](int count, const std::string& str) -> std::string {
	std::string result; 
	for (int i = 0; i < count; ++i) { 
		result += str;
	}
	return result;
};

// 调用 lambda 函数 
std::string result = func(3, "Hello");
```
#### 1. 捕获列表 `[]`

捕获列表表示lambda函数需要捕获的外部变量。

#### 2. 参数列表 `()`

参数列表为表示lambda函数接受的传入参数。

#### 3. 返回类型 `->`

箭头后的为返回类型，表示lambda函数返回的类型。

#### 4. 函数体 `{ ... }`

包含lambda函数的代码。

## 捕获方式

C++ 中的 lambda 表达式支持多种捕获方式，允许你在 lambda 中使用外部作用域的变量。以下是几种常见的捕获方式：

### 按值捕获 `[=]`

使用 `[=]` 可以按值捕获外部作用域中的所有变量。这意味着 lambda 内部会创建这些变量的副本，修改这些副本不会影响外部变量。
```cpp
int x = 10; auto lambda = [=]() { return x + 1; };
```

### 按引用捕获 `[&]`

使用 `[&]` 可以按引用捕获外部作用域中的所有变量。这意味着 lambda 内部对这些变量的修改会影响到外部变量。
```cpp
int x = 10; auto lambda = [&]() { x += 1; };
```

### 显式捕获

可以显式指定要捕获的变量，并选择按值或按引用捕获。
```cpp
int x = 10; 
auto lambda = [x]() { return x + 1; }; // 按值捕获
auto lambda = [&x]() { x += 1; }; // 按引用捕获
```

### 混合捕获

可以混合使用按值和按引用捕获。
```cpp
int x = 10, y = 20; 
auto lambda = [=, &y]() { return x + y; }; // x 按值捕获，y 按引用捕获
```

### 捕获 this 指针

在类的成员函数中，可以捕获 `this` 指针，以便在 lambda 中访问类的成员。

```cpp
class MyClass {
public:
    int value;
    void func() {
        auto lambda = [this]() { value += 1; };
    }
};
```

### 初始化捕获

C++14 引入，允许在捕获列表中初始化新的变量。

```cpp
int x = 10; 
auto lambda = [z = x + 1]() { return z; };
```

通过这些捕获方式，C++ 的 lambda 表达式提供了灵活的机制来访问和操作外部作用域中的变量。选择合适的捕获方式可以帮助你更好地控制 lambda 的行为和性能。