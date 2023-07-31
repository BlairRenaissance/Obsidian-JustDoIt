
## Type deduction for string literals

For historical reasons, string literals in C++ have a strange type. Therefore, the following probably won’t work as expected:

```cpp
auto s { "Hello, world" }; // s will be type const char*, not std::string
```

If you want the type deduced from a string literal to be `std::string` or `std::string_view`, you’ll need to use the `s` or `sv` literal suffixes.

```cpp
#include <string>
#include <string_view>

int main()
{
    using namespace std::literals; // easiest way to access the s and sv suffixes

    auto s1 { "goo"s };  // "goo"s is a std::string literal, so s1 will be deduced as a std::string
    auto s2 { "moo"sv }; // "moo"sv is a std::string_view literal, so s2 will be deduced as a std::string_view

    return 0;
}
```


## Type deduction drops const / constexpr qualifiers

In most cases, type deduction will drop the `const` or `constexpr` qualifier from deduced types. For example:
```cpp
int main()
{
    const int x { 5 }; // x has type const int
    auto y { x };      // y will be type int (const is dropped)

    return 0;
}
```
In the above example, `x` has type `const int`, but when deducing a type for variable `y` using `x` as the initializer, type deduction deduces the type as `int`, not `const int`.

If you want a deduced type to be const or constexpr, you must supply the const or constexpr yourself. To do so, simply use the `const` or `constexpr` keyword in conjunction with the `auto` keyword:
```cpp
int main()
{
    const int x { 5 };  // x has type const int (compile-time const)
    auto y { x };       // y will be type int (const is dropped)

    constexpr auto z { x }; // z will be type constexpr int 

    return 0;
}
```
In this example, the type deduced from `x` will be `int` (the `const` is dropped), but because we’ve re-added a `constexpr` qualifier during the definition of variable `z`, variable `z` will be a `constexpr int`.


## Type deduction for functions

When using an `auto` return type, all return statements within the function must return values of the same type, otherwise an error will result. For example:
```cpp
auto someFcn(bool b)
{
    if (b)
        return 5; // return type int
    else
        return 6.7; // return type double
}
```
In the above function, the two return statements return values of different types, so the compiler will give an error.

⚠️ Favor explicit return types over function return type deduction for normal functions.

**Trailing return type syntax**

The `auto` keyword can also be used to declare functions using a trailing return syntax, where the return type is specified after the rest of the function prototype.

Consider the following function:
```cpp
int add(int x, int y)
{
  return (x + y);
}
```
Using the trailing return syntax, this could be equivalently written as:
```cpp
auto add(int x, int y) -> int
{
  return (x + y);
}
```
In this case, `auto` does not perform type deduction -- it is just part of the syntax to use a trailing return type.

Why would you want to use this? The trailing return syntax is required for some advanced features of C++, such as ***lambdas*** (which we cover in lesson [12.7 -- Introduction to lambdas (anonymous functions)](https://www.learncpp.com/cpp-tutorial/introduction-to-lambdas-anonymous-functions/)).