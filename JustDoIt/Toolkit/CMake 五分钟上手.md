## 5分钟理解gcc/make/makefile/CMake

**1. gcc**

它是GNU Compiler Collection。GNU 可以看作一个开源平台，包含大量开源项目。GCC 是 GNU 中的编译器项目，也可以简单认为是**编译器**，它可以编译很多种编程语言（括C、C++、Objective-C、Fortran、Java等等）。

我们的程序**只有一个**源文件时，直接就可以用gcc命令编译它。

可是，如果我们的程序包含很**多个**源文件时，该咋整？用gcc命令逐个去编译时，就发现很容易混乱而且工作量大，所以出现了下面make工具。

**2. make**

make工具可以看成是一个智能的**批处理**工具，它本身并没有编译和链接的功能，而是用类似于批处理的方式，通过调用**makefile文件**中用户指定的命令来进行编译和链接的，最终还是要调用编译器。

**3. makefile**

这个是啥东西？简单来说 makefile 就像一首歌的乐谱，make工具就像指挥家，指挥家根据乐谱指挥整个乐团怎么样演奏，make工具就根据makefile中的命令进行编译和链接的。makefile命令中就包含了调用gcc（也可以是别的编译器）去编译某个源文件的命令。

makefile在一些简单的工程完全可以人工拿下，但是当工程非常大的时候，手写makefile也是非常麻烦的，如果换了个平台makefile又要重新修改，这时候就出现了下面的Cmake这个工具。

**4. cmake**

cmake就可以更加简单的生成makefile文件给上面那个make用。当然cmake还更牛，可以**跨平台**生成对应平台能用的makefile，我们就不用再自己去修改了。

可是cmake根据什么生成makefile呢？它又要根据一个叫CMakeLists.txt文件（学名：组态档）去生成makefile。

**5. CMakeList.txt**

到最后CMakeLists.txt文件谁写啊？—— 是你自己手写的。

## 更术语的解释

### 什么是构建系统？为什么需要构建系统？

构建系统(build system)是用来从源代码生成用户可以使用的目标(targets)的自动化工具。目标可以包括库、可执行文件、或者生成的脚本等等。

**构建系统的需求是随着软件规模的增大而提出的**。如果只是做软件编程训练，通常代码量比较小，编写的源代码只有几个文件。比如你编写了一段代码放入helloworld.c文件中，要编译这段代码，只需要执行以下命令：gcc helloworld.c。

当软件规模逐渐增加，这时可能有几十个源代码文件，而且有了模块划分，有的要编译成静态库，有的要编译成动态库，最后链接成可执行代码，这时命令行方式就捉襟见肘，需要一个构建系统。常见的构建系统有GNU Make（就是上文的make）。需要注意的是，构建系统并不是取代gcc这样的工具链，而是定义编译规则，最终还是会调用工具链编译代码。

当软件规模进一步扩大，特别是有多平台支持需求的时候，编写 GNU Makefile 将是一件繁琐和乏味的事情，而且极容易出错。这时就出现了生成Makefile的工具，比如CMake、AutoMake等等，这种构建系统称作元构建系统（Meta-Build System）。

### 常见的构建系统（Build System）

迄今为止C++至少有89种构建系统，这里主要介绍一些日常工作中会接触到的构建系统工具。

**1. Make（GNU Make，BSD Make）**

Make可以追溯到1976年，属于最早的构建系统，在类Unix系统上比较常用。在类Unix系统上，大多数C++的项目代码是通过编写或生成Makefile进行编译构建的。Makefile早期设计是给研发者编写的构建脚本，功能强大有很多高级功能。

问题：
1. 直接编写Makefile，复杂且难以阅读，维护困难；项目规模比较大的C++项目尤甚。
2. 构建速度相对较慢。笔者常常听人提到“CMake编译速度慢”，CMake表示“这锅我不背”，CMake默认的Build Backend是Make。

**2. Ninja**

Google的一名程序员推出的注重速度的构建工具，是一个专注于速度的小型构建系统。最初是为了对Chromium、Swift等进行快速编译构建，用来替代GNU Make。设计哲学是通过包含描述依赖关系图的方式提供快速的构建。一般作为元构建系统工具的Build Backend。

**3. Bazel**

Bazel是Google内部构建系统Blaze的子集。很久以前，Google使用大的，生成的Makefile来构建软件。这导致构建的速度慢而且不可靠，开始干扰开发人员的生产率和公司的敏捷度。因此，Google工程师创造了Bazel。Bazel优点很明显，缺点也很致命，并不适合大多数公司。考虑到部分同事是bazel的拥趸，笔者这里并不多讲，具体参阅下面的链接《寻找Google的Blaze》和《为什么google bazel构建工具流行不起来》。

### 元构建系统（Meta-Build System）

 Meta-Build System: a build system that generates other build systems。一个生成其他构建系统的构建系统。 如：cmake->make, cmake->ninja, gn->ninja
 
**1. CMake**

CMake是一个元构建系统工具，支持多种语言，多种Build Backend（Make, Ninja, Visual Studio, Xcode等）。在实践中可以通过`-G` option 指定Build Backend。

**2. GYP** (Generate Your Projects)

早期Chromium开源项目采用的是GYP构建系统，这也是一种元构建系统。Google工程师根据GYP规则编写构建工程文件（通常以gyp, gypi为后缀），GYP工具根据gyp文件生成GNU Makefile。接着Chromium项目又整出了Ninja构建系统，但这个Ninja并不是用来取代GYP的，而是**取代GNU Make**的，据谷歌官方的说法是速度有了好几倍的提升。

**3. GN** (Generate Ninja)

GN文件相当于GYP文件的下一代，和GYP差别不大，但是总体上比原来的GYP文件更清晰。

## CMake 实践

### 参考文档

1. 开始使用CMake项目：[`CMake Tutorial`](https://cmake.org/cmake/help/latest/guide/tutorial/index.html#guide:CMake%20Tutorial "CMake Tutorial")

2. 学习如何构建从互联网下载的源代码包：[`User Interaction Guide`](https://cmake.org/cmake/help/latest/guide/user-interaction/index.html#guide:User%20Interaction%20Guide "User Interaction Guide")

3. 学习构建第三方库：[`Using Dependencies Guide`](https://cmake.org/cmake/help/latest/guide/using-dependencies/index.html#guide:Using%20Dependencies%20Guide "Using Dependencies Guide")

### 语法

- `cmake_minimum_required()`

每个项目最顶层的 CMakeLists.txt 文件，都必须从使用该命令来规定最低 CMake 版本开始。这保证了后续的CMake函数都使用兼容的版本运行。

- `project()`

在 cmake_minimum_required() 之后需要立即调用project()命令来设置项目名称。每个项目都需要此调用。该命令还可以用于指定其他项目级别信息，例如语言或版本号。


### 构建

```shell
$ cd MyProject
$ mkdir build
$ cd build
$ cmake --build .
$ cmake --build . --target install
```

- `cmake --build .` 是 CMake 的构建命令，`.` 表示当前目录。这会构建当前目录下的所有目标。

- `cmake --build . --target install` 是 CMake 的一个命令，意思是构建当前目录下制定的目标 `install` 。
	`--target` 是 CMake 的一个选项，用于指定要构建的目标。这里，`install` 表示要构建的目标是 `install`。
