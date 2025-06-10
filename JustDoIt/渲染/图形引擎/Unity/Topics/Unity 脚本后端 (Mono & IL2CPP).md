在 Unity 中，可以选择使用 IL2CPP 或 Mono 作为脚本后端。这种选择通常是在构建设置中进行的。

## 设置Unity脚本后端

### 步骤

1. **打开构建设置**：
    - 在 Unity 编辑器中，点击菜单栏中的 `File`（文件） > `Build Settings...`（构建设置）。
2. **选择目标平台**：
    - 在构建设置窗口中，选择您要构建的目标平台（例如，Windows, Mac, Linux, iOS, Android 等）。
3. **打开播放器设置**：
    - 在构建设置窗口中，点击右下角的 `Player Settings...` 按钮。这将打开 `Player Settings` 窗口。
4. **选择脚本后端**：
    - 在 `Player Settings` 窗口中，展开 `Other Settings`（其他设置）部分。
    - 找到 `Configuration`（配置）部分，您会看到一个名为 `Scripting Backend`（脚本后端）的选项。
    - 点击 `Scripting Backend` 下拉菜单，您可以选择 `Mono` 或 `IL2CPP` 作为脚本后端。

### 注意事项

- **平台支持**：并非所有平台都支持 IL2CPP 和 Mono。例如，iOS 平台通常只支持 IL2CPP，而不支持 Mono。
- **性能和兼容性**：IL2CPP 通常提供更好的性能和安全性，但构建时间可能会更长。Mono 则提供更快的构建时间和更好的调试支持。
- **调试**：在开发和调试阶段，可能更倾向于使用 Mono，因为它提供了更好的调试支持和更快的构建时间。在发布阶段，可能更倾向于使用 IL2CPP 以获得更好的性能和安全性。

通过在 Unity 的构建设置中选择脚本后端，可以轻松地在 IL2CPP 和 Mono 之间切换。它们在调试支持方面有一些显著的差异。

---
## Mono

Mono 是一个开源的跨平台的 .NET 运行时实现。传统的.NET Framework仅支持Windows平台，Mono的目标是让 .NET 语言编写的程序能在 Windows 以外的平台（Linux、macOS、iOS、Android 等）运行。

它实现了 .NET 的公共语言运行时程序（CLR，Common Language Runtime）和基础类库（BCL，Base Class Library），并且支持 C# 等多种 .NET 语言。

### 1. Mono 的核心组成部分

Mono 的架构可以分为以下几个主要模块：

| 模块名称               | 作用描述                                                              |
| ------------------ | ----------------------------------------------------------------- |
| **编译器（C# 编译器）**    | 将 C# 源代码编译成中间语言（IL，Intermediate Language）字节码。                     |
| **类库（BCL）**        | 实现了 .NET 标准库的基础类，如 `System` 命名空间下的类型，提供字符串、集合、IO、网络等功能。           |
| **公共语言运行时程序（CLR）** | 这个运行时程序就是一个虚拟机(VM)，负责执行 IL 代码，管理内存、线程、安全、异常处理等。包括 JIT 编译器、垃圾回收器等。 |
| **平台适配层**          | 负责调用底层操作系统 API，屏蔽不同平台的差异。                                         |
| **工具链**            | 包括调试器、配置工具、AOT 编译器等辅助工具。                                          |

---
### 2. 详细架构层次

#### 2.1 C# 编译器（mcs）

- Mono 自带的 C# 编译器（mcs）将 C# 代码编译成标准的 .NET IL 字节码。
- 生成的 IL 是平台无关的，可以在任何支持 CLR 的环境中运行。

#### 2.2 公共语言运行时程序（CLR）

CLR 是 Mono 的核心，负责执行 IL 代码。它包含：

- **解释器**：可以直接解释执行 IL 代码（性能较低，主要用于调试）。
- **JIT 编译器**（Just-In-Time）：在运行时将 IL 编译成当前平台的机器码，提高执行效率。
- **垃圾回收器（GC）**：自动管理内存，回收不再使用的对象，避免内存泄漏。Mono 使用的是标记-清除和分代回收算法。
- **线程管理**：支持多线程，调度线程执行。
- **异常处理**：实现 .NET 异常机制，支持 try/catch/finally。
- **安全机制**：代码访问安全（CAS）等。

#### 2.3 基础类库（BCL）

- Mono 实现了大量 .NET 标准库的类，保证大部分 .NET 应用可以无修改运行。
- 包括 `System`, `System.Collections`, `System.IO`, `System.Net` 等。
- 这些类库调用底层平台 API 来实现功能。

#### 2.4 平台适配层

- Mono 通过 P/Invoke（平台调用）机制调用本地操作系统的 API。
- 例如文件操作、网络通信、线程同步等功能，Mono 会调用对应平台的系统调用。
- 这部分代码是 Mono 跨平台的关键，针对不同操作系统实现不同的适配代码。

#### 2.5 AOT 编译器（Ahead-Of-Time）

- 在某些平台（如 iOS）不允许 JIT 编译，Mono 提供 AOT 编译器，将 IL 预先编译成机器码。
- 这样可以绕过 JIT 限制，保证应用能运行。

---
### 3. Mono 的执行流程示意

1. **编译阶段**：C# 代码用 mcs 编译成 IL 字节码。
2. **加载阶段**：Mono 运行时加载程序集（.dll 或 .exe 文件）。
3. **执行阶段**：
    - JIT 编译器将 IL 转换成本地机器码。
    - 机器码在 CPU 上执行。
    - 运行时管理内存、线程和异常。
4. **调用本地 API**：通过 P/Invoke 调用操作系统底层功能。

---
### 4. Mono 的跨平台关键点

| 关键点             | 说明                                            |
| --------------- | --------------------------------------------- |
| **IL 字节码平台无关**  | IL 是中间语言，不依赖具体硬件或操作系统。                        |
| **JIT 编译器针对平台** | Mono 的 JIT 编译器针对不同 CPU 架构（x86, ARM 等）生成对应机器码。 |
| **平台适配层**       | 通过调用不同平台的系统 API，实现文件、网络、线程等功能的跨平台支持。          |
| **AOT 支持**      | 在不支持 JIT 的平台（如 iOS）使用 AOT 编译，保证兼容性。           |
| **基础类库兼容**      | 实现了大部分 .NET 标准库，保证代码可移植性。                     |

---
### 如何系统学习和理解 Mono 与 Unity 的关系？

1. **先理解 .NET 和 Mono 的基础**
    - 学习 .NET CLR 的工作原理，理解 IL、JIT、GC 等核心概念。
    - 阅读 Mono 项目文档，了解它如何实现 .NET 运行时。
2. **学习 Unity 脚本系统
    - 了解 Unity 如何使用 Mono 运行时加载和执行 C# 脚本。
    - 理解 MonoBehaviour 生命周期、脚本编译流程。
3. **结合 Unity 的源码和技术分享**
    - 通过社区资源、开源项目、技术博客，了解 Unity 如何集成 Mono。
    - 关注 Unity 近年来向 IL2CPP 迁移的背景和原因。

---
## IL2CPP

研究il2cpp的方法挺简单的：在Window下创建一个Unity工程，设置`Scripting Backend`为`il2cpp`，然后构建把`Create Visual Studio Solution`勾选上，就可生产一个vc工程，il2cpp的一切秘密尽在这个vc工程，断点调试即可分析il2cpp是怎么运作的。

### 1. 什么是 IL2CPP？

- **IL2CPP** 是 Unity 自研的一套技术，全称是 **Intermediate Language To C++**。
- 它的核心思想是：
    1. 把 C# 脚本先编译成中间语言（IL，Intermediate Language），
    2. 然后将 IL 转换成 C++ 代码，
    3. 最后用平台的 C++ 编译器编译成机器码。
- 这样生成的程序是**原生二进制**，不依赖 Mono 虚拟机。

---

### 2. Unity 为什么要从 Mono 过渡到 IL2CPP？

**Mono 的局限性**

- **性能瓶颈**：Mono 运行时采用 JIT（即时编译），运行时会有一定开销，尤其在移动设备和主机平台上性能有限。
- **平台限制**：部分平台（如 iOS）不允许 JIT 编译，必须使用 AOT（Ahead-Of-Time，提前编译）技术。Mono 的 AOT 支持有限。
- **安全性和代码保护**：Mono 运行时的 IL 代码相对容易被反编译，安全性较弱。
- **维护和兼容性**：Mono 是第三方开源项目，Unity 需要自己维护和适配，增加了复杂度。

**IL2CPP 的优势**

- **性能提升**：生成的 C++ 代码经过本地编译，运行效率更高，启动速度更快。
- **跨平台支持更好**：支持所有 Unity 支持的平台，包括 iOS、主机（PS、Xbox）、WebGL 等。
- **安全性更强**：生成的二进制代码不包含 IL，反编译难度大大增加。
- **更好的内存管理和调试支持**：Unity 可以更灵活地控制内存分配和调试流程。
- **统一的编译流程**：减少了对 Mono 版本和平台差异的依赖，简化了发布流程。

---
## IL2CPP vs Mono

| 特性    | Mono                   | IL2CPP                    |
| ----- | ---------------------- | ------------------------- |
| 编译方式  | JIT（即时编译）+ 部分 AOT 支持   | 完全 AOT（IL 转 C++，再编译成本地代码） |
| 性能    | 较好，但受限于 JIT 开销         | 更高，接近原生 C++ 性能            |
| 跨平台支持 | 支持多平台，但 iOS 等平台限制较多    | 支持所有 Unity 平台，尤其是 iOS 和主机 |
| 代码安全  | IL 代码易被反编译             | 生成的 C++ 二进制难以反编译          |
| 启动速度  | 较慢，JIT 需要时间            | 快，提前编译成机器码                |
| 维护复杂度 | 依赖 Mono 项目，需跟进 Mono 版本 | Unity 自研，统一维护             |
| 调试体验  | 支持丰富的调试功能              | 调试相对复杂，但 Unity 持续改进中      |
#### 1. 即时编译（JIT） vs. 静态编译（AOT）

- **Mono**：使用即时编译（JIT），这意味着代码在运行时被编译。这种方式允许更灵活的调试，因为可以在运行时动态加载和修改代码。
- **IL2CPP**：使用静态编译（AOT），将 C# 代码转换为 C++ 代码，然后再编译为机器码。这种方式在运行时不支持动态代码生成和修改，因此调试时的灵活性较低。

#### 2. 调试器支持

- **Mono**：由于其 JIT 特性，Mono 提供了更强大的调试器支持。开发者可以使用 Visual Studio、Rider 或 MonoDevelop 等 IDE 进行调试，支持设置断点、单步执行、查看变量值、修改变量值等功能。
- **IL2CPP**：由于其 AOT 特性，IL2CPP 的调试支持相对有限。虽然 Unity 提供了一些调试工具，但功能和灵活性不如 Mono。特别是在调试复杂的运行时行为和动态代码时，IL2CPP 的调试能力较弱。

#### 3. 编辑器中的即时反馈

- **Mono**：在使用 Mono 作为脚本后端时，Unity 编辑器可以提供更即时的反馈。由于代码在运行时被编译，开发者可以快速地进行代码修改并立即看到效果。这对于快速迭代和调试非常有帮助。
- **IL2CPP**：由于需要将 C# 代码转换为 C++ 代码并进行编译，使用 IL2CPP 时的构建时间较长。这意味着每次代码修改后，开发者需要等待更长的时间才能看到效果，不利于快速迭代和调试。

#### 4. 热重载和热替换

- **Mono**：支持热重载和热替换功能，允许开发者在不停止应用程序的情况下修改代码并立即应用更改。这对于调试和开发过程中的快速迭代非常有帮助。
- **IL2CPP**：由于其 AOT 特性，不支持热重载和热替换功能。每次代码修改后都需要重新编译和重启应用程序，增加了调试和开发的复杂性。

#### 5. 调试信息的可用性

- **Mono**：在调试时，Mono 提供了丰富的调试信息，包括详细的堆栈跟踪、变量值和类型信息等。这些信息对于定位和解决问题非常有帮助。
- **IL2CPP**：由于代码被转换为 C++ 并进行编译，调试信息相对较少。虽然 Unity 提供了一些工具来帮助调试 IL2CPP 生成的代码，但这些工具的功能和信息量不如 Mono。

#### 总结

尽管 IL2CPP 在性能和安全性方面具有优势，但在调试支持方面，Mono 提供了更好的体验。Mono 的即时编译特性、强大的调试器支持、编辑器中的即时反馈、热重载和热替换功能以及丰富的调试信息，使得开发者在调试和快速迭代时更加高效。因此，在开发和调试阶段，许多开发者更倾向于使用 Mono 作为脚本后端，而在发布阶段则可能选择 IL2CPP 以获得更好的性能和安全性。

## 反射

反射的主要作用：
- 运行时获取类型信息
- 动态调用

#### 运行时获取类型信息

无论何种语言的反射，都依赖编译器或者工具（比如unity的il2cpp和ue的UHT）在编译期把类型信息的元数据，以某种形式存起来，而反射api实际上是对这些元数据的检索api。所以分析反射实现的重点是分析这些元数据是如何存放和组织的。

**System.Reflection.Type**

System.Reflection.Type的il2cpp实现有几个很重要的数据结构：Il2CppReflectionType、Il2CppType、Il2CppClass、Il2CppTypeDefinition

System.Reflection.Type对象在生成代码是Type_t，但在libil2cpp里会强转为Il2CppReflectionType（两者成员上是完全一样的），所以C#的Type就是il2cpp里的Il2CppReflectionType，它的定义:

```javascript
typedef struct Il2CppReflectionType
{
    Il2CppObject object;
    const Il2CppType *type;
} Il2CppReflectionType;
```

il2cpp对象继承采用c风格实现，`Il2CppObject object`成员表示基类为System.Object。（ps：无独有偶，il2cpp的this指针也是显式的参数，虚函数调用也没用c++的，而是用指针数组来模拟）

其属于自己的信息就一个Il2CppType（别名RuntimeType）指针，该类型的定义如下：

```javascript
typedef struct Il2CppType
{
    union
    {
        // We have this dummy field first because pre C99 compilers (MSVC) can only initializer the first value in a union.
        void* dummy;
        TypeDefinitionIndex __klassIndex; /* for VALUETYPE and CLASS at startup */
        Il2CppMetadataTypeHandle typeHandle; /* for VALUETYPE and CLASS at runtime */
        const Il2CppType *type;   /* for PTR and SZARRAY */
        Il2CppArrayType *array; /* for ARRAY */
        //MonoMethodSignature *method;
        GenericParameterIndex __genericParameterIndex; /* for VAR and MVAR at startup */
        Il2CppMetadataGenericParameterHandle genericParameterHandle; /* for VAR and MVAR at runtime */
        Il2CppGenericClass *generic_class; /* for GENERICINST */
    } data;
    unsigned int attrs    : 16; /* param attributes or field flags */
    Il2CppTypeEnum type     : 8;
    unsigned int num_mods : 5;  /* max 64 modifiers follow at the end */
    unsigned int byref    : 1;
    unsigned int pinned   : 1;  /* valid when included in a local var signature */
    unsigned int valuetype : 1;
    //MonoCustomMod modifiers [MONO_ZERO_LEN_ARRAY]; /* this may grow */
} Il2CppType;
```

里面并没有我们所想象的类型元数据，比如最简单的类型名。

我们简单写个`typeof(Vector3).Name`然后断点跟进去。可以看到它的实现是通过vm::Class::FromIl2CppType获取Il2CppType对应的Il2CppClass，然后我们可以看到类型名原来存放在Il2CppClass上的name字段。再看下Il2CppClass，可以看到类型元数据全在这个类。

