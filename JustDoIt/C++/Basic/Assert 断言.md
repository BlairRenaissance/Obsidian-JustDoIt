
如果程序以 `std::exit` 的方式中断了，那么所有的调用堆栈与调试信息都会丢失。以 `std::abort` 方式中断更好，可以从中断的位置开始debugging。断言能够使程序以`std::abort` 方式中断。

## `assert`

An assertion is an expression that will be true unless there is a bug in the program. If the expression evaluates to `true`, the assertion statement does nothing. If the conditional expression evaluates to `false`, an error message is displayed and the program is terminated (via `std::abort`). This error message typically contains the expression that failed as text, along with the name of the code file and the line number of the assertion. 

In C++, runtime assertions are implemented via the assert preprocessor macro (断言预处理器宏), which lives in the `<cassert>` header. 

```
#include <cassert> // for assert()
#include <cmath> // for std::sqrt
#include <iostream>

double calculateTime(double initialHeight, double gravity)
{
  assert(gravity > 0.0); // Assert

  if (initialHeight <= 0.0) return 0.0;

  return std::sqrt((2.0 * initialHeight) / gravity);
}
```


## `static_assert`

C++也提供了另一种在编译期就进行检查的静态断言`static_assert`。A static_assert is an assertion that is checked at compile-time rather than at runtime, with a failing `static_assert` causing a compile error. Unlike assert, which is declared in the `<cassert>` header, static_assert is a keyword, so no header needs to be included to use it.

A `static_assert` takes the following form:
```
static_assert(condition, diagnostic_message)
```

If the condition is not true, the diagnostic message is printed. Here’s an example of using static_assert to ensure types have a certain size:

On the author’s machine, when compiled, the compiler errors:
```
1>c:\consoleapplication1\main.cpp(19): error C2338: long must be 8 bytes
```

A few useful notes about `static_assert`:

- Because `static_assert` is evaluated by the compiler, the condition must be a constant expression.
- `static_assert` can be placed anywhere in the code file (even in the global namespace).
- `static_assert` is not compiled out in release builds.

Prior to C++17, the diagnostic message must be supplied as the second parameter. Since C++17, providing a diagnostic message is optional.


## ⚠️注意

1. 一般情况下，assert仅在debug模式下生效。
	`assert` 宏在每一次检查中都有一些性能消耗。因此，开发者们更偏向于仅在debug模式下使用断言。为此，C++制定了一种关闭断言的方案：
	
    ***If the macro `NDEBUG` is defined, the assert macro gets disabled.*** 
    
    一些编译器会在release模式中默认定义`NDEBUG`，比如 Visual Studio中the following preprocessor definitions are set at the project level: `WIN32;NDEBUG;_CONSOLE`. 如果希望在release模式下使用断言，需要从设置中移除`NDEBUG`。

2. 谨防内存泄漏。
	断言使用的 `abort()` function 会立刻终止程序。Because of this, asserts should be used only in cases where corruption isn’t likely to occur if the program terminates unexpectedly.