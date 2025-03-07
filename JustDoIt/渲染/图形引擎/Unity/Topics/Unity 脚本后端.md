在 Unity 中，您可以选择使用 IL2CPP 或 Mono 作为脚本后端。这种选择通常是在构建设置中进行的。以下是如何在 Unity 中设置脚本后端的步骤：

## IL2CPP

### 深入浅出il2cpp：反射篇

研究il2cpp的方法挺简单的：在Window下创建一个Unity工程，设置`Scripting Backend`为`il2cpp`，然后构建把`Create Visual Studio Solution`勾选上，就可生产一个vc工程，il2cpp的一切秘密尽在这个vc工程，断点调试即可分析il2cpp是怎么运作的。

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

## 设置Unity脚本后端

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