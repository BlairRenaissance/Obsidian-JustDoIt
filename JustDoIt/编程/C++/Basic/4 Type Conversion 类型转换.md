
## Standard conversions

The C++ language standard defines how different fundamental types (and in some cases, compound types) can be converted to other types. These conversion rules are called the **standard conversions**.

The standard conversions can be broadly divided into 4 categories, each covering different types of conversions:
- Numeric promotions
- Numeric conversions
- Arithmetic conversions (covered in lesson [8.5 -- Arithmetic conversions](https://www.learncpp.com/cpp-tutorial/arithmetic-conversions/))
- Other conversions (which includes various pointer and reference conversions)

### Numeric promotions
A numeric promotion is the type conversion of certain narrower numeric types (such as a `char`) to certain wider numeric types (typically `int` or `double`) that can be processed efficiently and is less likely to have a result that overflows.

Not all widening conversions are numeric promotions.

Some widening type conversions (such as `char` to `short`, or `int` to `long`) are not considered to be numeric promotions in C++, they are `numeric conversions`. This is because such conversions do not assist in the goal of converting smaller types to larger types that can be processed more efficiently.

### Arithmetic conversion

The usual arithmetic conversion rules are pretty simple. The compiler has a prioritized list of types that looks something like this:

- long double (highest)
- double
- float
- unsigned long long
- long long
- unsigned long
- long
- unsigned int
- int (lowest)

There are only two rules:
- If the type of at least one of the operands is on the priority list, the operand with lower priority is converted to the type of the operand with higher priority.
- Otherwise (the type of neither operand is on the list), both operands are numerically promoted


### ⚠️注意

1. Brace initialization disallows narrowing conversions. `int x { 3.5 }; // Won't compile. brace-initialization disallows conversions that result in data loss`
2. Make intentional narrowing conversions explicit. Convert an implicit narrowing conversion into an explicit narrowing conversion using `static_cast`.
3. a compiler that has **signed/unsigned warnings** enabled will issue a warning for case when the source value of a narrowing conversion isn’t known until runtime, the result of the conversion also can’t be determined until runtime. 对于运行时才能得到值，编译器无法提前判断数值转换安全性的情况，编译器会给出警告。但是当转换的值用 **`constexpr`** 修饰时，编译器必须知道要转换的具体值。
	```
	#include <iostream>
	
	int main()
	{
	    constexpr int n1{ 5 };   // note: constexpr
	    unsigned int u1 { n1 };  // okay: conversion is not narrowing due to exclusion clause
	
	    constexpr int n2 { -5 }; // note: constexpr
	    unsigned int u2 { n2 };  // compile error: conversion is narrowing due to value change
	
	    return 0;
	}
	```
	
	`n1` is constexpr, and its value `5` can be represented exactly in the destination type (as unsigned value `5`). Therefore, this is not considered to be a narrowing conversion, and we are allowed to list initialize `u1` using `n1`.
	
	In the case of `n2` and `u2`, things are similar. Although `n2` is constexpr, its value `-5` cannot be represented exactly in the destination type, so this is considered to be a narrowing conversion, and because we are doing list initialization, the compiler will error and halt the compilation.


# Explicit type conversion

C++ supports 5 different types of casts: `C-style casts`, `static casts`, `const casts`, `dynamic casts`, and `reinterpret casts`. The latter four are sometimes referred to as named casts.

`Const casts` and `reinterpret casts` should generally be avoided because they are only useful in rare cases and can be harmful if used incorrectly.

###  C-style cast

`(double)x` or `double(x)`

Although a `C-style cast` appears to be a single cast, it can actually perform a variety of different conversions depending on context. This can include a `static cast`, a `const cast` or a `reinterpret cast` (the latter two of which we mentioned above you should avoid). As a result, `C-style casts` are at risk for being inadvertently misused and not producing the expected behavior, something which is easily avoidable by using the C++ casts instead. 

So, avoid using C-style casts.

### Static cast

`static_cast<double>(x)`

`static_cast` is best used to convert one fundamental type into another.

The main advantage of `static_cast` is that it provides compile-time type checking, making it harder to make an inadvertent error.