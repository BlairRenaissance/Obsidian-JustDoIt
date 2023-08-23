
C#语言中的所有类型都是由基类System.Object继承过来的，包括最常用的基础类型：int, byte, short，bool等等，就是说所有的事物都是对象。如果申明这些类型时都在堆(HEAP)中分配，会造成极低的效率。.NET如何解决这个问题？正是通过将类型分成值型(value)和引用型(referencetype)，C#中定义的值类型包括原类型（`Sbyte、Byte、Short、Ushort、Int、Uint、Long、ulong、Char、Float、Double、Bool、Decimal`）、枚举(`enum`)、结构(`struct`)，引用类型包括：类、数组、接口、委托、字符串等。

值型就是在栈中分配内存，在申明的同时就初始化，**以确保数据不为null**； 引用型是在堆中分配内存，初始化为null，引用型是需要GARBAGE COLLECTION来回收内存的，值型不用，超出了作用范围，系统就会自动释放。

装箱
装箱就是隐式的将一个值型转换为引用型对象。
In the following example, the integer variable `i` is _boxed_ and assigned to object `o`.
```C#
int i = 123;
// The following line boxes i.
object o = i;
```

拆箱
拆箱就是将一个引用型对象转换成任意值型。
The object `o` can then be unboxed and assigned to integer variable `i`:

```C#
o = 123;
i = (int)o;  // unboxing
```

值类型和引用型有什么区别呢？

值类型的变量包含自身的数据，而引用类型的变量是指向数据的内存块的，并不是直接存放数据。对于值类型，每个变量都有一份自己的数据复制，对另一个值类型变量的操作并不影响这一个变量的值。而对于引用类型，两个变量有可能引用同一对象，因此对一个变量的操作会影响到另一个变量。