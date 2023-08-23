
When you compile your project, you might expect that the compiler compiles each code file exactly as you’ve written it. This actually isn’t the case.

在编译之前，每个代码（.cpp）文件都会经历一个预处理阶段。在这个阶段，一个称为预处理器的程序会对代码文件的文本进行各种更改。预处理器实际上并不会修改原始的代码文件，而是通过在内存中临时进行更改或使用临时文件来完成所有预处理所需的更改。

在历史上，预处理器是编译器的一个单独程序，但在现代编译器中，预处理器可能被直接构建到编译器本身中。

预处理器所做的大部分工作都是相当无趣的。例如，它删除注释，并确保每个代码文件以换行符结尾。然而，预处理器确实有一个非常重要的作用：它处理include和define指令。

When the preprocessor has finished processing a code file, the result is called a translation unit. This translation unit is what is then compiled by the compiler.











