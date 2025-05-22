## Code Coverage 代码覆盖率

Consider the following function:

```cpp
int foo(int x, int y)
{
    int z{ y };
    if (x > y)
    {
        z = x;
    }
    return z;
}
```

Calling this function as `foo(1, 0)` will give you complete statement coverage for this function, as every statement in the function will execute.

For our `isLowerVowel()` function:

```cpp
bool isLowerVowel(char c)
{
    switch (c) // statement 1
    {
    case 'a':
    case 'e':
    case 'i':
    case 'o':
    case 'u':
        return true; // statement 2
    default:
        return false; // statement 3
    }
}
```

This function will require two calls to test all of the statements, as there is no way to reach statement 2 and 3 in the same function call.

While aiming for 100% statement coverage is good, it’s often not enough to ensure correctness.

---
## Assert 断言

the **assert** preprocessor macro, which lives in the `<cassert>` header.

断言比注释更好，因为它们既能记录条件，又能强制执行。当代码更改而注释未更新时，注释可能会变得过时，但过时的断言会造成代码正确性问题，因此开发人员不太可能让它们闲置。

### Making assert more descriptive

There’s a little trick you can use to make your assert statements more descriptive. Simply add a string literal joined by a logical AND:

```cpp
assert(found && "Car could not be found in database");
```

When the assert triggers, the string literal will be included in the assert message:
```

Assertion failed: found && "Car could not be found in database", file C:\\VCProjects\\Test.cpp, line 34
```

That gives you some additional context as to what went wrong.

---
### `NDEBUG`

No Debug 的缩写，表示非 Debug 模式。

you can enable or disable asserts within a given translation unit. To do so, place one of the following on its own line **before** any `#includes`: `#define NDEBUG` (to disable asserts) or `#undef NDEBUG` (to enable asserts). Make sure that you do not end the line in a semicolon.

`assert` 的典型实现逻辑（简化版）
```
#ifdef NDEBUG
    #define assert(expression) ((void)0)
#else
    #define assert(expression) \
        ((expression) ? (void)0 : __assert_fail(...))
#endif
```

如果定义了 `NDEBUG`，`assert` 宏会被定义成空操作（通常是 `(void)0`），也就是说所有的 `assert` 语句都会被编译器忽略，不会生成任何代码。

---
### `static_assert`

A **static_assert** is an assertion that is checked at compile-time rather than at runtime, with a failing `static_assert` causing a compile error. 

Unlike assert, which is declared in the `<cassert>` header, `static_assert` is a keyword, so no header needs to be included to use it.

- 由于 static_assert 由编译器执行，因此条件必须是常量表达式。 
- static_assert 可以放置在代码文件中的任何位置（即使是全局命名空间）。
- static_assert 在发布版本中不会像普通断言那样被停用。
- 由于编译器会执行求值，因此 static_assert 没有运行时开销。

---
## 输入检查

编写程序时，请考虑用户会如何误用程序，尤其是在文本输入方面。对于每个文本输入点，请考虑： 
- 提取可能会失败吗？ 
- 用户输入的内容可能会超出预期吗？ -> 使用 `getOperator` 的方式检查
- 用户可能会输入无意义的输入吗？ -> 使用 `clearFailedExtraction` 的方式检查
- 用户可能会输入溢出吗？

综合的错误检查：

``` C++
#include <cstdlib> // for std::exit
#include <iostream>
#include <limits> // for std::numeric_limits

void ignoreLine()
{
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
}

// returns true if extraction failed, false otherwise
bool clearFailedExtraction()
{
    // Check for failed extraction
    if (!std::cin) // If the previous extraction failed
    {
        if (std::cin.eof()) // If the stream was closed
        {
            std::exit(0); // Shut down the program now
        }

        // Let's handle the failure
        std::cin.clear(); // Put us back in 'normal' operation mode
        ignoreLine();     // And remove the bad input

        return true;
    }

    return false;
}

double getDouble()
{
    while (true) // Loop until user enters a valid input
    {
        std::cout << "Enter a decimal number: ";
        double x{};
        std::cin >> x;

        if (clearFailedExtraction())
        {
            std::cout << "Oops, that input is invalid.  Please try again.\n";
            continue;
        }

        ignoreLine(); // Remove any extraneous input
        return x;     // Return the value we extracted
    }
}

char getOperator()
{
    while (true) // Loop until user enters a valid input
    {
        std::cout << "Enter one of the following: +, -, *, or /: ";
        char operation{};
        std::cin >> operation;

        if (!clearFailedExtraction()) // we'll handle error messaging if extraction failed below
             ignoreLine(); // remove any extraneous input (only if extraction succeded)

        // Check whether the user entered meaningful input
        switch (operation)
        {
        case '+':
        case '-':
        case '*':
        case '/':
            return operation; // Return the entered char to the caller
        default: // Otherwise tell the user what went wrong
            std::cout << "Oops, that input is invalid.  Please try again.\n";
        }
    }
}

void printResult(double x, char operation, double y)
{
    std::cout << x << ' ' << operation << ' ' << y << " is ";

    switch (operation)
    {
    case '+':
        std::cout << x + y << '\n';
        return;
    case '-':
        std::cout << x - y << '\n';
        return;
    case '*':
        std::cout << x * y << '\n';
        return;
    case '/':
        if (y == 0.0)
            break;

        std::cout << x / y << '\n';
        return;
    }

    std::cout << "???";  // Being robust means handling unexpected parameters as well, even though getOperator() guarantees operation is valid in this particular program
}

int main()
{
    double x{ getDouble() };
    char operation{ getOperator() };
    double y{ getDouble() };

    // Handle division by 0
    while (operation == '/' && y == 0.0)
    {
        std::cout << "The denominator cannot be zero.  Try again.\n";
        y = getDouble();
    }

    printResult(x, operation, y);

    return 0;
}
```