
## Int

### Initialization

### Unsigned integer

优点：
1. 在进行位操作时，首选使用无符号数。在某些算法（如加密和随机数生成）中需要定义明确的环绕行为时非常有用。
2. 其次，在某些情况下，无法避免地需要使用无符号数，主要涉及数组索引。

缺点：Unsigned integer overflow
What happens if we try to store the number `280` (which requires 9 bits to represent) in a 1-byte (8-bit) unsigned integer? The answer is overflow.If an unsigned value is out of range, it is divided by one greater than the largest number of the type, and only the remainder kept. The number `280` is too big to fit in our 1-byte range of 0 to 255. 1 greater than the largest number of the type is 256. Therefore, we divide 280 by 256, getting 1 remainder 24. The remainder of 24 is what is stored.

Let’s take a look at this using 2-byte shorts:
```cpp
#include <iostream>

int main()
{
    unsigned short x{ 65535 }; // largest 16-bit unsigned value possible
    std::cout << "x was: " << x << '\n';

    x = 65536; 
    std::cout << "x is now: " << x << '\n';

    x = 65537; 
    std::cout << "x is now: " << x << '\n';

    return 0;
}
```

```
x was: 65535
x is now: 0
x is now: 1
```

It’s possible to wrap around the other direction as well. 0 is representable in a 2-byte unsigned integer, so that’s fine. -1 is not representable, so it wraps around to the top of the range, producing the value 65535. -2 wraps around to 65534. And so forth.
```cpp
#include <iostream>

int main()
{
    unsigned short x{ 0 }; // smallest 2-byte unsigned value possible
    std::cout << "x was: " << x << '\n';

    x = -1; // -1 is out of our range, so we get modulo wrap-around
    std::cout << "x is now: " << x << '\n';

    x = -2; // -2 is out of our range, so we get modulo wrap-around
    std::cout << "x is now: " << x << '\n';

    return 0;
}
```

```
x was: 0
x is now: 65535
x is now: 65534
```

在电脑游戏《文明》中，甘地经常是第一个使用核武器的人，这似乎与他预期的被动本性相反。玩家有一个理论，甘地的侵略性设置最初设置为1，但如果他选择民主政府，他将获得-2的侵略性修正值（将他当前的侵略性值降低2）。这会让他的攻击力溢出到255，让他的攻击力达到最大！然而，最近席德梅尔（游戏作者）澄清事实并非如此。


### std::size_t

std::size_t is defined as an unsigned integral type, and it is typically used to represent the size or length of objects.

Much like an integer can vary in size depending on the system, `std::size_t` also varies in size. `std::size_t` is guaranteed to be unsigned and at least 16 bits, but on most systems will be equivalent to the address-width of the application. That is, for 32-bit applications, `std::size_t` will typically be a 32-bit unsigned integer, and for a 64-bit application, `std::size_t` will typically be a 64-bit unsigned integer.


### ⚠️注意

1. 慎用 Unsigned types for holding quantities。
2. 慎用 The 8-bit fixed-width integer types，因为8-bit int 通常被视为字符而不是整数值（因系统而异）。
	
	In cases where `std::int8_t` is treated as a char, input from the console can also cause problems:
	```cpp
	#include <cstdint>
	#include <iostream>

	int main()
	{
	    std::cout << "Enter a number between 0 and 127: ";
	    std::int8_t myint{};
	    std::cin >> myint;

	    std::cout << "You entered: " << static_cast<int>(myint) << '\n';
	
	    return 0;
	}
	```
	
	A sample run of this program:
	```
	Enter a number between 0 and 127: 35
	You entered: 51
	```

	Here’s what’s happening. When `std::int8_t` is treated as a char, the input routines interpret our input as a sequence of characters, not as an integer. So when we enter `35`, we’re actually entering two chars, `'3'` and `'5'`. Because a char object can only hold one character, the `'3'` is extracted (the `'5'` is left in the input stream for possible extraction later). Because the char `'3'` has ASCII code point 51, the value `51` is stored in `myint`, which we then print later as an int.


