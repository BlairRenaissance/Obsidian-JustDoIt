
主流的 C/C++ 工具链 (Toolchain) 及其特点、适用场景和平台的总结：

具体实践可以参考：[[运行时库#3. 各平台的C/C++运行时库]]

---
## GNU 工具链

- 核心组件：
	- 编译器：`gcc`（C）、`g++`（C++）
		一般文档中提到 `gcc` 包含 `gcc++
	- 链接器：`ld`（GNU Binutils）
	- 标准库：`glibc`（C）、`libstdc++`（C++）
	- 其他工具：`ar`（静态库）、`objdump`（反汇编）、`gdb`（调试器）
- 平台： 
	Linux、Unix-like 系统为主，支持跨平台（通过 MinGW 或 Cygwin 在 Windows 运行）。
- 特点：
	- 开源、高度兼容 C/C++ 标准（支持最新草案）。
	- 优化能力强，适合服务器、嵌入式和高性能计算。
	- 与 Linux 生态深度绑定（如内核编译依赖 GCC）。

---
## LLVM/Clang 工具链

- 核心组件：
	- 编译器：`clang`（C/C++/Objective-C）
		`clang`（C）、`clang++`（C++）、一般文档中提到 `clang` 包含 `clang++`
	- 链接器：`lld`（LLVM 链接器）
	- 标准库：`libc++`（C++）、`libc`（实验性 C 库）
	- 其他工具：`llvm-ar`、`llvm-objdump`、`lldb`（调试器）
- 平台：
	跨平台（Windows、Linux、macOS、嵌入式等）。
- 特点：
	- 模块化设计，编译速度快，错误提示友好。
	- 支持代码静态分析（Clang-Tidy）、代码格式化（Clang-Format）。
	- 苹果生态默认工具链（Xcode 使用 Clang 编译 macOS/iOS 应用）。

---
## Microsoft 工具链（MSVC）

- 核心组件：
	- 编译器：`cl.exe`（将 C/C++ 源码编译为 Windows 平台的原生PE格式二进制文件，如 .exe, .dll, .obj）。
	- 链接器：`link.exe`（将目标文件 .obj 和库文件 .lib 链接为最终可执行文件或动态库。
	- 标准库：`MSVC CRT`（C）、`MSVC STL`（C++）
	- 其他工具：`lib.exe`（静态库）、Windows SDK 集成
- 平台：
	Windows 原生开发，支持跨平台（需配合其他工具链）。
- 特点：
	- 深度集成 Windows API 和 Visual Studio 生态。
	- 调试工具强大（如 Visual Studio 调试器）。
	- 闭源，但提供免费社区版。

---
## 嵌入式专用工具链

- 示例：
	- ARM GCC（arm-none-eabi）：用于 ARM Cortex-M/R/A 系列芯片开发。
	- RISC-V GCC（riscv64-unknown-elf）：RISC-V 架构嵌入式开发。
	- Keil MDK：ARM 单片机开发（含编译器、IDE、调试器）。
- 特点：
	- 针对特定硬件架构优化，支持裸机（Bare Metal）或 RTOS。
	- 通常包含交叉编译器（Cross-Compiler）和硬件调试工具。

---
 
如何选择工具链？

1. 平台优先：
	Windows 开发首选 MSVC，Linux/macOS 优先 GCC/Clang。
2. 性能需求：
	高性能计算可选 Intel ICC，通用场景选 GCC/Clang。
3.开发体验：
	Clang 适合需要友好错误提示和代码分析的项目。