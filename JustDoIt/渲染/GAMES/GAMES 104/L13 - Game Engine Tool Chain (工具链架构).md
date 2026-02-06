## 1. GUI 系统构建 (GUI System)

游戏引擎工具链的 UI 开发主要涉及两种核心模式以及设计模式的应用。

### 1.1 两大类 GUI 实现方式

- **Immediate Mode (立即模式)**
    
    - **代表技术**：IMGUI (Unity 旧版 GUI, Dear ImGui)。
    
    - **特点**：即时生效，每一帧都在重绘。
    
     - **原理 (Deep Dive)**：
        
        - UI 状态通常存储在**栈 (Stack)** 上而非堆 (Heap) 上。
        
        - **Code is the View**：不需要创建复杂的 UI 对象树，代码执行到哪，UI 就画到哪。
	
    - **优缺点**：
        
        - 缺点：压力集中在逻辑层，每一帧都要重新计算布局，难以做复杂动画。
        
        - 优点：**开发速度极快**。非常适合制作 Debug 面板或开发者工具，因为程序员不需要去维护复杂的 UI 状态同步。
    
- **Retained Mode (保留模式)**
    
    - **代表技术**：WPF, Unreal UMG/Slate, Qt。
        
    - **特点**：UI 工具与引擎逻辑拆分。UI 系统保留了控件的状态信息，只有在状态改变时才重绘。
        
    - **优点**：拓展性强，适合复杂的编辑器界面开发。

>[!Question]
>Q：IMGUI 的适用场景？

A：虽然 IMGUI 压力在逻辑层，但它在In-Game Debug工具或者极简单的参数调节面板中依然非常流行，因为写起来最快（几行代码就能出一个按钮）。

---
### 1.2 工具 UI 系统构建原则

**原则**：不要反复造轮子 (Don't reinvent the wheel)。利用现有的成熟框架。

**设计模式 (Design Patterns)**：非常重要，用于解耦逻辑与显示。

#### 1.2.1 MVC (Model - View - Controller)

- **流程**：单行线 (One-way flow)。
    
- **机制**：
    - User `uses` Controller
        
    - Controller `manipulates` Model
        
    - Model `updates` View
        
    - View `sees` User
    
- **问题**：View 和 Model 依然存在耦合（View 需要观察 Model 的更新）。

#### 1.2.2 MVP (Model - View - Presenter)

- **核心**：更彻底地解耦 View 和 Model。
    
- **机制**：
    - **View 完全不关心 Model 数据**。
        
    - User events 传给 Presenter。
        
    - Presenter 修改 Model。
        
    - Model 更新后通知 Presenter。
        
    - Presenter 再 `updates` View。
    
- **特点**：Presenter 作为中间人，双向沟通。

#### 1.2.3 MVVM (Model - View - ViewModel)

- **核心**：**Binding (数据绑定)**。
    
- **机制**：
    - View `<-->` ViewModel `<-->` Model
        
    - View 和 ViewModel 之间通过自动化的 Binding 机制同步。
    
- **优点**：灵活性强，控件复用性极强（WPF 和现代 Web 前端常用模式）。


> [!Question]
> Q：为什么游戏引擎编辑器偏爱 MVVM/MVP？ 

A：因为编辑器的数据（Assets）非常复杂且经常变动。如果用 MVC，View 代码里会充斥着大量“如果 A 变了，B 就要变”的胶水代码。MVVM 的“数据绑定”能自动处理这些同步，极大减少 Bug。

---
## 2. 数据的存储与加载 (Data Storage & Loading)

### 2.1 数据的序列化与反序列化 (Serialization / Deserialization)

数据的本质是存储和传输。

- **Text File (文本文件)**
    
    - **格式**：.txt, .obj, .xml, .html, .json。
        
    - **特点**：
        - 结构化描述语言 (Structured description language)。
            
        - 文本其实是一种**容器**，通过树状标签系统 (Tree Structure) 把信息存住。
            
        - 更轻量的版本：**JSON** (目前最主流)。
    
- **Binary Files (二进制文件)**
    
    - **案例**：FBX 格式同时提供 Text (ASCII) 和 Binary 版本。
        
    - **能力**：读写速度快。
        
    - **优点**：**Binary 资产体积小很多**（不需要存大量字符标记，直接存内存布局）。

### 2.2 数据的引用 (Data Referencing)

如何管理资产之间的关系：

- **Asset Reference**：资产引用（硬盘上的文件路径或 GUID）。
	
	引擎内部不应使用文件路径引用资源（路径会变），而应为每个资源生成一个唯一的 GUID (Global Unique Identifier)。
    
- **Object Instance in Scene**：场景中的实例。
    
- **Object Instance Variance**：实例变种（Prefab 变体）。

**构建变种 (Variance) 的两种方式：**

1. **Build Variance by Copying (复制)**
    
    - **问题**：如果只改动了很小的局部也要全部复制，或者希望牵一发而动全身（修改母体，子体跟着变），单纯的复制会**打破关联**！
    
2. **Build Variance by Data Inheritance (数据继承)**
    
    - **核心**：只存储与原型的“差异数据”（Delta）。（例如：Unity 的 Prefab 机制）。
    

### 2.3 数据的加载

- **大小端问题 (Big Endian / Little Endian)**：不同 CPU 架构（PC vs 主机 vs 手机）读取二进制数据的顺序不同，跨平台开发必须处理。

---
## 3. 资产的管理

### 3.1 资产处理管线 (The Asset Pipeline) 

在数据存储之前，资源需要经历从“原始文件”到“引擎资产”的转化过程。

1. **DCC Tool (源数据)**
    
    - 工具：Maya, Blender, Photoshop。
        
    - 产出：`.mb`, `.psd`, `.max` 等原始格式。
    
2. **Asset Import (资产导入)**
    
    - **Asset Processor**：一个后台进程，监控文件变化。一旦 DCC 导出新文件，自动触发导入器。
        
    - **Logical Asset (逻辑资产)**：导入后生成的中间层，包含 GUID 和引用关系，但还不是最终可用的数据。
    
3. **Asset Processing (资产处理)**
    
    - 根据平台（PC/Mobile/Console）进行压缩、格式转换（如贴图转 ASTC/DXT）。
    
4. **Binary / Cook (打包)**
    
    - **Physical Asset (物理资产)**：最终生成的、针对特定硬件优化的二进制文件。这是游戏运行时真正读取的数据。

### 3.2 资产的版本兼容 (Asset Compatibility)

随着引擎迭代，数据格式会变。

**方向**：Backward compatibility (向后兼容，旧数据能在新引擎跑) / Forward compatibility (向前兼容)。

**解决方案：**

1. **Solve Compatibility by Version Hardcode (版本硬编码)**
    
    - 在代码里写 `if (version < 3) { ... } else { ... }`。
    
2. **Solve Compatibility by Field ID (利用字段 ID)**
    
    - **代表协议**：**Protocol Buffers (PB)**。
        
    - **机制**：定义每个 field (属性) 时都生成一个 unique ID。
        
    - **优点**：比较 Unique ID 就可以获知数据是否有更新，或者是否存在该字段，容错率高。

### 3.3 崩溃恢复 (Crash Recovery)

编辑器崩溃时，如何不丢失用户刚才的操作？

- **核心模式**：**Command Pattern (命令模式)**。
    
- **实现步骤**：
    
    1. 首先要有 **UID** (唯一标识符)。
        
    2. 利用 **Serialization/Deserialization** 将每一个操作指令序列化存到磁盘。
        
    3. 崩溃重启时，读取日志进行 **Restore** (重放操作)。

除了崩溃恢复，**Command Pattern** 模式也是实现 **Undo / Redo (撤销/重做)** 功能的标准做法。把每一步操作都封装成一个 Command 对象，压入栈中。Undo 就是把栈顶的命令 Pop 出来并执行它的 `Undo()` 方法。

---
## 4. Schema

Assets 的生成过程完全不同（模型、贴图、关卡），工具该怎么定义呢？

核心思路是寻找共性。定义 **Schema (模式/架构)**，即统一的描述与定义。告诉各种各样的工具如何去理解某个数据，每个数据的**语义**是什么。

此处就有一个经典的架构选择题：**“数据结构（Schema）应该定义在哪里？”**

是应该写在一个**独立的文本文件**里，还是直接写在**程序的源代码**里？

### 4.1 独立的 Schema 定义文件

这种方式就像是 **“填表”**。先设计好一张空白表格（XML, JSON, XSD 文件），规定好这里填名字、那里填数字，然后程序去读取这张表。

- **典型代表：** XML, JSON, Protobuf。
    
- **优点：**
	
    - **易于理解 (Comprehension easily)：** 格式通常很清晰，非程序员（策划、美术）也能看懂。
        
    - **低耦合 (Low coupling)：** 数据定义和程序逻辑是分开的。修改数据结构不需要重新编译代码。
    
- **缺点：**
	
    - **版本不匹配风险 (Ease to mismatch...)：** 这是最大的痛点。如果代码升级了（比如把“速度”从整数改成了浮点数），但独立的 Schema 文件忘了更新，程序读的时候就会报错或崩溃。
	    
    - **难以定义功能 (Difficult to define function...)：** 文本文件只能描述“有什么数据”，很难描述“有什么逻辑”或“怎么动”。
	    
    - **需要实现解析器 (Need to implement complete syntax)：** 你必须专门写一段代码来“翻译”这个文件，工作量大。

### 4.2 在代码中定义

- **代表**：**Unreal Engine**。
    
- **方法**：用高级语言定义一个类（比如 C++），再用宏 (Macros) 去描述这是一个 Metadata (元数据)。
	
- **实现**：通过 Header Tool 分析宏，自动生成反射代码。
	
- **优点：**
    
    - **容易实现反射 (Ease to accomplish Function reflection)：** “反射”是指程序在运行时能知道自己有哪些变量。在代码里定义，程序天生就知道自己长什么样，很容易自动生成编辑界面。
	    
    - **天然支持继承 (Natural support for inheritance)：** 代码语言（如 C++）天生就支持“猫继承自动物”这种关系，不需要额外造轮子。
    
- **缺点 ：**
    
    - **难以理解 (Difficult to understand)：** 对于非程序人员，直接看 C++ 代码里的头文件是很痛苦的，且充满了干扰信息。
        
    - **高耦合 (High coupling)：** 数据结构和逻辑绑死了。想改一个字段的名字，可能牵一发而动全身，需要重新编译整个引擎。

### 4.3 Schema 生成代码

还有一种常见的混合做法（工业界最常用）：

- 你写一个独立的 Schema 文件（比如 Google 的 Protocol Buffers，或者 Qt 的 `.ui` 文件）。
    
- 然后用一个工具，**把这个文件“编译”成 C++ 代码**。
    
- 生成的 C++ 代码里**包含了**反射逻辑。

在这种情况下，**原本独立的 Schema 变成了反射的“源头”**。

**反射 (Reflection)** 是Schema能够工作的底层魔法。反射允许程序在运行时检查自己的结构（有哪些类、哪些变量）。

在 **Unreal** 中，你写的 `UPROPERTY()` 就是在利用反射系统告诉编辑器：“嘿，这个变量需要在编辑器面板里显示出来供策划修改”。如果没有反射，每加一个变量，程序员都要手动去写 UI 控件代码，工作量会爆炸。

下一章会对反射有更详细的讲述。

