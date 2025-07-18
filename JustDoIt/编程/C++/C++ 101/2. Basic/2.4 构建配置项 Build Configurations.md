# 构建类型

When you create a new project in your IDE, most IDEs will set up two different build configurations for you: a **release configuration**, and a **debug configuration**.

---

**For Visual Studio users**

There are multiple ways to switch between _debug_ and _release_ in Visual Studio. The easiest way is to set your selection directly from the _Solution Configurations_ dropdown in the _Standard Toolbar Options_:

![VS Solution Configurations Dropdown](https://www.learncpp.com/images/CppTutorial/Chapter0/VS-BuildTarget-min.png?ezimgfmt=rs:873x66/rscb2/ng:webp/ngcb2)

Set it to _Debug_ for now.

You can also access the configuration manager dialog by selecting _Build menu > Configuration Manager_, and change the _active solution configuration_.

To the right of the _Solutions Configurations_ dropdown, Visual Studio also has a _Solutions Platform_ dropdown that allows you to switch between x86 (32-bit) and x64 (64-bit) platforms.

---

**For gcc and Clang users**

Add `-ggdb` to the command line for debug builds and `-O2 -DNDEBUG` for release builds. Use the debug build options for now.

For GCC and Clang, the `-O#` option is used to control optimization settings. The most common options are as follows:

- `-O0` is the recommended optimization level for debug builds, as it disables optimization. This is the default setting.
- `-O2` is the recommended optimization level for release builds, as it applies optimizations that should be beneficial for all programs.
- `-O3` adds additional optimizations that may or may not perform better than `-O2` depending on the specific program. Once your program is written, you can try compiling your release build with `-O3` instead of `-O2` and measure to see which is faster.

See [https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html) for information on optimization options.

---

**For VS Code users**

When you first ran your program, a new file called _tasks.json_ was created under the _.vscode_ folder in the explorer pane. Open the _tasks.json_ file, find _“args”_, and then locate the line _“${file}”_ within that section.

Above the _“${file}”_ line, add a new line containing the following command (one per line) when debugging:  
`"-ggdb",`

Above the _“${file}”_ line, add new lines containing the following commands (one per line) for release builds:  
`"-O2",`  
`"-DNDEBUG",`


# 语言版本选择

**Setting a language standard in Visual Studio**

As of the time of writing, Visual Studio 2022 defaults to C++14 capabilities, which does not allow for the use of newer features introduced in C++17 and C++20.

To use these newer features, you’ll need to enable a newer language standard. Unfortunately, there is currently no way to do this globally -- **you must do so on a project-by-project basis.**

> Warning ⚠️
> With Visual Studio, you will need to reselect your language standard every time you create a new project.

To select a language standard, open your project, then go to _Project menu > (Your application’s Name)_ Properties, then open _Configuration Properties > C/C++ > Language_.

![](https://www.learncpp.com/images/CppTutorial/Chapter0/VS2019-Project-Language-min.png?ezimgfmt=rs:838x597/rscb2/ng:webp/ngcb2)

First, make sure the _Configuration_ is set to “All Configurations”.

Make sure you’re selecting the language standard from the dropdown menu (don’t type it out).

---

**Setting a language standard in GCC/G++/Clang**

For GCC/G++/Clang, you can use compiler options `-std=c++11`, `-std=c++14`, `-std=c++17`, `-std=c++20`, or `-std=c++23` (to enable C++11/14/17/20/23 support respectively). If you have GCC 8 or 9, you’ll need to use `-std=c++2a` for C++20 support instead. You can also try the latest code name (e.g. `-std=c++2c`) for experimental support for features from the upcoming language standard.

---

**Setting a language standard for VS Code**

For VS Code, you can follow the rules above for setting a language standard in GCC/G++/Clang.

Place the appropriate language standard flag (including the double quotes and comma) in the `tasks.json` configuration file, in the `"args"` section, on its own line before `"${file}"`.

We also want to configure Intellisense to use the same language standard. For C++20, in `settings.json`, change or add the following setting on its own line: `"C_Cpp.default.cppStandard": "c++20"`.

---

**使用CMake设置**

如果你的项目使用 CMake 构建系统，可以在 `CMakeLists.txt` 文件中查看和设置 C++ 标准版本。

 在 Android Studio 的项目视图中，找到并打开 `CMakeLists.txt` 文件。

查找 `set(CMAKE_CXX_STANDARD ...)` 指令。如果没有找到，可以添加它。例如：

```CMake
set(CMAKE_CXX_STANDARD 11)  # 使用 C++11 标准
```