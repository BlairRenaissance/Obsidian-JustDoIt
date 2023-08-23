
## Template parameter declaration

```cpp
template <typename T> // this is the template parameter declaration
T max(T x, T y) // this is the function template definition for max<T>
{
    return (x < y) ? y : x;
}
```

There is no difference between the `typename` and `class` keywords in this context. You will often see people use the `class` keyword since it was introduced into the language earlier. However, we prefer the newer `typename` keyword, because it makes it clearer that the type template parameter can be replaced by any type (such as a fundamental type), not just class types.


## Functions templates with multiple template type parameters

```cpp
#include <iostream>

template <typename T, typename U> // We're using two template type parameters named T and U
T max(T x, U y) // x can resolve to type T, and y can resolve to type U
{
    return (x < y) ? y : x; // uh oh, we have a narrowing conversion problem here
}
```


## Abbreviated function templates (C++20)

C++20 introduces a new use of the `auto` keyword: When the `auto` keyword is used as a parameter type in a normal function, the compiler will automatically convert the function into a function template with each auto parameter becoming an independent template type parameter. This method for creating a function template is called an abbreviated function template.

For example:
```cpp
auto max(auto x, auto y)
{
    return (x < y) ? y : x;
}
```

is shorthand in C++20 for the following:
```cpp
template <typename T, typename U>
auto max(T x, U y)
{
    return (x < y) ? y : x;
}
```

There isn’t a concise way to use abbreviated function templates when you want more than one auto parameter to be the same type. That is, there isn’t an easy abbreviated function template for something like this:
```cpp
template <typename T>
auto max(T x, T y) // two parameters of the same type
{
    return (x < y) ? y : x;
}
```


## Non-type template parameters

A **non-type template parameter** is a template parameter with a fixed type that serves as a placeholder for a constexpr value passed in as a template argument.

A non-type template parameter can be any of the following types:

- An integral type
- An enumeration type
- `std::nullptr_t`
- A floating point type (since C++20)
- A pointer or reference to an object
- A pointer or reference to a function
- A pointer or reference to a member function
- A literal class type (since C++20)

Here’s a simple example that uses an int non-type template parameter:
```cpp
#include <iostream>

template <int N> // declare a non-type template parameter of type int named N
void print()
{
    std::cout << N << '\n'; // use value of N here
}

int main()
{
    print<5>(); // 5 is our non-type template argument

    return 0;
}
```

This example prints:
```
5
```