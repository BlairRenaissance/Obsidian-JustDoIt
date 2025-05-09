# 编译器

[[工具链 Toolchain]]

| 编译器                  | 主要平台      | 所属工具链 | 编译语言  |
| -------------------- | --------- | ----- | ----- |
| gcc                  | Linux     | GNU   | C     |
| g++                  | Linux     | GNU   | C++   |
| Clang                | Mac / 跨平台 | LLVM  | C/C++ |
| Visual Studio cl.exe | Windows   | MSVC  | C/C++ |
编译器最终生成的二进制文件可以是以下几种类型：

- **可执行文件**：可以直接运行的程序文件（如 `.exe`、`.out` 等）。
- **目标文件**：包含机器代码的中间文件，通常需要进一步链接（如 `.o`、`.obj` 文件）。
- **共享库**：动态链接库，可以在运行时加载和使用（如 `.dll`、`.so`、`.dylib` 文件）。

# 二进制文件

不同平台的库文件的二进制格式是不一样的，文件名后缀也不一样。
[[运行时库#3.3 库的文件格式]]

| 文件格式   | 适用平台          | 动态库名                  | 静态库名              | 主程序文件名 |
| ------ | ------------- | --------------------- | ----------------- | ------ |
| ELF    | Linux、Android | .so                   | .a                | 无后缀    |
| PE     | Windows       | .dll                  | .lib              | .exe   |
| Mach-O | macOS、iOS     | .dylib或<br>.framework | .a或<br>.framework | 无后缀    |
注：
  ELF：Executable and Linkable Format
  PE：Portable Executable
  Mach-O：Mach Object file format

ELF、PE 和 Mach-O 都是二进制文件格式，用于在不同操作系统上存储可执行文件、目标代码、共享库等。以下是对每种格式的简要介绍：

### ELF

- **全称**：Executable and Linkable Format
- **使用平台**：主要用于 Unix 和类 Unix 系统，如 Linux、BSD 等。
- **用途**：用于存储可执行文件、共享库、核心转储和目标代码。
- **特点**：
    - 灵活且可扩展。
    - 支持多种处理器架构。
    - 包含多个段和节，用于存储代码、数据、符号表等。

### PE

- **全称**：Portable Executable
- **使用平台**：主要用于 Windows 操作系统。
- **用途**：用于存储可执行文件（.exe）、动态链接库（.dll）、对象文件（.obj）和其他文件类型。
- **特点**：
    - 基于 COFF（Common Object File Format）。
    - 包含 DOS 头、PE 头、节表和多个节（如代码节、数据节等）。
    - 支持 Windows 特定的功能，如资源表、导入表和导出表。

### Mach-O

- **全称**：Mach Object
- **使用平台**：主要用于 macOS 和 iOS 操作系统。
- **用途**：用于存储可执行文件、动态库（.dylib）、内核扩展（.kext）和目标代码。
- **特点**：
    - 设计灵活，支持多种处理器架构。
    - 包含头部、加载命令和多个段。
    - 支持 macOS 和 iOS 特定的功能，如 fat binaries（多架构二进制文件）。

### 总结

ELF、PE 和 Mach-O 都是用于不同操作系统的二进制文件格式。它们的主要用途是存储可执行文件、共享库和目标代码，但它们在结构和特性上有所不同，以适应各自操作系统的需求。

- **ELF**：主要用于 Unix 和类 Unix 系统，如 Linux。
- **PE**：主要用于 Windows 系统。
- **Mach-O**：主要用于 macOS 和 iOS 系统。

每种格式都有其独特的设计和功能，以支持其目标平台的特定需求。