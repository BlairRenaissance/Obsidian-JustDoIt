
> [!summary]
> The entire process of preprocessing, compiling, and linking is called **translation**.  Here is a list of [translation phases](https://en.cppreference.com/w/cpp/language/translation_phases).

## Translation phases

Translation is performed [as if](https://en.cppreference.com/w/cpp/language/as_if "cpp/language/as if") in the order from phase 1 to phase 9. 

Implementations behave as if these separate phases occur, although in practice different phases can be folded together. 尽管在实际操作中，不同的阶段可能会合并在一起，但实现仍然表现得好像这些独立的阶段确实发生了一样。

> [!Info] The as-if rule
> Allows any and all code transformations that do not change the observable behavior of the program.
> 
> 在C++中，“as-if rule”（仿佛规则）是一个重要的优化原则。根据这个规则，编译器可以自由地对代码进行任何形式的优化，只要这些优化不会改变程序的可观察行为。换句话说，编译器可以对代码进行重排、合并或删除等优化操作，只要最终生成的程序在行为上与未优化的程序表现一致。
> 具体来说，“as-if rule”允许编译器进行以下优化：
> 1. **代码重排**：编译器可以改变代码的执行顺序，只要这种改变不会影响程序的最终结果。
> 2. **消除冗余**：编译器可以删除不必要的代码，只要删除这些代码不会影响程序的行为。
> 3. **内联函数**：编译器可以将函数调用替换为函数体的实际代码，以减少函数调用的开销。
> 
> 这些优化的前提是，程序的可观察行为（如输出结果、修改的全局变量、与外部系统的交互等）必须保持不变。

### Phase 1: Mapping source characters 字符映射

将源代码中的字符转换为编译器或预处理器能够理解的内部表示形式。这通常涉及字符集的转换，例如从源文件的字符编码转换为编译器使用的翻译字符集元素 [translation character set](https://en.cppreference.com/w/cpp/language/charset#Translation_character_set "cpp/language/charset")。

| Detail                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Version       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| 1) 源代码文件中的每个字节都被映射到翻译字符集元素 [translation character set](https://en.cppreference.com/w/cpp/language/charset#Translation_character_set "cpp/language/charset")。源代码文件的字节映射方式由具体实现定义(implementation-defined)。操作系统相关的行结束指示符被替换为换行符。<br><br>2) The set of source file characters accepted is implementation-defined(since C++11). 自C++11起，源文件中接受的字符集是实现定义的。任何无法映射到基本源字符集的源文件字符将被替换为其通用字符名（使用`\u`或`\U`进行转义）。<br><br>3) Trigraph sequence are replaced by corresponding single-character representations. (until C++17)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | (until C++23) |
| Input files that are a sequence of UTF-8 code units (UTF-8 files) are guaranteed to be supported. The set of other supported kinds of input files is implementation-defined. If the set is non-empty, the kind of an input file is determined in an implementation-defined manner that includes a means of designating input files as UTF-8 files, independent of their content (recognizing the byte order mark is not sufficient). 一定支持UTF-8编码的输入文件。如果同时支持其他类型的输入文件，那么确定输入文件类型的方法也是由具体实现定义的。而这个方法必须包括一种能够将输入文件指定为UTF-8文件的方式，且这种指定方式不能仅仅依赖于文件内容（例如，不能仅仅依赖于字节顺序标记（byte order mark, BOM）来识别UTF-8文件）。<br><br>- **UTF-8文件处理**：<br>    - 确定为UTF-8文件后，文件必须是格式正确的UTF-8编码字节序列。<br>    - 解码后生成Unicode标量值序列。<br>    - 将Unicode标量值映射到翻译字符集元素 [translation character set](https://en.cppreference.com/w/cpp/language/charset#Translation_character_set "cpp/language/charset")。<br>    - 替换输入序列中的回车和换行组合以及单独的回车为新行字符。<br>	<br>- **其他类型的输入文件处理**：<br>    - 对于其他类型的输入文件，字符映射到翻译字符集元素的方式由具体实现定义。<br>    - 操作系统相关的行结束标志（例如Windows的CRLF、Unix的LF等）将被替换为新行字符。 | (since C++23) |
- **实现定义的字符集**：编译器可以决定接受哪些字符集，这可能包括各种字符编码（如UTF-8、ISO-8859-1等）。不同的编译器可能支持不同的字符集。
- **通用字符名**：通用字符名是一种标准的字符表示形式，使用`\u`或`\U`进行转义。例如，Unicode字符`U+00A9`可以表示为`\u00A9`。这种表示方式确保了字符在不同环境下的一致性。
- **三字母序列**（trigraph sequences）：C++语言中的一种字符序列，由三个问号（`??`）开头，后跟一个特定的字符，用来表示一些无法直接输入的字符。如源代码中包含`??=define`, 编译器会将其替换为`#define`。三字母序列在C++17标准中已经被弃用，但在之前的标准中仍然有效。

### Phase 2: Splicing lines 行拼接

1) If the first translation character is byte order mark (U+FEFF) , it is deleted. (since C++23)

2)  combining two physical source lines into one logical source line. 行末尾反斜杠(`\`)表示是同一行，在这里完成两个物理行到一个逻辑行到拼接工作。

3) If a non-empty source file does not end with a newline character after this step (end-of-line backslashes are no longer splices at this point), a terminating newline character is added. 添加新行符。

### Phase 3: Lexing 词法分析

词法分析 (lexing) 接受字符串并将其转换成 tokens 流 (streams of tokens)，这些 token 是编程语言的基本语法单元，例如关键字、标识符、操作符、字面量等。

1) The source file is decomposed into [preprocessing tokens](https://en.cppreference.com/w/cpp/language/translation_phases#Preprocessing_tokens) and whitespace.

```
// The following #include directive can de decomposed into 5 preprocessing tokens:
 
//     punctuators (#, < and >) 
//          │
// ┌────────┼────────┐
// │        │        │
   #include <iostream>
//     │        │
//     │        └── header name (iostream)
//     │
//     └─────────── 预处理指令
```

2) 双引号之间的内容，被 Phase 1 改变的在此刻被还原；C++ 23 后 Phase 2 改变的内容也会被还原。

3) 空白字符的处理规则：
	- 注释被替换为一个空格字符。
	- 新行字符被保留。
	- 除了新行字符之外的其他空白字符序列是否被保留或替换为一个空格字符是未指定的，由具体实现决定。

#### Preprocessing Tokens

> [!Info] Preprocessing tokens
> A _preprocessing token_ is the minimal lexical element of the language in translation phases 3 through 6.
> The categories of preprocessing token are:
> - header names (such as `<iostream>` or `"myfile.h"`)
> - placeholder tokens produced by preprocessing import and module directives (i.e. `import XXX;` and `module XXX;`) (since C++20)
> - identifiers （标识符，命名变量、函数、类、模块等的名称）
> - preprocessing numbers （预处理数字）
> - character literals, including user-defined character literals(since C++11) 
> - string literals, including user-defined string literals(since C++11)
> - operators and punctuators, including [alternative tokens](https://en.cppreference.com/w/cpp/language/operator_alternative "cpp/language/operator alternative") （操作符如 `+ - * /` 标点符号如 `; , ()`）
> - individual non-whitespace characters that do not fit in any other category （如关键字 `int`）

| 符号         | 类型    | 词法分析阶段 Token 类型 |
| ---------- | ----- | --------------- |
| `int`      | 关键字   | keyword         |
| `main`     | 标识符   | identifier      |
| `#include` | 预处理指令 | 不参与词法分析         |

### Phase 4: Preprocessing 预处理

1) The [preprocessor](https://en.cppreference.com/w/cpp/preprocessor "cpp/preprocessor") is executed. 预处理器执行，预处理器会处理源代码中的预处理指令，例如宏定义 (`#define`)、条件编译 (`#if`, `#ifdef`, `#ifndef` 等)、文件包含 (`#include`) 等。

2) Each file introduced with the `#include` directive goes through phases 1 through 4, recursively. 递归地处理 `#include`。

3) At the end of this phase, all preprocessor directives are removed from the source. 在预处理阶段结束时，所有的预处理指令都会被移除。这意味着源代码中不会再有 `#include`, `#define`, `#if` 等指令，因为它们已经被处理并替换为相应的代码或被移除。
