
# 1 概述


在移动设备上实现高性能、低功耗的图形渲染，是现代游戏和应用开发面临的核心挑战。Vulkan 作为新一代图形 API，为 Android 开发者提供了对 GPU 硬件前所未有的底层控制权，从而解锁了巨大的性能潜力。然而，这种控制权也带来了更高的复杂性和调优难度。

本文旨在为 Vulkan 开发者提供一套系统化的性能优化方法论。我们将首先解析移动端 GPU（如 分块渲染器）的工作原理，奠定优化基础。随后，我们将详细指导如何利用专业的 性能分析工具 和 关键 GPU 指标 科学地定位瓶颈。

文章的核心将围绕三大性能瓶颈展开深入讨论：CPU 瓶颈、GPU 计算瓶颈和内存带宽瓶颈。我们将为每一种限制提供具体的优化思路和最佳实践。此外，本文还将涵盖对 内存资源的高效管理 以及在移动环境中至关重要的 功耗优化策略。

通过掌握这些知识，我们将能够精准识别性能瓶颈的主导因素，并运用 Vulkan 的强大能力，在 Android 平台上构建出性能卓越、延迟更低、电池续航更持久的下一代图形应用。

---
# 2 GPU原理


想要掌握甚至精通Vulkan程序的性能优化，了解GPU的工作原理是非常重要的一个基础前提。了解了GPU是如何运行的之后，才能知道在GPU在哪些流程会产生性能瓶颈，才能看明白性能分析工具的那一大堆性能参数，才能在众多的性能数据中洞察到正确的优化方向。

## 2.1 GPU架构

### 2.1.1 宏观架构

一个通用的GPU硬件的宏观架构大致如下：

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051511877.png)

**1. 分层结构:**

一个 GPU 由多个 Core (Core 1, Core 2, Core 3...) 组成，每个 Core 内部又有多个基本的计算单元 (Shader Core)。

**2. 存储层次结构:**

- **寄存器文件 (Register File):** 位于最快的一级，专供 Shader Core 快速存取局部变量。
- **共享内存 (Shared Memory):** 位于 Core 内部，用于线程组 (Work Group) 内的高速通信和数据共享。
- **L1 Cache:** 位于 Core 内部，供 Core 内部的所有 Shader Core 使用，提供低延迟的数据访问。
- **L2 Cache:** 位于 GPU 芯片级别，被所有 Core 共享，作为 L1 Cache 的下一级，靠近内存控制器。
- **显存 (VRAM/DRAM):** 最外层和最大的存储空间，由内存控制器管理。
- 访问速度上：寄存器 > 共享内存 > L1 Cache > L2 Cache > 显存。

### 2.1.2 微观架构

如果再深入到一个ShaderCore中，我们以ARM Valhall架构GPU为例（比如Mali-G77/G78/G710等），其内部结构大概如下：

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051511652.png)

整个结构图可以分为 **输入调度、计算单元、数据单元** 和 **输出** 四个部分。

#### 2.1.2.1 输入调度 (Warp Creation)

这里会将 CPU 提交的命令转换成可由 Shader Core 执行的并行线程束（Warp）。

**Non-fragment warp creator (非片段线程束创建器):**

- 负责处理所有 **非像素** 相关的计算任务，包括：顶点着色 (Vertex Shading)、几何体处理 (Geometry Processing)、计算着色 (Compute Shading) 等。

**Fragment warp creator (片段线程束创建器):**

- 负责处理所有 **像素** 相关的计算任务，主要是片段着色 (Fragment Shading) 的调度。

#### 2.1.2.2 计算单元 (Processing Units - PUs)

PU 是 Shader Core 的核心 ALU/FPU 部分，用于执行着色器指令。每个 PU 内部包含三个关键的数学单元：

- **FMA (Fused Multiply-Add):** **融合乘加单元**。这是 GPU 中最常见的计算单元，可以将乘法和加法操作合并为一个指令，常用于浮点运算（如光照计算、矩阵乘法），效率极高。
- **CVT (Convert Unit):** **转换单元**。负责数据类型转换（如浮点数转整数），以及一些简单的数学操作。
- **SFU (Special Function Unit):** **特殊函数单元**。用于执行复杂的、非线性的数学函数，如三角函数 (sin, cos)、指数、对数、倒数平方根等。

#### 2.1.2.3 数据与访存单元 (Data and Memory Units)

这些单元协助计算单元高效地获取和存储数据。

- **Message fabric (消息总线):** 负责核心内部的通信，在 PU、内存单元和外部缓存之间传递数据。
- **Load/store unit (加载/存储单元):** 负责将数据从缓存或内存中 **加载 (Load)** 到寄存器，以及将结果从寄存器 **存储 (Store)** 回缓存或内存。它管理着对共享内存和 L1 缓存的访问。
- **Varying unit (可变值单元):** 负责处理从顶点着色器输出的 **可变值 (Varyings)**。它执行这些数据的插值，并将插值后的值提供给片段着色器使用。
- **Texture unit (纹理单元):** 专门处理纹理采样操作。它执行**纹理地址计算、纹理过滤（如双线性、三线性）和格式解压**，将最终的颜色值提供给片段着色器。它也被称为 **TMU (Texture Mapping Unit)**。

#### 2.1.2.4 输出 (Output)

**Fragment blend (片段混合单元):** 这是渲染管线的最后一步。它接收片段着色器计算出的最终颜色，并执行**深度/模板测试**、以及与帧缓冲区中现有像素的**颜色混合 (Blending)** 操作。


## 2.2 GPU是如何运行的？

我们写的着色器，经过编译和链接，最终会运行在GPU上。那你有没有想过，一个着色器程序，在GPU上具体是如何运行的呢？

### 2.2.1 GPU并行执行模型

当 CPU 通过 Vulkan 提交一个 Draw Call 或 Dispatch Compute 命令给 GPU 后，会经历以下阶段。

**1. 任务划分与调度**

- **顶点着色器：** 被划分为处理每个顶点的任务。
- **片段着色器：** 被划分为处理屏幕上每个潜在像素的任务。
- **计算着色器：** 被划分为用户定义的通用计算任务。

**2. 线程束创建 (Warp Creation)**

GPU 的 **Job Manager** 和 **Warp Creator** 接收这些任务，并将它们组织成 **线程束 (Warp)**。Warp是 GPU 的 **最小调度单位**。

**3. SIMT 执行**

线程束被分配给一个或多个 **Shader Core** 执行。在 Shader Core 内部，所有线程束中的线程都会**同时执行相同的指令**（即 SIMT）。

这种执行模型有以下特点：

- **高并行性：** GPU 通过同时运行数百甚至数千个线程束来隐藏内存延迟。当一个线程束在等待数据时，Shader Core 可以立即切换到另一个准备就绪的线程束。
- **分化处理：** 如果线程束内的线程遇到 `if-else` 或 `switch` 语句需要执行不同的代码路径（即 **线程束分化**），GPU 会串行地执行这两个路径，这会导致性能损失。

**4. 数据访问**

单个线程：从**寄存器**（局部数据）、**共享内存**（块内共享数据）或 **L1/L2 缓存/显存**（全局数据）中获取数据，并在 **ALU/FPU 单元**上执行着色器指令。

### 2.2.2 区分线程、Warp和工作组

这三个概念构成了 GPU 并行执行的三个层次：

**1. 线程 (Thread) - 最小单元**

- **定义：** **执行着色器代码的最小逻辑单元。** 在图形管线中，一个线程通常对应一个顶点、一个片段或一个计算单元。
- **特点：** 每个线程都有自己独立的程序计数器和一套寄存器，用于存储自己的局部变量和临时结果。
- **执行方式：** 线程不是独立调度的，它必须包含在线程束内。
- **数量：** 由输入数据决定。例如，如果有 N 个顶点，就有 N 个顶点着色线程。


**2. 线程束 (Warp) - 最小调度单位**

- **定义：** **GPU 硬件并行执行的基本单位。** 一组线程（线程数量取决于厂商和架构）被打包在一起，**同时**执行相同的指令。
- **特点：** 它是 **SIMT (单指令多线程)** 模型的物理实现。GPU 的调度器最小只能调度一个线程束。
- **关系：** 多个线程组成一个线程束。线程束内的所有线程共享同一个指令计数器。
- **数量：** 由 GPU 硬件固定。


**3. 工作组 (Work Group) - 逻辑组织单位**

- **定义：** **由程序员在计算着色器 (Compute Shader) 中定义的逻辑分组单元。** 一组线程共同协作，完成一个较大的任务。
- **特点：** 工作组是**共享内存**的边界。同一工作组内的所有线程可以通过快速的 **共享内存 (Shared Memory)** 互相通信和数据交换，并使用 **`barrier` (同步屏障)** 进行同步。
- **执行方式：** 一个工作组通常由一个或多个Warp组成，并被分配给一个 **Core** 执行（因为分配给一个 Core，所以能共享内存）。
- **数量：** 由程序员在调度计算任务时定义（例如：`vkCmdDispatch(X, Y, Z)`）。

|概念|逻辑作用|硬件关系|谁来定义|关键特点|
|:--|:--|:--|:--|:--|
|**线程 (Thread)**|最小逻辑执行单元|包含在线程束内|输入数据量|有自己的寄存器和局部变量。|
|**线程束 (Warp)**|最小调度单位|包含多个线程|GPU 硬件|SIMT 并行执行；若分化则串行。|
|**工作组 (Work Group)**|最小协作和同步单位|包含多个线程束|程序员|线程可以访问共享内存；可以进行同步。|

### 2.2.3 一个GPU Core可以并发执行多少Warp？

了解了GPU架构和GPU执行原理之后，我们知道了GPU是一个高度并发的执行模型，那一个GPU Core能够并发执行的数量，就决定了这个着色器程序的执行效率，也就是性能高不高。

一个 GPU **核心 (Core)** 同时可以执行的Warp数量取决于两个主要因素：

1. **硬件限制：** 核心的物理限制和寄存器文件大小。
2. **资源需求：** 着色器代码对寄存器和共享内存的需求量。

#### 2.2.3.1 硬件限制

每个 GPU 核心都有一个固定的最大容量来存储活跃线程束的状态，这主要由以下因素决定：

- **最大Warp数量：** 每个 Core 都有一个硬性的上限，规定了最多可以容纳多少个活跃的线程束。
	
	- 例如，NVIDIA 的一个 Core 可能最多可以驻留 32 个活跃线程块（对应上百个 Warp）。
	
- **寄存器文件大小：** 寄存器是 GPU 最宝贵的资源。一个Core的总寄存器数量是固定的。

 一个 SM/CU（GPU 的一级 Core）有多少个 Shader Core没有统一答案，但有明确的行业规律（按主流架构举例）：
 
 1. NVIDIA（CUDA Core = Shader Core）
	- 老架构（Kepler/Maxwell）：每个 SM 含 **32 个** CUDA Core；
	- 主流架构（Pascal/Turing/Ampere）：每个 SM 含 **64 个** CUDA Core；
	- 新架构（Ada Lovelace/Blackwell）：每个 SM 含 **128 个** CUDA Core（拆分了 FP32/INT32 单元，本质仍是 Shader Core 数量翻倍）。
	
2. AMD（Stream Processor = Shader Core）
	- RDNA1/RDNA2 架构：每个 CU 含 **64 个** Stream Processor；
	- RDNA3 架构：每个 CU 含 **64 个** Stream Processor（但每个 CU 的算力提升，核心数未变）；
	- 老 GCN 架构：每个 CU 含 **64 个** Stream Processor（长期保持稳定）。

#### 2.2.3.2 资源需求

实际能够并发的Warp数量通常受到**资源约束**的影响，即受到你编写的着色器代码的限制：

$$ \text{并发 Warp 数量} = \min\left( \frac{\text{核心总寄存器数量}}{\text{每个 Warp 的寄存器需求}}, \frac{\text{核心总共享内存}}{\text{每个工作组的共享内存需求}}, \text{硬件上限} \right) $$

- **寄存器需求：** 如果你的着色器代码使用了大量的**局部变量**（即对寄存器需求高），那么每个线程束会消耗更多的寄存器资源。核心的总寄存器数量有限，高需求会导致可分配给的线程束数量减少。
    
    - _例如：_ 核心有 65536 个寄存器。如果你的代码每个 Warp 需要 4096 个寄存器，那么最多只能驻留 65536 / 4096 = 16 个 Warp。
    
- **共享内存需求：** 如果你使用 **计算着色器** 并且需要大量的 **共享内存 (Shared Memory)**，这也会限制并发的工作组（以及它们包含的线程束）数量。

## 2.3 移动GPU渲染架构

### 2.3.1 IMR

传统桌面 GPU 使用的 **即时渲染器 (IMR：Immediate Mode Renderer)** 架构（例如 NVIDIA GeForce, AMD Radeon）。

IMR的工作方式是逐图元 (Per-Primitive) 渲染。GPU 接收到 Draw Call 后，立即处理所有图元（三角形），并将其渲染结果（像素）直接写入到外部的显存 (DRAM/VRAM)中的帧缓冲区。

它的渲染流程是： 几何体处理 -> 光栅化 -> 像素着色 -> **写回全局内存**。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051530597.png)

在移动端上，IMR存在致命的缺陷： 

1. **高内存带宽需求：** 每次像素写入都必须访问速度较慢、功耗较高的外部显存。
2. **过度绘制时功耗高：** 如果一个像素被绘制了N次（即N-1次 Overdraw），就会有N次写入操作到全局内存，浪费大量功耗和带宽。

### 2.3.2 TBR

**TBR（Tile-Based Rendering）** 是移动 GPU 引入的用于解决IMR缺陷的方案，它改变了渲染目标和渲染顺序。

TBR的核心思想是将屏幕（或渲染目标）分割成许多小块，称为 **“瓦块” (Tiles)**。GPU 每次只处理一个瓦块的所有渲染工作。

TBR的渲染流程如下：

1. **分块 (Tiling):** 将屏幕划分为固定大小的 Tile（如 16x16 或 32x32 像素）。
    
2. **几何处理:** 对所有顶点执行位置着色(Position Shading)和属性着色(Varying Shading)，计算每个的Tile包含的图元列表，记录在显存中。
	
	- 属性着色（Varying Shading）是**GPU 渲染管线中连接顶点着色器与片段着色器的核心技术**，本质是通过**光栅化阶段的插值计算**，将顶点着色器输出的 “顶点属性”（如颜色、纹理坐标、法线、光照强度等）平滑传递到每个像素（片段）。
    
3. **Tile渲染:** 将当前瓦块的渲染所需数据从全局内存加载到 GPU 芯片上的高速**片上存储 (On-Chip Memory / Tile Memory)**，对该瓦块执行完整的像素着色和混合，将最终的瓦块渲染结果写回全局内存。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051532669.png)

TBR 仍需要多次访问全局内存，但至少将大量操作集中在 Tile Memory 中。

### 2.3.4 TBDR

**TBDR（Tile-Based Deferred Rendering）** 是目前主流移动 GPU（如 Arm Mali, Imagination PowerVR）普遍采用的优化架构，它在 TBR 的基础上增加了 **“延迟” (Deferred)** 概念。

TBDR不仅将屏幕分成瓦块，而且**延迟**了像素着色和深度/模板测试，直到几何体处理完成后才进行。它的渲染流程是一个两阶段管线：

**阶段 1: 几何体处理 (Geometry Processing) - Non-Fragment Job**

1. **分块 (Tiling):** 将屏幕划分为瓦块。
    
2. **顶点着色:** 主要执行位置着色 (Position Shading) 以确定图元所属的 Tile，属性着色 (Varying Shading) 可能会被优化延迟执行。
    
3. **图元分配:** 找出每个 Tile 内包含哪些三角形（图元ID），并将这个Polygon List存储到片上或全局内存。

在这一阶段，没有像素被绘制，也没有纹理被采样。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051537830.png)

**阶段 2: 像素处理 (Pixel Processing) - Fragment Job**

1. **Tile 加载:** 逐个加载 Tile 到 Tile Memory。
    
2. **隐藏表面消除 (Hidden Surface Removal - HSR):** GPU 使用 Tile Memory 中高效的深度/模板测试来确定哪些片段是可见的。在执行片段着色器之前，不可见的像素就被剔除掉了。
    
3. **片段着色 (Fragment Shading):** 仅对 **可见的** 像素执行复杂的片段着色器和纹理采样。
    
4. **Tile 存储:** 最终的颜色结果从 Tile Memory 写回全局帧缓冲区。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051540971.png)

图里的 **Transaction Elimination（事务消除）** 是**ARM Mali GPU（图中是 ARM 架构）特有的带宽优化技术**，核心作用是**减少 Frame Buffer（帧缓冲）的冗余内存写入，从而节省带宽与功耗**。结合图中的流程，它的原理和作用如下：

1. 技术定位（在图中的位置）

在这个片段处理流程中，它位于 **Tile Write（瓦片写入）之后、Frame Buffer（帧缓冲）之前** —— 负责在 “将渲染后的瓦片数据写入全局帧缓冲” 这一步，过滤掉 “没有变化的冗余数据”。

2. 核心原理

它以**16×16 像素的 Tile（瓦片）为单位**工作，核心逻辑是 “**对比新旧帧的 Tile，消除重复写入**”：

- 渲染完成后，Tile RAM 中会暂存当前帧的瓦片数据；
- Transaction Elimination 会计算当前 Tile 的**CRC 签名**（可以理解为 Tile 内容的 “数字指纹”）；
- 将这个签名与**上一帧对应位置 Tile 的签名**对比：
    - 如果签名一致（Tile 内容没变化）：跳过这个 Tile 的写入操作（消除冗余的内存事务）；
    - 如果签名不一致（Tile 内容有变化）：才将这个 Tile 写入 Frame Buffer。

### 2.3.5 TBDR 相较于 IMR 的核心优势

|特征|IMR (桌面/传统)|TBDR (移动端/Mali, Adreno)|
|:--|:--|:--|
|**内存访问**|频繁写入外部 DRAM（高功耗）。|大部分读写发生在高速 **Tile Memory** (片上存储器) 中。|
|**过度绘制 (Overdraw)**|浪费带宽：每个 Overdraw 都要写入 DRAM。|**消除浪费：** **只对最终可见的像素**执行片段着色和写入，极大地节省了功耗和带宽。|
|**带宽瓶颈**|极易受内存带宽限制。|有效缓解，将瓶颈转移到 **Tile Memory**。|

**总结：** TBDR 架构通过利用高速片上存储器和 **HSR（隐藏表面消除）** 技术，将移动 GPU 最昂贵的操作（内存访问和片段着色）推迟到必要时才执行，是移动设备功耗优化的基石。


---
# 3 性能分析工具


在对GPU原理有了基础的了解之后，我们再学习性能分析及优化就有了理论基础。但是性能优化是一个系统工程，要快速分析出当前程序存在的性能问题，一个功能强大的性能分析工具会助我们事半功倍。正所谓工欲善其事，必先利其器。

目前Android Vulkan程序性能分析工具，主流有以下4个：

- Arm Performance Studio：Arm厂商工具
- Snapdragon Profiler：高通厂商工具
- Android GPU Inspector：Google官方工具
- PerfDog：腾讯性能优化工具


## 3.1 Arm Performance Studio

Arm Performance Studio是Arm为Mali架构GPU设备提供的一个性能分析和优化工具套件，主要包含以下三个工具：

- StreamLine：核心工具，系统级性能分析器
- Frame Advisor：基于Streamline数据的自动化分析和性能建议工具
- Mali Offline Compiler：Shader离线性能分析器

### 3.1.1 Streamline(性能数据采集与分析)

**StreamLine** 是 Arm Performance Studio 的核心工具，是一个强大的**系统级性能分析器**， 其关键功能包括：

- **时间线视图:** 在时间轴上关联显示 CPU 线程活动、GPU 核心利用率、内存带宽使用等数据，帮助识别同步等待 (Stalls)。
- **计数器 (Counters):** 收集数百个硬件和软件计数器，例如 GPU 核心活跃度、缓存命中率、带宽等。

Steamline能够采集非常多的数据，这里我们挑一些常用的数据，来说明其含义和使用场景：

#### 1. Mali GPU Usage/Utilization

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051708974.png)

**GPU Active：** 1个时间单位帧内 GPU 所使用的时钟周期。我们把采样周期调为1s，这样我们就能看到1秒钟内 GPU 所用的周期是 183M 左右。

按照之前章节讲的 GPU 的工作原理，Job Manager 会将任务交给 Non-fragment 和 Fragment。**理想状态下它们可以与 CPU 任务同时进行，而且 Non-fragment 和 Fragment 也可以并行进行**。于是会有如下指标：

- **Non-fragment queue active：** 非片元队列活动周期，队列包含顶点着色器、Tile 分块以及计算着色器。
- **Fragment queue active：** 片元队列活动周期。上图中可以看到一共 62M 用于片元，相对来说还是比较少的。
- **Tiler active：** Tiler 任务所使用的时钟周期。TBR渲染架构中会有Tiler这个阶段，主要处理屏幕分块。
- **Interrupt active：** 中断产生的时钟周期。

这些参数可以在下图中一一对应找到：

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051710154.png)

而上图中的 Mali GPU Utilization 就是各个指标的利用率，比如Tiler Utilization = Tiler active / GPU active。这里要注意下，==因为各个阶段可能是并行的，所以利用率加起来可能大于100%。==

==如果某个阶段的利用率特别高，就要考虑这个阶段是不是存在性能瓶颈==，需要进行优化了。

#### 2. Mali Memory Bandwidth

GPU 读写内存就会产生带宽，比如片元着色器渲染结果写入 FrameBuffer 就会产生写入带宽（Write bandwidth），Shader 中采样纹理就需要从内存中读取，产生读取带宽（Read bandwidth）。带宽如果过高，则会导致性能问题，而且最重要的是比较费电。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051713039.png)

前面介绍的TBR/TBDR渲染架构，就是为了降低带宽，从而在移动GPU上节省功耗。

带宽的单位是giga-bytes/second，也就是GB/s。上图中读写带宽都是3.59GB/s，也就是每秒要读写3.59GB的数据。

现在手机中使用的内存是 Low-Power DDR SDRAM 缩写 LPDDR，通过名字就能看出它是一种低能耗的随机存储器。如果内存频率是2133MHZ，那么它的**带宽上限 = 内存频率 x 位宽 = 2133MHZ x 16 = 34Gb/s**。

上图中可以看出我们的带宽读写加起来大概 7.68GB/s，ARM 给的建议是移动手机上 5GB/S 以内是安全的，所以这个程序带宽是用的比较多的，需要想办法优化带宽的使用。

#### 3. Mali Memory Stall Rate / Read Latency

因为GPU访问显存是比较慢的，所以 GPU 会提供一个 L2 缓存，如果缓存命中率高就会减少带宽的使用。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051719412.png)

Mali Memory Stall Rate：内存停顿率。 高的停顿率表示GPU所请求的数据量超出了内存系统所能提供的数量，所以说优化的策略还是减少带宽。

Mali Memory Read Latency：内存读取延迟周期。 文档中表示超过 256 个 cycles 就表示GPU请求的数据量超出了内存系统所能提供的了，这时就需要优化带宽了。

- 若内存延迟是`N`个 cycle，那么需要至少`N`个就绪 Warp，才能让 GPU 在这`N`个 cycle 内一直执行其他 Warp 的指令，完全隐藏延迟。
- 实际场景中，Mali 每个 Core 的可用就绪 Warp 数量通常远低于 256。
- “延迟超 256cycle” 的本质是“内存系统供不上数据”，背后对应的是**内存系统的带宽 / 缓存能力不足**。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051931330.png)

#### 4. Mali Geometry Usage

再回顾下之前章节讲的TBDR渲染流水线，分为Vertex Pass和Fragment Pass两个阶段。

**Vertex Pass：**

1. Position Shading：这里先只处理Vertex Shader中的顶点数据。
2. Facing Test Culling：剔除掉背面的多边形。
3. Frustum Test Culling：剔除掉视锥体以外的多边形，也就是剔除摄像机看不到的。
4. Sample Test Culling：剔除掉非常小小到连 1 个像素都没有的多边形。
5. Varying shading：如果几何体没有被之前的剔除掉，开始执行Vertex Shader的其他着色计算。
6. Polygon List：生成多边形列表，也就是前面提到的 Tile List 写入显存中，这里会发生一次内存写入。

**Fragment Pass：**

1. GPU 从内存中读取 Tile list，这里会发生一次内存读取。
2. 光栅化：将多线形变成 2D 像素。
3. Early ZS Test：这个非常重要，这时 2D 的几何体是有深度的，GPU 知道如何进行排序，这样就可以将完全挡住的像素删掉，后面的就不会在进行。但是如果你的 Shader 中用了 Alpha Test 的 discard clip 这种操作，此时 GPU 就无法相信光栅化后的排序，这个像素对应的后面的计算就需要执行，那么性能就差了。
4. 创建 Fragment 线程开始执行片元着色器。
5. Late ZS Test：片元着色器都执行完后，这时候就能真正确认每个像素的遮挡关系，继续剔除掉被挡住的像素。但是注意 Fragment 压力是非常大的，所以一定要优先支持被 Early Z Test 剔除掉。
6. Blend：开始混合颜色，也就是上色。
7. Tile RAM & Tile Write：将颜色存入每个 Tile 块中，此时还在 GPU 中，所以这里不会占用带宽。

这时候再看 **Mail Geometry Usage** 指标就很好理解了：

- Visible Primitives：最终显示的几何图元。
- Culled Primitives：被裁掉的几何图元。
- Input Primitives：输入的几何图元。

如果被裁掉的几何图元过多，那么可能就需要考虑使用LOD进行优化。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051947662.png)

**Mali Geometry Culling Rate：** 三个culled primitive rate，是跟Vertex Pass中的三个culling阶段一一对应的。

**Mali Geometry Threads：** GPU 会将开发者提供的顶点着色器分成两部分（Position Shading & Varying Shading），一个是专门处理位置的着色器，另一个是处理其他参数，Mali Geometry Threads就是处理这两部分的线程的数量。上图中因为剔除率不是很高，所以这里两个着色器的执行线程数量也就十分接近。

**Mali Geometry Efficiency:** 这个指标组用于衡量几何体处理单元的工作效率，特别是评估将输入的原始几何体（三角形/图元）转化为屏幕上可见像素的工作量。

Position threads / input primitive，这个指标代表每个输入到几何处理阶段的图元需要消耗多少个着色线程来计算其顶点位置。 Varying shading threads / visible primitive，这个值表示平均每个可见的图元需要消耗多少个着色线程来计算顶点属性（Varying Attributes，如法线、纹理坐标、颜色等）。

这两个值都是越接近1越好，如果大于1，就说明被重复执行了。

#### 5. Mali Varying Usage

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051953302.png)

Mali Varying Usage，这个指标对应的是Vertex Pass中Varying Shading这个阶段，衡量的是 GPU 插值硬件单元 (Varying unit) 的繁忙程度和插值的精度，这直接影响到几何体处理和片段着色之间的带宽和计算。

- Varying unit issue: 衡量 Varying unit 单元因等待数据或资源而停顿的总周期数。如果这个值比较高，通常是由于其依赖的 Load/Store unit 正在忙碌，或者寄存器溢出导致数据加载延迟，是当前性能瓶颈的一个关键信号。
- 16-bit interpolation active (16位插值活跃): 衡量使用 低精度 (FP16) 进行 Varying 插值的活跃周期数。
- 32-bit interpolation active (32位插值活跃): 衡量使用 标准精度 (FP32) 进行 Varying 插值的活跃周期数。

#### 6. Mali Pixels

这几个指标对应的是TBDR中Fragment Pass这个阶段。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512051956469.png)

**Mali Pixels：** 比如现在测试机 GPU 是 850MHZ 共 3 核心，分辨率是 2280X1080，目标帧率是60。总像素量就是「246 万 × 60」≈ 14800 万（1.48 亿）像素 / 秒。上图中我们的采样周期是1秒，Mali Pixels 显示1秒产生了153M的像素。比理论的1.48 亿多一点，因为存在Overdraw。3 个核心，每个核心每秒能跑850MHz（就是 1 个核心 1 秒能完成 850,000,000 次 “最小操作”，这个 “最小操作” 叫**时钟周期**。随之我们就能算出这台手机要想流畅运行60帧的话，每个像素使用的 GPU 时钟周期必须小于 (850MHZ X3)/(2280 X 1080 X 60) = 17，当然这还包括前面处理顶点的时间周期。

**Mali Overdraw:** 1秒产生了153M个像素，理想情况下每个像素只绘制一次。但实时上是不太可能的，比如Alpha blend会在同一个像素上Blend多次，这就是我们说的OverDraw（实际Fragment的数量除以最终产生的像素数），上图中的值是2.98threads，大概每个像素被过度绘制2.98倍。注意，过度绘制 2.98 倍是提交的片元数（4.4 亿）÷ 理想像素数（1.48 亿），而看到的 153M 是最终处理的片元数 —— 两者差了TBDR 架构的片元剔除优化。包含：

- HSR (Hidden Surface Removal)：**HSR 是 TBDR 区别于 TBR 的决定性特性**，仅让**通过深度测试的片元**进入真正的片元着色阶段，被遮挡片元直接丢弃。
	
- Early-Z 与各种 Z 测试优化：提前发现并剔除不可见片元的系列技术。
	
- **Z-Prepass**：专门执行一次仅计算深度的渲染 pass，生成高精度深度图。主渲染 pass 利用此深度图进行更精准的早期剔除，特别适合复杂场景。

| 对比项        | **HSR**                                   | **Early-Z**                              |
| ---------- | ----------------------------------------- | ---------------------------------------- |
| **处理范围**   | 对**整个 Tile / 屏幕区域**的所有图元统一处理，建立全局可见性表     | 对**每个图元独立**处理，只检查当前图元与已绘制内容的遮挡关系         |
| **执行阶段**   | 等待**所有图元光栅化完成**后，再进行统一的可见性判断和着色           | 在**每个图元光栅化后立即**进行深度测试，然后执行着色             |
| **提交顺序**   | **完全无关**，不管绘制顺序如何，都能正确识别最终可见片元            | **高度依赖**，必须按**从近到远**的顺序绘制，否则会产生错误的遮挡判断   |
| **过度绘制消除** | **彻底解决**，实现 "零 Overdraw"，每个像素只执行最终可见片元的着色 | **部分缓解**，无法处理图元间复杂交叉遮挡，仍会有无效着色           |
| **硬件架构**   | 专属于**TBDR 架构**，需要片上内存支持完整可见性缓冲区           | 适用于**所有渲染架构**(IMR/TBR/TBDR)，是现代 GPU 基础特性 |
⚠️ HSR 工作流程包含了 Early Z 阶段——首先对所有图元进行初步光栅化，仅计算深度值，不执行着色，此阶段使用 Early Z 测试。

|对比项|HSR|Z-Prepass|
|---|---|---|
|**实现层级**|**硬件架构层面**（TBDR 特有）|**应用 / 引擎层面**的优化技巧|
|**内存使用**|**片上高速 SRAM**（带宽高、延迟低）|系统内存 / 显存（带宽低、延迟高）|
|**处理范围**|**全局深度排序**（整个 Tile）|基于**已存在的深度缓冲区**做测试|
|**overdraw 消除**|**100% 消除**（理论上）|大幅减少但**无法完全消除**|
|**性能影响**|硬件原生支持，**几乎无额外开销**|需**额外渲染 pass**，增加约 30-50% 负载|

**Mali Pixel Throughput：** 像素吞吐量。计算方式是GPU实际执行的周期数除以像素数，这样就能算出像素的吞吐量，也就是每个像素消耗的平均GPU周期。

**Early ZS Rate：** 我们要尽可能的让像素被提前 Early Z 裁减掉，这样就可以避免执行片元着色，从而提升绘制效率。这里可以看到真正被 Early Z 裁减掉的比例。不透明物体一定要从前往后绘制，这样大部分都可以被 Early Z 裁剪掉。

- **Early ZS test rate：** 参与 Early Z 深度和模板测试的比例。    
- **Early ZS update rate：** Early Z 深度和模板测试期间更新帧缓冲区的光栅化四边形的百分比。
- **Early ZS Kill rate：** 被 Early Z 深度和模板测试剔除掉的光栅化四边形的百分比。
- **FPK kill rate：** Forward Pixel Kill，如果被前面的像素挡住将终止这个线程，后面也不会将不透明的像素写入相同像素中。

**Mali Late ZS Rate：** 在用了Alpha Test的情况下，一个像素可能会在这里被剔除掉。但是到这里实际上片元着色器已经执行完了，所以优先让像素被 Early Z剔除掉才是最优的。

#### 7. Mali Core Warps

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512052003097.png)

**Mali Core Warps** 是指Shader Core在一定时间内执行的线程束 (Warp) 的总数量。

- Non-fragment warps：非片元着色任务执行的线程束数量。
- Fragment warps：片元着色任务执行的线程束数量。

下图中可以很好的区分Non-fragment warps和Fragment warps。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512052004547.png)

**Mail Core Throughout**，着色器核心吞吐量，是指每个线程执行所销毁的瓶颈GPU周期。数值越大，表示完成单个任务所需的计算时间越长，效率越低。

那么 Mali Core Warps 和 Mail Core Throughout 这两个指标有什么区别呢？简单来说：

- Mali Core Warps 衡量的是 工作总量 (Volume)。
- Mali Core Throughput 衡量的是 工作效率 (Efficiency)。

如果 Fragment Warps 多，说明像素处理工作量大，可能是 Overdraw 或 高分辨率。 
如果 Non-fragment Warps 多，说明几何体处理工作量大，可能是 顶点数过多 或 Draw Call 过多。

如果Cycles/fragment thread 高，说明片段着色器过于复杂，需要大量 ALU 周期。 
如果Cycles/non-fragment thread高，说明顶点着色器过于复杂，需要大量 ALU 周期。

上图中 Cycles/non-fragment thread (40.3) 远高于 Cycles/fragment thread (2.6)，说明尽管片段工作总量大，但片段着色器本身非常简单；顶点着色器的每线程计算量（40.3 周期）则相对复杂。

#### 8. Mali Core Utilization / Core Unit Utilization

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512052009060.png)

Mali Core Utilization 和 Mali Core Unit Utilization 这两个指标是Arm Mali GPU 性能分析中最高层次的利用率指标。它们是诊断 GPU 是否处于满载状态 以及 哪个内部单元是瓶颈 的关键。

**1. Mali Core Utilization** 核心整体利用率，包含以下指标：

- Shader Core usage (着色器核心使用率) : 整体核心利用率。衡量所有 Shader Core 执行的利用率。
- Non-fragment utilization (非片段利用率): 几何/计算处理的利用率。衡量处理顶点、几何和计算任务的利用率。
- Fragment utilization (片段利用率): 像素处理的活跃度。衡量处理像素着色任务的利用率。
- Fragment FPK buffer utilization (片段FPK缓冲区利用率) : FPK (Fragment Payload Kicker) 缓冲区的忙碌程度。FPK 是 Mali 内部用于管理和调度片段着色任务的关键缓冲区。
- Execution core utilization (执行核心利用率) : 实际 ALU/FPU 单元的繁忙程度。衡量执行着色器指令的计算单元的利用率。

**2. Mali Core Unit Utilization** 核心单元利用率，这个指标深入到 Shader Core 内部的专业化单元，用于精确定位性能瓶颈是发生在 ALU、纹理单元还是内存访问单元。

- Arithmetic unit utilization (算术单元利用率): ALU/FPU (FMA, CVT, SFU) 的忙碌程度。衡量执行数学运算（如加减乘法、光照计算）的单元的利用率。
- Varying unit utilization (可变值单元利用率): 可变值插值单元的忙碌程度。衡量处理顶点到片段数据插值的单元的利用率。
- Texture unit utilization (纹理单元利用率): TMU (纹理映射单元) 的忙碌程度。衡量执行纹理采样、过滤、解压的单元的利用率。
- Load/store unit utilization (加载/存储单元利用率): 内存访问和寄存器加载/存储单元的忙碌程度。衡量从缓存或内存中读取和写入数据的单元的利用率。
#### 9. Mali Core Program Property Rate / Workload Property Rate

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512082123792.png)

**Mali Core Program Property Rate：** 着色器程序属性率。这个指标组关注的是 着色器代码本身 的执行效率和内存访问效率。

- **Warp divergence rate (线程束分化率):** 衡量线程束内线程因分支逻辑而走不同代码路径的频率。图示数值0.05%，表明着色器执行效率高，没有因分支而产生大量浪费的周期。
- **All registers warp rate (所有寄存器线程束率):** 衡量线程束因为需要使用所有可用寄存器而被限制在核心中驻留的频率，是衡量寄存器压力的指标。图示数值9.07%意味着大约 9% 的线程束在使用时占用了核心提供的所有寄存器。这会导致寄存器不足，核心无法同时运行更多的 Warps，从而导致性能下降。
- **Shader blend rate (着色器混合率):** 衡量在着色器中执行颜色混合 (Blending) 操作的频率。图示数值 0.00% 表明着色器代码中几乎没有使用复杂的颜色混合逻辑。

**Mali Core Workload Property Rate：** 核心工作负载属性率。这个指标组关注的是 几何体和 Tile 渲染 的效率，特别是 TBDR 架构中的 分块 (Tiling) 和 隐藏表面消除 (HSR) 效率。

- **Partial coverage rate (部分覆盖率) :** 衡量渲染的图元（三角形）只覆盖了部分瓦块 (Tile) 的比例。当一个三角形很大或刚好位于瓦块边缘时，就会发生部分覆盖。如果这个值极高，可能表示场景中有很多极小的三角形，或是渲染目标的分辨率/瓦块大小不匹配。
- **Full warp rate (完全线程束率) :** 衡量几何/计算工作量足以填满整个线程束的比例。这是一个衡量 GPU 填充效率的指标。图示数值 91.7% 为极高。这意味着当 GPU 调度非片段任务时，它几乎总是能够高效地将工作打包成完整的线程束进行执行，充分利用了 SIMD/SIMT 硬件，避免了浪费。
- **Unchanged tile kill rate (未更改瓦块剔除率) :** 衡量 GPU 在渲染帧时，检测到整个瓦块的内容没有变化，从而直接跳过该瓦块的渲染和存储操作的比例。图示数值 0.0% 为极低，可能没有开启或利用 Frame Buffer Fetch (FBF) 或 Tile Write Back 优化，如果场景中有大量的静态 UI 或背景，这个值应该是非零的。

#### 10. Texture相关指标

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512091552151.png)

**Mali Texture Usage**: 衡量纹理单元的活跃程度和不同过滤模式的开销。

- Texture filtering active: 纹理单元总体忙碌的周期数。衡量 TMU (Texture Mapping Unit) 执行纹理操作的总时间。
	
- Full bilinear filter active (全双线性过滤活跃)：执行完整双线性过滤所消耗的周期数。
	
- Full trilinear filter active (全三线性过滤活跃)：执行完整三线性过滤所消耗的周期数。三线性过滤通常用于处理 mipmap 之间的平滑过渡。

**Mali Texture CPI**：CPI (Cycles Per Instruction) 衡量的是纹理操作的效率。具体是指执行每个纹理指令平均消耗的周期数。 理想值接近 1。数值越高，效率越低。

**Mali Texture Workload Property Rate (纹理工作负载属性率)**：衡量纹理采样时的硬件利用率。

- Full speed filter rate (全速过滤率)：纹理过滤操作达到硬件最大吞吐量的比例，越高越好。

**Mali Texture Bus Rate (纹理总线速率)**：衡量纹理数据在总线上的传输效率。

- Input bus utilization (输入总线利用率)：纹理数据被拉入 L1/L2 缓存的总线利用率。
	
- Output bus utilization (输出总线利用率)：纹理过滤结果输出到着色器核心的总线利用率。

**Mali Texture Memory Usage**：衡量纹理数据在各级缓存和外部内存的传输量。 

- L2 read bytes/cycle (L2 读取字节/周期)：GPU 从 L2 缓存读取纹理数据的速率。 
	
- External read bytes/cycle (外部读取字节/周期)：GPU 从显存读取纹理数据的速率。这直接消耗系统带宽。

#### 11. Load/Store Unit相关指标

Load/Store uint负责将数据从缓存/内存加载到寄存器，以及将寄存器数据存储回内存。Load/Store相关指标组用于诊断内存访问和寄存器管理是否造成了性能瓶颈。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512091600714.png)

**Mali Load/Store Usage：** 这个指标组衡量了 Load/store unit自身的忙碌程度以及停顿情况。

- **Load/store unit issue:** 衡量 Load/store unit 因等待资源（如缓存行、总线或内部依赖）而停顿的总周期数。这个值如果很高，也会影响到Execution Core使用率。
	
- **Full read (完整读取)：** 执行完整的 128 位缓存行读取操作的频率。
	
- **Partial read (部分读取) ：** 执行不足 128 位的部分数据读取操作的频率。如果这个值很高，意味着大量 Load/Store 操作读取的数据量小于一个完整的缓存行，效率相对较低。
	
- **Full write (完整写入)：** 执行完整的 128 位缓存行写入操作的频率。
	
- **Partial write (部分写入)：** 执行部分数据写入操作的频率。
	
- **Atomic access (原子访问)：** 执行原子操作（如 Compute Shader 中的原子加法）的频率。

**Mali Load/Store Memory Usage：** 这个指标组衡量了 Load/store unit 从不同层级内存读取和写入数据的带宽。

- **L2 read bytes/cy (L2 读取字节/周期)：** GPU 从 L2 缓存读取数据的速率。
	
- **External read bytes/cy (外部读取字节/周期)：** GPU 从显存读取数据的速率。这直接消耗系统带宽。上图中6.6 bytes/cy是比较高的。持续的 6.6 bytes/cy 外部读取对系统内存带宽造成巨大压力。这也会导致 Load/store unit 的高停顿率。
	
- **L2 write bytes/cy (L2 写入字节/周期)：** GPU 向 L2 缓存写入数据的速率。上图中 56.4 bytes/cy 是极高的。这表示 GPU 产生了大量的写入流量，通常是因为：
    - Tile Memory 的写回
    - 寄存器溢出 产生的存储操作
    - 渲染目标写入

#### 12. Mali Tile Memory Usage

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512091604985.png)

Mali Tile Memory Usage (瓦块内存使用情况)，这个指标组关注的是在 TBDR 架构 中，片段着色器处理完毕后，将颜色、深度、模板等数据从 Local Tile Memory（本地瓦块内存） 写入 L2 缓存和显存 的流量。

Tile write bytes/px (瓦块写入字节/像素) ：衡量 GPU 在渲染每个最终像素时，向内存写入的平均字节数。通常，如果只渲染一个颜色目标（如 RGBA8888），理论值应为 4 字节/像素。

#### 13. Memory相关指标

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512091607439.png)

上图展示了关于 Core 内存带宽 的三个关键指标组。这些指标按数据流向和层级，精细衡量了 L2 缓存和外部内存的读取和写入流量（以 Gibi-bytes/秒为单位）。

**Mali Core L2 Memory Reads：** 衡量数据从 L2 缓存读取到 Shader Core 内部各单元的流量。

- **Front-end unit bytes：** 供几何体处理前端（Tiler/Non-frag Front-end）使用的 L2 读取量。这包括属性数据和部分几何体工作集。
	
- **Load/store unit bytes：** 供 Load/store unit 访问的 L2 读取量：包括寄存器溢出数据、Uniforms、片段着色器的输入数据等。
	
- **Texture unit bytes：** 供 Texture unit 访问的 L2 读取量。

**Mali Core External Memory Reads：** 衡量数据直接从 外部显存 读取到 GPU 的流量。这是消耗系统带宽的关键指标。

- **Front-end unit bytes：** 供几何体处理前端使用的显存读取量。 主要为 Attributes 和 Geometry Working Set。
	
- **Load/store unit bytes：** 供 Load/store unit 访问的显存读取量。 主要为 L2 缓存未命中的数据、寄存器溢出数据、以及其他非缓存数据。
	
- **Texture unit bytes：** 供 Texture unit 访问的显存读取量。 即 L2 缓存未命中的纹理数据。

**Mali Core Memory Writes：** 衡量数据从 GPU 写入到 L2 缓存和外部 DDR 内存的流量。

- Load/store unit bytes：Load/store unit 产生的写入量。包括寄存器溢出写入、渲染目标写入等。
	
- Tile unit bytes：Fragment Back-end 将渲染结果从 Tile Memory 写回到 L2/DDR 的流量。

到这里Streamline的性能指标基本就介绍完了。可以看到，理解了Arm Mali GPU的工作流程之后，再看这些指标其实是非常容易理解的。剩下的工作可能就是拿到一份Streamline的数据报告之后，如何根据这些指标来分析出程序的性能瓶颈，并找出解决方案了。这个我们会在下一节进行介绍。

### 3.1.2 Frame Advisor (帧级瓶颈分析)

**Frame Advisor** 是 Arm Performance Studio 提供的一个**自动化分析工具**，它基于 StreamLine 采集到的数据，提供高层次、可操作的优化建议。

Frame Advisor可以分析单帧或一段时间内的性能数据，并根据 **Arm 最佳实践**和 **TBDR 架构特点** 自动诊断瓶颈类型，快速识别瓶颈是属于 **CPU-Bound（比如高 Draw Call、线程同步）** 还是 **GPU-Bound（带宽瓶颈、计算瓶颈）**，并指出具体的优化方向，而无需用户手动解读复杂的硬件计数器。

**关键功能：**

- **瓶颈分类：** 将问题分类为 **Fragment-Bound (片段着色瓶颈)**、**Geometry-Bound (几何体瓶颈)**、**Bandwidth-Bound (带宽瓶颈)** 等。
	
- **优化建议：** 为识别出的瓶颈提供具体的、量化的优化建议（例如：“过度绘制严重，建议启用深度预通道”）。

下图是一个Frame Advisor的分析结果界面：

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512091609542.png)

**1. 整体数据统计 (Render Pass Metrics)**

报告顶部的表格提供了渲染通道（Render Pass）级别的宏观数据：

- `Frame`：当前分析的帧编号（这里是 188）
    
- `Render pass`/`Call`：渲染通道、DrawCall 的调用次数
	   
- `Prims`：当前调用处理的**图元数量**（如三角形、线段）
	
- `Unique Indices`：唯一顶点索引数（反映索引复用情况）
	
- `Vert size`：单个顶点的**数据字节大小**（这里有 527、336 等，最大 96 字节）
	
- `VSE (Vertex Shading Efficiency)`: 顶点着色效率。
    
- `VME (Vertex Memory Efficiency)`: 顶点内存效率（如 0.90 代表有 10% 内存浪费）。


**2. 按管线阶段统计 (Per-Stage Metrics)**

报告下方的表格提供了每个着色器管线阶段（Vertex, Fragment, Compute）的详细利用率统计：

- `LS（Local Size）`：计算着色器的**本地工作组大小**（影响并行度）
	
- `Occupancy`：GPU 计算核心的**占用率**（反映硬件资源利用程度）
	
- Work regs (工作寄存器): 该着色器需要的寄存器数量。
    
- Uniforms (统一变量): 该着色器使用的统一变量数量。
    
- Stack spills (堆栈溢出): 衡量寄存器溢出到内存的频率。高溢出会导致严重的 Load/store unit issue。
    
- FP16 usage (FP16 使用): 衡量 16 位浮点精度的使用情况（这里是 0，代表未利用低精度优化）。


**3. 数据分析及建议 (Detailed Metrics)**

这是优化的核心参考区，分 4 个维度：

- **Mesh complexity（网格复杂度）**
    - `Shader vertices`：顶点着色器处理的总顶点数（5480）
    - `Primitive count`：图元总数（452）
    - `Vertex size`：单个顶点的字节数（76 字节，偏大）
    - `Mesh bandwidth`：网格数据的内存带宽消耗（5376600 字节）
    
- **Mesh efficiency（网格效率）**
    - `Vertex shading efficiency`：顶点着色效率（0.00，严重低效）
    - `Duplicate vertices`：重复顶点数（5255，占比 99.98%，几乎全重复）
    - `Temporal/Spatial locality`：时间 / 空间局部性（82.45/50.90，偏低，缓存复用差）
    
- **Mesh encoding efficiency（网格编码效率）**
    - `Index rate`：索引复用率（0.55，偏低，索引未充分复用）
    
- **Attribute stream efficiency（属性流效率）**
    - `Mesh bandwidth`：实际带宽是 “理想带宽” 的 1495.4%（带宽浪费超 14 倍）

可以很轻松的从Detailed Metrics中看出，这个 Draw Call 存在主要的内存瓶颈，这两个问题严重影响了 GPU 的效率：

- 致命的重复顶点问题： 99.98% 的顶点是重复的，直接导致 VSE 几乎为 0.00，造成 GPU 核心进行了大量不必要的着色计算和数据传输。
    
- 低效的内存使用： 96 字节的大顶点大小、20 字节的填充。
	
- 糟糕的局部性 。

Q：**Temporal/Spatial locality（时间 / 空间局部性）**

这两个指标是**GPU 缓存优化的核心**（直接影响内存带宽和渲染效率），本质是衡量 “顶点 / 网格数据被 GPU 缓存复用的效率”——GPU 访问片上缓存（快、低功耗）比访问显存（慢、高功耗）快 100 倍以上，局部性越高，缓存命中率越高，性能越好。

1. Spatial Locality（空间局部性）

- **定义**：访问某块数据后，**相邻数据被快速访问的概率**（GPU 按 “缓存行”（如 64 字节）读取数据，空间局部性高 = 缓存行里的数据都能被用上，不浪费）。
	
- **通俗举例**：
	假设顶点数据是 “顶点 A→顶点 B→顶点 C” 按顺序存在内存里，GPU 读顶点 A 时，会把相邻的 B、C 也载入缓存；若渲染时刚好按 A→B→C 的顺序处理，缓存里的 B、C 直接用（空间局部性高）；若渲染顺序是 A→Z→B，缓存里的 B、C 没用上，还要重新读 Z（空间局部性低）。
    
- **图中场景（50.90 偏低）**：
    50.90 意味着只有 50.9% 的缓存行数据被有效利用，剩下近 50% 的缓存空间浪费（比如顶点数据碎片化、排列混乱），导致 GPU 频繁从显存重读数据，带宽浪费严重。

2. Temporal Locality（时间局部性）

- **定义**：**同一数据在短时间内被重复访问的概率**（数据被载入缓存后，短时间内多次复用，不用重新读显存）。
	
- **通俗举例**：
	同一批顶点数据，若 1 帧内被多个 DrawCall / 渲染通道重复使用（比如同一模型的不同 LOD、同一纹理的多次采样），缓存不会失效，直接复用（时间局部性高）；若顶点数据只用 1 次就被淘汰出缓存，下次用还要重读（时间局部性低）。
    
- **图中场景（82.45 尚可）**：
    82.45 说明大部分数据在短时间内被复用，但仍有 17% 左右的缓存失效，可通过 “合并 DrawCall、复用顶点缓冲” 进一步优化。

3. 怎么优化这两个指标？

| 局部性类型           | 优化方向                                                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 空间局部性（50.90 偏低） | 1. 重排顶点数据：按 “图元绘制顺序” 打包顶点（比如三角形的 3 个顶点连续排列）；<br><br>2. 对齐数据格式：顶点属性按 4/8 字节对齐（匹配 GPU 缓存行大小）；<br><br>3. 合并小顶点缓冲：避免碎片化的小缓冲导致缓存行浪费。 |
| 时间局部性（82.45）    | 1. 复用顶点 / 索引缓冲：同一模型的多次绘制复用同一缓冲，不重复创建；<br><br>2. 减少缓冲切换：连续 DrawCall 尽量用相同的顶点缓冲，降低缓存失效概率；<br><br>3. 合并 DrawCall：减少批次切换带来的缓存刷新。    |
> Q：顶点缓冲怎么理解？如何确保连续 DrawCall 尽量用相同的顶点缓冲？

**顶点缓冲（Vertex Buffer）怎么理解？**

顶点缓冲是**GPU 显存里的一块 “连续内存区域”**，专门用来存**顶点的所有属性数据**（比如每个顶点的位置、法线、纹理坐标、颜色等）。

可以把它比作：

- 你要组装一批玩具（渲染多个模型），“顶点数据” 是玩具的零件（螺丝、积木块）；
- 顶点缓冲就是你提前把所有零件按顺序放进的 “大货架”（连续、规整）；
- GPU 渲染时，直接从这个 “货架” 里取零件，不用每次从杂乱的抽屉（CPU 内存 / 零散显存）里找 ——一次把零件摆好，多次复用，既省时间又省力气。

它的核心作用是：

- 避免 CPU→GPU 的重复数据传输（顶点数据只传一次到缓冲，后续渲染直接用）；
-  让 GPU 能 “连续读取” 数据（提升空间局部性，缓存命中率更高）；
-  支持顶点复用（配合索引缓冲，多个模型共享同一个缓冲里的顶点）。

**如何确保连续 DrawCall 尽量用相同的顶点缓冲？**

核心思路是：把多个模型的顶点数据 “打包” 进同一个顶点缓冲，让连续的 DrawCall 从这个缓冲的不同区域读数据，避免频繁切换缓冲。

具体方法有 4 个：

1. 合并多个模型的顶点数据到同一个缓冲

把多个小模型的顶点数据，按顺序 “拼接” 到同一个顶点缓冲里，每个模型对应缓冲里的一个**数据段**（用 “偏移量 + 长度” 标记）。

举个例子：

- 模型 A 的顶点数据存在缓冲的`0~1023字节`区域；
- 模型 B 的顶点数据存在缓冲的`1024~2047字节`区域；
- 连续执行 “渲染模型 A”“渲染模型 B” 的 DrawCall 时，只需要告诉 GPU：“从同一个顶点缓冲的`0`位置读模型 A，从`1024`位置读模型 B”——**全程不用切换缓冲，GPU 直接从同一个 “货架” 的不同层取零件**。

2. 用索引缓冲配合，复用顶点缓冲里的顶点

如果多个模型共享部分顶点（比如重复的 UI 元素、同类型的地图瓦片），不用重复存顶点数据 —— 把共享的顶点存在同一个缓冲里，用**索引缓冲（Index Buffer）** 标记 “每个模型用哪些顶点”。

比如：

- 顶点缓冲里存了 “正方形的 4 个顶点”；
- 渲染 10 个正方形时，只用同一个顶点缓冲，通过不同的索引组合（重复用这 4 个顶点）就能渲染出 10 个正方形 ——**连续 10 个 DrawCall 都用同一个顶点缓冲，既省内存又省带宽**。

3. 用 “实例化渲染（Instanced Rendering）” 批量复用

如果要渲染**多个完全相同的模型**（比如批量的树木、粒子、按钮），只需要：

- 把 “基础模型的顶点数据” 存 1 份到顶点缓冲；
- 额外传一份 “实例数据”（每个模型的位置、缩放、颜色）；
- 一个 DrawCall 就能渲染所有实例 ——**全程只用同一个顶点缓冲，连 DrawCall 都不用多调用，效率最高**。

4. 按 “顶点缓冲” 分组安排 DrawCall 的执行顺序

渲染前，把**使用同一个顶点缓冲**的 DrawCall 排在一起执行，避免 “用一个缓冲→切另一个→再切回原缓冲” 的频繁切换。

比如：

- 原本的 DrawCall 顺序是：“用缓冲 A 渲染→用缓冲 B→用缓冲 A→用缓冲 B”；
- 调整后顺序：“用缓冲 A 渲染 2 次→用缓冲 B 渲染 2 次”——**减少缓冲切换的开销，同时提升 GPU 缓存的复用率（空间 / 时间局部性都会变好）**。

**总结**

复用顶点缓冲的核心是 “**把多个模型的顶点数据集中存、重复用**”，这样连续 DrawCall 就能共享同一个缓冲 —— 这刚好能解决你之前遇到的 “顶点重复、带宽浪费、缓存局部性低” 的问题，是提升渲染性能的关键技巧之一。


### 3.1.3 Mali Offline Compiler (着色器代码优化)

**Mali Offline Compiler**是一个**静态分析工具**，用于在不实际运行程序的情况下，评估着色器代码的效率。

该工具接收着色器源码（GLSL/HLSL），并模拟 Mali GPU 硬件，给出着色器编译后的详细报告。它可以帮助开发者识别着色器代码中潜在的 **GPU 计算瓶颈**，例如**高寄存器使用量**、**指令周期数过多**、**线程束分化风险**等。

**关键功能：**

- **寄存器使用报告：** 报告顶点和片段着色器所需的寄存器数量，帮助预测**寄存器溢出**和**低占用率**的风险。
    
- **周期/指令计数：** 预估着色器执行所需的 ALU 周期数和指令数量，指导开发者简化着色器逻辑。
    
- **性能热点识别：** 标记出着色器代码中可能导致性能损失的复杂指令（如 SFU 使用、复杂数学运算）。

如下是对一个Compute Shader进行的离线分析结果：

```shell
./malioc --compute test.comp.spv -d                              
Mali Offline Compiler v8.8.0 (Build 888cd7)
Copyright (c) 2007-2025 Arm Limited. All rights reserved.

Configuration
=============

Hardware: Mali-G1 r0p1
Architecture: Arm 5th Generation
Driver: r55p0-00rel0
Shader type: Vulkan Compute

Main shader
===========

Work registers: 64 (100% used at 50% occupancy)
Uniform registers: 72 (56% used)
Shared storage size: 0 bytes
Stack use: 96 bytes
 - Spill region: 84 bytes
16-bit arithmetic: 0%

                                A     FMA     CVT     SFU      LS       T    Bound
Total instruction cycles:    3.67    2.42    1.09    1.25   52.00    0.00       LS
Shortest path cycles:        0.02    0.00    0.02    0.00    0.00    0.00   A, CVT
Longest path cycles:          N/A     N/A     N/A     N/A     N/A     N/A      N/A

A = Arithmetic, FMA = Arith FMA, CVT = Arith CVT, SFU = Arith SFU,
LS = Load/Store, T = Texture

Shader properties
=================

Has uniform computation: true
```

这份报告指出了两个关键的性能问题：

1. Work registers (工作寄存器）：着色器使用了 GPU 提供的所有 64 个工作寄存器，达到 100% 利用率。这直接将 核心占用率 (Occupancy) 限制在 50%。这意味着 GPU 无法同时运行更多的线程束（Warp），损失了一半的并行度。
    
2. Spill region (溢出区域)：存在寄存器溢出。 由于寄存器压力巨大 (100% used)，编译器将 84 字节的数据溢出到了内存。这会导致额外的 Load/Store unit issue 和内存带宽消耗，进一步加剧性能瓶颈。

因此我们可以根据这个报告，来优化对应的Shader。比如上面这个Shader，就可以通过将大型Shader拆分成小Shader来降低复杂度，从而减少寄存器的使用。

## 3.2 Snapdragon Profiler

**SnapDragon Profiler** 是一款由高通 (Qualcomm) 开发的，专用于分析和优化基于 Snapdragon 平台 的移动应用性能的专业工具。

与专注于 Arm Mali GPU 的 Streamline 不同，Snapdragon Profiler 提供了一个更广泛、更深入的视角，它能够同时收集 CPU、GPU (Adreno)、DSP 和系统级 的实时数据。

如果是优化运行在骁龙芯片上的应用，Snapdragon Profiler 就是最准确、最全面的诊断工具，因为它直接利用了高通硬件的底层接口。

相比于Streamline会获取Mali 特有的硬件计数器，如 Tile Write Back、Warp、Varying Usage等，Snapdragon Profiler会获取Adreno 特有的硬件计数器，如 Binning、Z-cull 等。对于具体硬件指标，感兴趣的同学可以到[高通官网](https://www.qualcomm.com/developer/software/snapdragon-profiler)上去学习，在此我们就不详细介绍了。

## 3.3 AGI

Android GPU Inspector (AGI) 是一款由 Google 开发的，专为 Android 平台设计、用于深度分析和调试 GPU 性能的官方工具。

AGI 的目标是提供一个统一的、跨越不同 GPU 硬件（包括 Arm Mali、Qualcomm Adreno、IMG PowerVR 等）的性能分析框架，帮助开发者在 Android 设备上高效地识别和解决渲染瓶颈。

AGI 提供了一套全面的工具集，包括系统级追踪、帧剖析 (Frame Profiling) 和渲染 API 调试，特别适用于 Vulkan 应用程序的性能分析和优化。


---
# 4 FPS优化

针对 Android Vulkan 程序性能优化的第一步，就是是要分析性能瓶颈在哪里。分析性能瓶颈是一个系统性的过程，因为它涉及到 CPU 线程调度、Vulkan API 开销、GPU 渲染管线以及内存带宽等多个方面。

因此我们首先来介绍下如何进行性能瓶颈分析。

## 4.1 性能瓶颈分析

性能瓶颈通常可以分为三个主要类别：CPU 瓶颈、GPU 计算瓶颈（Non-fragment or Fragment）和 GPU 带宽瓶颈。

首先，我们要识别当前Vulkan程序是CPU瓶颈还是GPU瓶颈，最简单的方式是在在渲染循环函数中打点计时。通常Vulkan程序的渲染循环代码如下：

```javascript
// 获取当前帧使用的交换链图像
AcquireNextImageFromSwapchain();

// CPU对当前帧进行：数据更新 & 命令录制
UpdateAndRecordCommands(current_frame, image_index_);

// 等待GPU完成上一帧绘制
WaitGPUFrameRenderEnd();

// 提交下一帧的渲染命令
SubmitRenderCommands();

// 提交下一帧呈现
QueuePreset();
```

在Vulkan程序渲染帧率达不到预期的情况下（比如实时帧率只有20FPS，预期是30FPS），可以针对上面5个步骤进行打点计时：

1. 如果等待GPU完成上一帧绘制耗时较久，则说明是GPU瓶颈。此时可以在录制GPU命令时通过添加vkCmdWriteTimestamp命令来统计每个Pipeline的执行耗时，从而确定具体哪个管线是性能瓶颈，从而能进行针对性的更深入的瓶颈分析。
    
2. 如果是其他4个步骤耗时较久，则说明是CPU瓶颈。此时可以在CPU侧进行更细致的各个阶段的耗时统计，找到真正造成瓶颈的操作，从而进行针对性的优化。


## 4.2 CPU瓶颈

CPU瓶颈产生的原因比较多，下面为大家介绍下比较常见的问题。

### 4.2.1 滥用同步API

帧渲染循环里，除了WaitGPUFrameRenderEnd（等待GPU完成上一帧绘制外），不能使用任何的同步API，包括：

- vkQueueWaitIdle
    
- vkDeviceWaitIdle
    
- vkWaitFences

这些同步API会等待GPU完成特定任务，在此之前会一直阻塞CPU线程，从而影响CPU的吞吐量。

### 4.2.2 未使用多帧并行渲染

**多帧并行渲染** 是现代图形管线中实现高效并行的核心模式，在这种模式下，CPU和GPU会并行执行不同帧的任务。

**1. 传统串行模式 (单线程) 的低效**

在 OpenGL ES 等老一代 API 中，驱动程序通常是 **单线程** 且具有很强的 **隐式同步**。

- **流程:** CPU (Frame N) → GPU (Frame N)
    
- **问题:** CPU 必须等待 GPU 完成当前帧的大部分工作，才能开始下一帧（Frame N+1）的 CPU 工作，这导致 CPU 或 GPU 经常处于 **空闲等待** 状态，产生 **渲染气泡 (Render Bubbles)**。


**2. 多帧并行模式 (异步执行) 的原理**

这种模式的核心思想是允许 **CPU 始终领先于 GPU 几个帧 (N, N+1, N+2, ...)** 执行任务，将 CPU 和 GPU 的工作流解耦。

- **CPU (生产者):** 负责准备 **Frame N+1** 的渲染命令（Draw Calls, 状态设置、资源上传等）。
    
- **GPU (消费者):** 负责执行 **Frame N** 的渲染命令。

通过这种方式，CPU 和 GPU 几乎可以持续保持满载工作，隐藏了彼此的延迟。


**3. 最佳实践：三级流水线 (Triple Buffering)**

最高效的多帧并行通常采用 **三级流水线**（或三帧延迟）：

|阶段|CPU 工作 (生产者)|GPU 工作 (消费者)|
|:--|:--|:--|
|**时刻 T**|录制 **Frame N+2** 命令|渲染 **Frame N**|
|**时刻 T+1**|录制 **Frame N+3** 命令|渲染 **Frame N+1**|
|**时刻 T+2**|录制 **Frame N+4** 命令|渲染 **Frame N+2**|

### 4.2.3 动画计算耗时

Vulkan 程序中 3D 骨骼骨骼动画计算：**骨骼蒙皮（Skinning）计算** 或 **骨骼矩阵生成（Bone Matrix Calculation）** 会占用大量的 CPU 资源。这是一个非常常见的性能瓶颈，尤其是在移动平台上，如果场景中包含大量蒙皮角色，CPU 很容易成为瓶颈。

**1. 蒙皮矩阵计算耗时**

这种场景下，CPU 耗时的核心原因在于它会执行大量的**矩阵乘法**和**向量变换**操作。对于场景中的每个角色和动画帧，CPU 需要计算出一系列用于最终顶点变换的矩阵：

- **局部变换矩阵（Local Transforms）：** 读取动画数据，计算每个骨骼相对于其父骨骼的平移、旋转、缩放。
    
- **全局变换矩阵（Global Transforms）：** 将局部变换递归地与父骨骼的全局变换矩阵相乘，得到每个骨骼相对于模型空间的变换矩阵（即骨骼的最终世界矩阵）。
    
- **蒙皮矩阵（Skinning Matrices）：** 最后，将全局变换矩阵与骨骼的**逆绑定姿态矩阵（Inverse Bind Pose Matrix）** 相乘，得到最终用于顶点着色器的蒙皮矩阵集合。

**2. 优化方案**

CPU不擅长矩阵和向量运算，但是可以做以下优化：

- **SIMD/Neon 优化：** 使用 ARM NEON/SIMD 指令集或对应的数学库（如 GLM 的 Neon 优化版本）对矩阵乘法和向量变换进行汇编级优化，一次处理多个浮点数。
    
- **并行计算：** 利用多线程，将不同角色的骨骼计算分摊到多个 CPU 核心上并行执行。
    
- **脏位追踪：** 只计算在当前帧中发生变换的骨骼，避免对静态骨骼进行不必要的重复计算。

除此之外，也可以尝试将蒙皮矩阵的计算转移到到GPU侧执行，因为GPU更擅长矩阵和向量计算。这是最有效的优化方式，可以完全消除 CPU 端的矩阵计算瓶颈。

### 4.2.4 Draw Call 提交开销

尽管单个 Draw Call 的 CPU 开销在 Vulkan 中比 OpenGL ES 小得多，但如果一个渲染帧中包含 **数千个 Draw Call**，它们累积的总 CPU 时间仍会成为瓶颈。

在 Vulkan 中，一个 Draw Call (例如 `vkCmdDrawIndexed`) 往往需要伴随一系列状态设置：

- **频繁的状态切换：** 每次切换 Pipeline (管线)、Descriptor Set (描述符集)、或 Vertex/Index Buffer (顶点/索引缓冲) 时，CPU 都需要记录新的命令，这会消耗 CPU 时间。
    
- **低效的渲染排序：** 如果 Draw Call 顺序混乱，频繁地切换渲染状态，会抵消 Vulkan 低开销的优势。例如，交替渲染不透明物体和透明物体，或者无序地切换材质。
    
- **Vulkan 特有的开销：** 尽管 Vulkan 做了优化，但每次调用 `vkCmdDraw*` 都会产生 CPU 侧的指令录制开销。

**优化方案：**

1. **Batching (批处理) / 合并**： 将使用相同 Pipeline、相同材质、相同 Descriptor Set 的几何体合并到单个 Draw Call 中，这样可以大幅减少 Draw Call 数量。
    
2. **Instancing (实例化)** 渲染：对于渲染大量相同的物体（如草、粒子）但位置不同的情况，可以使用 `vkCmdDrawIndexedIndirect` 或 `vkCmdDrawIndexed` 的 `instanceCount` 参数。这样可以将数千次 Draw Call 降至一次。

### 4.2.5 没有按最佳实践使用Vulkan API

Vulkan API 提供了极大的灵活性，但也要求开发者进行**显式同步**和**资源管理**。如果 API 调用不当，会导致驱动程序或验证层执行额外的、不必要的 CPU 工作。

- **Descriptor Set (描述符集) 的频繁更新：** 在 Draw Call 之间频繁调用 `vkUpdateDescriptorSets` 来更换 Uniform Buffer 或 Sampler。这会使驱动程序不断追踪和管理新的资源绑定。
    
- **Command Buffer (命令缓冲区) 的录制：** 频繁地创建、重置和销毁 Command Buffer，而不是重用它们，会增加 CPU 内存分配和清理的开销。
    
- **Validation Layer (验证层) 的开销：** 在开发模式下，Vulkan 验证层提供了强大的错误检查，但这会消耗大量 CPU 时间。虽然不应在发布版本禁用，但在性能测试时需要清楚其影响。
    

**优化方案：**

|优化策略|Vulkan 实现方法|目的|
|:--|:--|:--|
|**Descriptor Set 分级更新**|**不频繁更新：** 将全局不变或每帧不变的资源（如相机矩阵、天空盒）放在单独的 **Descriptor Set 0**，只更新一次。|最小化 `vkCmdBindDescriptorSets` 的调用次数。|
|**使用 Push Constant**|对于 **小而频繁变化** 的数据（如每个 Draw Call 的世界矩阵、材质 ID），使用 **Push Constant** (最大 128/256 字节) 代替 Uniform Buffer。Push Constant 的更新开销极低。|避免频繁的 Descriptor Set 绑定和 Uniform Buffer 内存映射开销。|
|**管线屏障的精准使用**|确保 `vkCmdPipelineBarrier` 只在绝对必要的资源转换时使用。使用 **最优的 `srcStageMask` 和 `dstStageMask`**，精准同步所需的管线阶段。|避免驱动程序不必要的 CPU/GPU 同步。|
|**重用命令缓冲区**|对于静态或变化较少的场景部分，预先录制 Command Buffer，然后在每一帧中直接提交 (Submit) 并重用，**避免重复录制**。|消除每一帧的 Command Buffer 录制开销。|
|**多线程 Command Buffer 录制**|将场景划分为多个部分，使用 **多个 CPU 线程** 并行录制 Command Buffer。最后，在一个主线程中提交这些次级 Command Buffer。|将 CPU 渲染工作分摊到多核，充分利用现代移动 CPU 的多线程能力。|

按照这些最佳实践，可以将 CPU 瓶颈从渲染线程中解除，从而让 CPU 能够始终领先 GPU 几个帧，实现最高效的 **多帧并行渲染模式**。

## [](https://km.woa.com/articles/show/644635?kmref=article_recommend#43-gpu%E7%93%B6%E9%A2%88)4.3 GPU瓶颈

如果通过第一步的性能瓶颈分析，初步确认了是GPU瓶颈的话，我们需要进一步分析是卡在了GPU内部哪个环节上。

以ARM Mali GPU为例，下图清晰的展示了Execution Core里，每个硬件单元负责什么事情：

![](https://km.woa.com/asset/000100022511009b3bde6fb9bc4e9201?height=863&width=862&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

- Processing Unit：FMA 处理复杂的数学运算、CVT 处理简单的数学运算、SFU 处理特殊功能。
    
- Load/Store Unit: 加载/储存单元，负责和纹理采样以外的内存访问：比如内存访问、缓存区访问。它通常配备专属的 L1 缓存。
    
- Varying Unit: 插值单元，比如Vertex Shader 里的数据，插值后才会传递给片元着色器。
    
- ZS/Blend Unit: 混合单元，负责将颜色写入缓存中。
    
- Texture Unit: 纹理单元，负责所有的纹理内存访问。
    

如果是 GPU 瓶颈，那就一定是以上五个单元出了问题，在 **Arm Mali Streamline** 中应该查看各自单元对应的关键指标:

**1. Processing Unit** (ALU / Arithmetic)

|关键指标 (Streamline/AGI)|理想状态 (非瓶颈)|瓶颈特征 (高利用率或高停顿)|
|:--|:--|:--|
|**Arithmetic unit utilization** (ALU 利用率)|较低（例如 < 50%）。|**高 (>70%)**：GPU 是 **计算限制 (Compute Bound)**。意味着着色器中的数学指令过多、过于复杂，ALU 单元是速度限制器。|

**2. Load/Store Unit (LS)**

|关键指标 (Streamline/AGI)|理想状态 (非瓶颈)|瓶颈特征 (高利用率或高停顿)|
|:--|:--|:--|
|**Cycles/Fragment Thread**|低（例如 < 10 cycles）。|**高**：表明片段着色器计算量大，是 ALU 瓶颈的直接原因。|
|**Load/store unit issue** (LS 停顿周期)|极低（理想接近 0 cycles）。|**极高 (例如 428M cycles)**：GPU 是 **内存访问限制 (Memory Access Bound)**。核心在等待 LS 单元完成数据加载、存储或处理寄存器溢出。|
|**Load/store unit utilization** (LS 利用率)|低于 ALU 利用率。|**相对高**：意味着 LS 单元频繁地处理寄存器溢出、Uniforms、或 **Varying 数据** 相关的内存请求。|

**3. Varying Unit** (插值单元)

|关键指标 (Streamline/AGI)|理想状态 (非瓶颈)|瓶颈特征 (高利用率或高停顿)|
|:--|:--|:--|
|**Varying unit issue** (Varying 停顿周期)|极低。|**高 (例如 157M cycles)**：GPU 是 **插值或瓦块内存带宽限制 (Varying Bound)**。通常由 **32-bit interpolation active** 过高引起，导致瓦块内存和 Load/Store 单元压力过大。|
|**32-bit interpolation active**|低。|**高**：是 Varying Unit Issue 的主要原因。应优化为 **16-bit interpolation**。|

**4. ZS/Blend Unit** (混合单元/片段后端)

|关键指标 (Streamline/AGI)|理想状态 (非瓶颈)|瓶颈特征 (高利用率或高停顿)|
|:--|:--|:--|
|**Tile write bytes/px** (瓦块写入字节/像素)|接近渲染目标理论值（例如 RGBA8888 目标为 4 bytes/px）。|**高 (例如 9.58 bytes/px)**：GPU 是 **填充率限制 (Fillrate Bound)** 或 **带宽限制**。表明每个像素的最终写回成本极高，通常是由于多渲染目标 (MRT) 或高精度帧缓存格式。|
|**Fragment Warps** (片段工作总量)|根据分辨率和 Overdraw 变化。|**极高**：暗示 **Overdraw (过度绘制)** 严重，导致 ZS/Blend Unit 需要处理大量冗余像素。|

**5. Texture Unit** (纹理单元)

|关键指标 (Streamline/AGI)|理想状态 (非瓶颈)|瓶颈特征 (高利用率或高停顿)|
|:--|:--|:--|
|**Texture filtering active** (纹理过滤活跃周期)|根据纹理采样量变化。|**高**：意味着纹理采样是 GPU 工作的主要部分。|
|**Texture CPI** (每指令周期数)|接近 1.0 (理想)。|**高**：表明纹理访问延迟高，可能由于 **缓存不命中** 导致。|
|**External read bytes/cy (Texture)**|极低。|**高**：表明纹理数据必须从慢速的 **DDR 外部内存** 中读取，导致带宽瓶颈。|

判断和区分GPU瓶颈，可以按照以下方式流程：

1. **检查利用率:** 查看 **Shader Core Utilization (核心利用率)** 是否接近 100%。如果是，则存在 GPU 瓶颈。
    
2. **检查 ALU 饥饿：** 查看 **Arithmetic unit utilization**。
    
    - **如果高：** 瓶颈是 **Processing Unit (计算限制)**。
        
    - **如果低 (<15%) 且核心利用率高：** 瓶颈是 **等待数据**。转向第 3 步。
        
3. **检查数据瓶颈 (Load/Store & Varying):** 检查 **Load/store unit issue** 和 **Varying unit issue**。
    
    - **如果高：** 瓶颈是 **内存带宽/延迟**。检查 **32-bit interpolation active** 和 **External read bytes/cy** 以定位根源。
        
4. **检查Fragment-end (ZS/Blend):** 检查 **Tile write bytes/px** 和 **Fragment Warps**。
    
    - 如果这些指标异常高，瓶颈是 **填充率/过度绘制**。
        
5. **检查纹理 (Texture):**
    
    - 如果上述瓶颈都不明显，但 **Texture filtering active** 或 **Texture CPI** 高，则瓶颈可能是 **纹理单元**。
        

## [](https://km.woa.com/articles/show/644635?kmref=article_recommend#44-gpu%E8%AE%A1%E7%AE%97%E7%93%B6%E9%A2%88)4.4 GPU计算瓶颈

GPU计算瓶颈的核心特征是：GPU的大部分时间都花在执行着色器指令上，等待数据输入或输出的时间相对较少。着色器核心（ALUs, 算术逻辑单元）的利用率高，但吞吐量受限于每秒能执行的指令数量。

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#441-%E7%9D%80%E8%89%B2%E5%99%A8%E5%A4%8D%E6%9D%82%E5%BA%A6%E4%B8%8E%E6%8C%87%E4%BB%A4%E6%95%B0%E9%87%8F%E9%97%AE%E9%A2%98)4.4.1 着色器复杂度与指令数量问题

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#1-%E5%A4%8D%E6%9D%82%E7%9A%84%E6%95%B0%E5%AD%A6%E8%BF%90%E7%AE%97-%E9%AB%98-alu-%E8%B4%9F%E8%BD%BD)1. 复杂的数学运算 (高 ALU 负载)

片段着色器或计算着色器中包含了大量的复杂数学运算。例如，在片段着色器中执行昂贵的 **PBR (物理渲染)** 光照模型、复杂的阴影滤波、或多重采样，导致 ALU 利用率饱和。

**优化方案：简化算法 + 预计算**

- **近似计算：** 在视觉效果可接受的前提下，用更简单、更快的数学模型（如用 Blinn-Phong 或更简化的模型代替全 PBR）来替代精确但复杂的计算。
    
- **查找表 (LUT)：** 将复杂的、静态的计算结果（例如环境光照或散射项）预先烘焙并存储在 **查找纹理 (LUT)** 中。运行时只需一次纹理采样即可获取结果，大大减少了 ALU 的实时计算量。
    
- **将计算移到 CPU：** 将每帧不变或变化缓慢的计算（如某些矩阵运算、常量因子计算）移到 CPU 侧进行预处理，作为 Uniform 数据传递给 GPU。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#2-%E7%B2%BE%E5%BA%A6%E6%B5%AA%E8%B4%B9-32-%E4%BD%8D%E6%BB%A5%E7%94%A8)2. 精度浪费 (32 位滥用)

对 **非必须高精度** 的计算（如颜色、UV 坐标、法线）使用了 32 位浮点数。在 Arm Mali 等移动 GPU 上，32 位操作通常比 16 位操作消耗更多的 ALU 资源、寄存器空间和指令周期。

**优化方案：使用 FP16/`mediump`：**

- 在 GLSL 着色器代码中，将所有不要求高精度的变量（如纹理坐标、非线性颜色、法线和切线）明确声明为 **`mediump`**（对应 16 位浮点数）。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#3-%E4%B8%8D%E5%BF%85%E8%A6%81%E7%9A%84%E6%8C%87%E4%BB%A4%E6%89%A7%E8%A1%8C-%E5%88%86%E6%94%AF)3. 不必要的指令执行 (分支)

着色器中存在大量复杂的 **动态分支 (`if`/`else`、`switch`)**。在 SIMD/SIMT 架构下，同一线程束 (Warp/Wavefront) 中的所有线程必须执行所有分支路径的指令，导致实际执行的指令数量远超逻辑指令数量，造成 ALU 性能损失。

**优化方案：常量化与分支消除**

**3.1 使用数学函数代替**

尽量使用数学函数（如 `mix`、`step`、`clamp`）代替简单的 `if/else` 语句。这些函数可以确保所有线程并行执行相同的指令，避免性能损失。

**3.2 使用 Uniform 分支**

**Uniform 分支** 是指着色器中的分支条件**只依赖于 Uniform 值、Push Constant 值或着色器常量**（即在 Draw Call 内部是不变的）。目标是确保编译器和 GPU 知道分支的结果是**在运行时固定不变**的，从而避免分支发散。

**编译器优化：** 现代 GPU 编译器（如 Mali 离线编译器）非常擅长识别这种分支。由于条件在整个 Draw Call 中是固定的，编译器可以：

1. **静态消除 (Static Elimination)：** 编译器可以在运行时检查 Uniform/常量的值，如果可能，它会编译出两个独立的指令流，并在运行时只执行其中一个。
    
2. **避免发散：** 即使不能完全消除，GPU 硬件也知道这个分支在整个 Warp 中不会发散，因此执行效率极高。
    

在 Vulkan 中，使用 **Uniform 分支** 的最高效形式是使用 **Specialization Constants (特化常量)**：

- **方法：** 在 Vulkan 中，这些常量在 **Pipeline 创建时**（而非运行时）被赋值。
    
- **着色器端：** 着色器中的 `if` 条件依赖于特化常量。
    
- **编译器优化：** 编译器在编译 Pipeline 时，会根据特化常量的值来决定哪些代码块应该被编译进最终的着色器二进制文件中，从而 **消除未被使用的分支**。这比 Pipeline 变体更灵活，但仍需要管理不同的 Pipeline 变体。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#4-%E4%BD%8E%E6%95%88%E7%9A%84%E5%BE%AA%E7%8E%AF%E5%B1%95%E5%BC%80)4. 低效的循环展开

循环展开是一种程序变换技术，它通过减少循环的迭代次数，但增加循环体内的指令数量，来换取性能提升。着色器中的复杂循环或具有非常量循环次数的循环，可能导致编译器无法高效地进行 **循环展开 (Unroll)**。这使得 GPU 必须执行额外的控制流指令来管理循环，降低了 ALU 的有效指令吞吐量。

**优化方案：**

**1. 使用常量循环次数：**

确保循环次数是常量，或者可以通过着色器常量（Uniform）来确定，从而帮助编译器在编译时彻底展开循环。

**2. 限制循环复杂度：**

优化循环体内部的指令数量和逻辑，提高编译器展开效率，减少控制流开销。

**3. 手动展开循环：**

在Compute Shader中，如果可以通过增加WorkGroupCount，同时调整Shader代码手动减少循环，收益也是非常大的。这也是 GPGPU (通用 GPU 计算) 优化的核心原则之一。

将一个由 **单线程执行循环** 完成的**粗粒度（Coarse-grained）**任务，分解成由 **多个线程并行执行** 的**细粒度（Fine-grained）** 任务务。

|收益点|详细说明|解决的瓶颈|
|:--|:--|:--|
|**最大化并行度**|GPU 的核心数是有限的（例如 Mali-G77 有多个着色器核心）。增加 WorkGroupCount 确保所有核心都能持续满载工作，从而提高 **核心利用率 (Utilization)**。|**GPU 饥饿 (Under-utilization)**|
|**消除循环控制流**|在单线程中，处理 100 个元素的循环需要 100 次指令执行、100 次循环变量递增、100 次比较和 100 次跳转。将其分配给 100 个线程后，这些重复的 **控制流指令** 几乎被完全消除。|**ALU 瓶颈** (减少了不必要的指令)|
|**隐藏延迟**|即使每个线程的工作量很小，大量线程并发运行也能更好地隐藏内存访问延迟。当一个线程因等待数据而被阻塞时，GPU 可以立即切换到另一个线程，确保 ALU 单元保持活跃。|**内存/带宽延迟瓶颈**|

只要您能控制好单个线程的资源占用，这种方法几乎总能带来巨大的性能收益。

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#442-%E5%AF%84%E5%AD%98%E5%99%A8%E5%8E%8B%E5%8A%9B%E4%B8%8E%E5%B9%B6%E8%A1%8C%E5%BA%A6%E9%97%AE%E9%A2%98)4.4.2 寄存器压力与并行度问题

着色器所需的寄存器数量过多，会直接影响 GPU 能够同时运行的线程束（Warp/Wavefront）数量，即 **核心占用率 (Occupancy)** 下降，使得 ALU 虽然活跃，但无法维持高吞吐量。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#1-%E5%AF%84%E5%AD%98%E5%99%A8%E5%8E%8B%E5%8A%9B%E8%BF%87%E5%A4%A7)1. 寄存器压力过大

着色器代码使用了过多的临时变量或复杂的控制流，导致所需的 **工作寄存器数量** 逼近或达到硬件上限（例如 Mali-G1 的 64 个）。当寄存器需求过高时，GPU 硬件被迫限制可以同时运行的线程束数量，从而将 **核心占用率 (Occupancy)** 降低到 50% 或更低。这直接损失了 GPU 的并行计算能力。

**优化方案 (降低寄存器需求)：**

- **优化变量生命周期：** 尽可能缩短着色器中局部变量的生命周期。一旦变量不再需要，编译器就可以释放它所占用的寄存器。
    
- **减少着色器输入/输出：** 减少 Varying 或 Attribute 的数量，因为它们都需要占用寄存器进行处理和传递。
    
- **权衡处理：** 在寄存器减少效果不佳时，如果瓶颈仍是计算而非内存，可以接受较低的占用率，但应确保着色器没有其他瓶颈。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#2-%E5%AF%84%E5%AD%98%E5%99%A8%E6%BA%A2%E5%87%BA-spilling)2. 寄存器溢出 (Spilling)

寄存器压力过高时，编译器无法将所有活动数据存储在快速的寄存器中，被迫将部分数据存储到 **内存 (Spill Region)** 中。每次读写这些溢出到内存的数据时，都会产生额外的 **Load/Store** 指令。

这类问题的**指标特征：** 在离线编译报告中，`Stack use: Spill region` 显示一个非零值；在运行时分析中，`Load/store unit issue` 周期会异常增加，加剧内存带宽瓶颈。

**优化方案：**

- **删除无用参数/变量：** 排查着色器中是否定义了无用的参数和变量。
    
- **降精度优化：** 广泛使用 **FP16/`mediump`** 精度。这是减少寄存器使用的最有效方法之一，因为编译器通常可以用一个 32 位寄存器存储两个 16 位值。
    
- **外部化数据：** 将大型或复杂的查找表、常量数组等数据，存储在 **纹理或 Storage Buffer** 中，通过访问内存而不是寄存器来获取，从而减轻寄存器压力。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#3-uniform-%E5%8F%98%E9%87%8F%E5%86%97%E4%BD%99)3. Uniform 变量冗余

着色器中，Uniform 变量被频繁使用但它们的值很少变化，或者通过 Uniform Buffer 将过多的数据（如大型数组或查找表）传递到着色器。这不仅增加了数据传输的带宽负担，也会增加寄存器的读取和访问开销。

**优化方案：使用 Push Constant**

对于小且频繁变化的 Uniform 数据（例如每个 Draw Call 的世界矩阵、材质 ID 等），使用 **Vulkan Push Constant** 代替传统的 Uniform Buffer。Push Constant 不直接降低着色器的工作寄存器数量，但通过将频繁访问的小数据从慢速的 Uniform Buffer 转移到快速的片上缓冲，显著减轻了 Load/Store Unit 的工作量和等待时间。这间接缓解了由于寄存器溢出（Spilling）而对性能造成的负面影响。

## [](https://km.woa.com/articles/show/644635?kmref=article_recommend#45-gpu%E5%B8%A6%E5%AE%BD%E7%93%B6%E9%A2%88)4.5 GPU带宽瓶颈

带宽瓶颈的核心特征是 **Load/Store Unit** 停顿周期高，**GPU 核心的 ALU 单元处于饥饿状态（等待数据）**。

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#451-%E5%83%8F%E7%B4%A0%E5%86%99%E5%B8%A6%E5%AE%BD%E7%93%B6%E9%A2%88-fill-rate-tile-write-bound)4.5.1 像素写带宽瓶颈 (Fill Rate & Tile Write Bound)

这类瓶颈发生在渲染管线的后端，主要由 GPU 核心的 **ZS/Blend Unit** 负责，涉及到将颜色数据写入帧缓冲区或 Tile Memory 的效率。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#1-%E4%B8%A5%E9%87%8D%E7%9A%84%E8%BF%87%E5%BA%A6%E7%BB%98%E5%88%B6-overdraw)1. 严重的过度绘制 (Overdraw)

同一个像素在屏幕空间上被渲染了多次，导致 **ZS/Blend Unit** 必须执行多次片段着色器，并多次尝试将数据写入帧缓冲。这浪费了大量的 ALU 周期和带宽。

**Mali 指标特征：** `Fragment Warps` 指标极高。

**优化方案 (减少像素工作量)**

- **Early Z / Depth Prepass：** 这是解决 Overdraw 的最有效手段。确保深度测试尽早发生。可以执行一个 **深度预通道 (Depth Prepass)**，提前将所有不透明物体的深度信息写入深度缓冲，后续渲染时即可利用 **Early Z-culling** 快速淘汰被遮挡的像素。
    
- **优化渲染顺序：** 确保不透明物体按 **从前往后（正面到背面）** 的顺序进行渲染。这能最大化 GPU 的 Early Z-culling 效率。
    
- **透明度优化：** 减少使用高复杂度的透明物体或粒子系统。透明物体通常需要关闭深度写入，无法利用 Early Z 优化。
    
- **避免使用discard：** 在 Vulkan 和 Mali 平台上，使用 discard 是直接放弃了 GPU 最核心的两个性能优化机制：Z-culling 和 Tile 内存优化。只要有可能，就应该用 Z-Test 或 Blending/To Coverage 来取代 discard。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#2-%E9%AB%98%E6%98%82%E7%9A%84tile-write)2. 高昂的Tile Write

**渲染目标 (Render Targets)** 的数量过多（如 G-Buffer 中的多个颜色附件），或渲染目标的格式精度过高（例如使用 RGBA32F 格式来存储非 HDR 数据），导致 GPU 每次将 Tile 内存中的数据写回外部 DDR 内存时，数据量巨大。

**Mali 指标特征：** `Tile write bytes/px` 指标异常高（例如远超 RGBA8888 的 4 bytes/px）。

**优化方案 (优化渲染目标)**

- **降低精度：** 将非必须高精度的渲染目标（如 G-Buffer 中的法线、粗糙度贴图、AO 贴图）从 32 位降至 **16 位 (FP16)** 或 **8 位 (Unorm8)** 格式。
    
- **Frame Buffer Fetch (FBF)：** 这是 Mali GPU 的核心优化。在 Vulkan 中，使用 **Subpass Input Attachment** 特性。这允许 GPU 在 Tile 内存内读取前一个 Subpass 的输出结果，**避免将中间数据写回外部 DDR 内存**，从而大幅节省写带宽。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#3-varying-%E6%95%B0%E6%8D%AE%E8%86%A8%E8%83%80)3. Varying 数据膨胀

顶点着色器和片段着色器之间传递了过多的 **Varying 变量**（即插值数据），并且这些变量使用了默认的 **高精度 (32 位)**。这会占用 Tile Memory 的大量空间，并在插值阶段给 **Varying Unit** 带来巨大的带宽压力。

- **Mali 指标特征：** `Varying unit issue` 指标高；`32-bit interpolation active` 指标高。
    

**优化方案：**

- **使用 FP16/`mediump`：** 将所有非必须高精度的 Varying 变量（如 UV 坐标、法线、切线等）声明为 **`mediump`**，以启用 16 位插值。这能有效节省 Tile Memory 空间和带宽。
    
- **移除不必要的 Varying：** 仔细检查着色器，移除任何未在片段着色器中实际使用的 Varying 变量，避免传输和插值冗余数据。
    

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#452-%E5%A4%96%E9%83%A8%E5%86%85%E5%AD%98%E8%AF%BB%E5%B8%A6%E5%AE%BD%E7%93%B6%E9%A2%88-external-read-bound)4.5.2 外部内存读带宽瓶颈 (External Read Bound)

这类问题涉及从慢速的外部 DDR 内存读取纹理、模型或常量数据时，主要涉及 **Load/Store Unit** 和 **Texture Unit** 的工作效率。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#1-%E7%BA%B9%E7%90%86%E6%95%B0%E6%8D%AE%E9%87%8F%E5%BA%9E%E5%A4%A7)1. 纹理数据量庞大

场景中使用了大量的未压缩、高分辨率纹理（例如 4K 贴图、3D 纹理）。GPU 在采样时必须从外部内存读取巨量数据，导致纹理单元带宽受限。

**Mali 指标特征：** `External read bytes/cy (Texture)`（外部读取字节/周期）指标高；`Texture CPI`（每指令周期数）指标高。

**优化方案 (纹理优化与 LOD)**

- **压缩：** 对所有非必需无损的纹理使用 **硬件压缩格式**（例如 ASTC、BC 格式），这能将纹理的内存占用和读取带宽需求降低 4 倍到 8 倍。
    
- **LOD (细节分级)：** 确保所有纹理都正确生成并使用了 **Mipmaps**。对于远距离物体，GPU 自动使用较低的 Mipmap 层级，从而减少所需读取的数据量。
    
- **Culling (剔除)：** 利用 **视锥体剔除** 或 **遮挡查询** 在 CPU 或 GPU 前端剔除完全不可见的物体，避免加载其所有纹理数据。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#2-%E5%86%85%E5%AD%98%E8%AE%BF%E9%97%AE%E5%B1%80%E9%83%A8%E6%80%A7%E5%B7%AE)2. 内存访问局部性差

GPU 的内存访问模式缺乏空间或时间局部性，导致 L1/L2 缓存频繁不命中。每次缓存不命中都需要从慢速的 DDR 内存中加载数据，迫使 GPU 核心等待。这尤其影响顶点数据和 Storage Buffer 的访问。

**Mali 指标特征：** `Load/store unit issue` 指标高；`Vertex Memory Efficiency`（顶点内存效率）指标低。

**优化方案 (优化数据布局)**

- **顶点缓存优化：** 重新排序网格的索引数据。使用专门的工具，确保在处理一个顶点时，其相邻的顶点数据仍在 GPU 的顶点缓存中，提高 `Vertex Memory Efficiency`。
    
- **对齐访问：** 确保 Uniform Buffer 或 Storage Buffer 中的数据访问是 **对齐和连续的**。避免使用非对齐的结构体，以提高 Load/Store Unit 的缓存读取效率。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#3-%E5%AF%84%E5%AD%98%E5%99%A8%E6%BA%A2%E5%87%BA-spilling)3. 寄存器溢出 (Spilling)

虽然这源于 **计算瓶颈**（着色器过于复杂导致寄存器需求过高），但它直接转化为带宽问题。当寄存器压力过大时，数据被编译器溢出到外部内存，导致 **Load/Store Unit** 必须频繁地读写这些溢出数据。

**Mali 指标特征：** `Load/store unit issue` 指标极高；离线编译器报告中 `Stack use: Spill region` 非零。

**优化方案 (解决计算瓶颈)**

- **降精度：** **广泛使用 FP16/`mediump`**。这是解决寄存器溢出的最有效方法，因为它能大幅减少单个线程的寄存器使用量。
    
- **代码简化：** 简化着色器代码，减少临时变量和复杂的控制流，从根本上降低寄存器压力。
    

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#4-%E9%80%9A%E8%BF%87culling-%E5%89%94%E9%99%A4-%E5%87%8F%E5%B0%91%E8%AF%BB%E5%B8%A6%E5%AE%BD)4. 通过Culling (剔除) 减少读带宽

**视锥体剔除/遮挡剔除/背面剔除**，确保了只有可见的物体才会提交 Draw Call。这直接避免了 GPU 浪费读带宽去加载不可见物体的顶点、索引和 Uniform 数据，从而减少了 **`External read bytes/cy`** 的总量。

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#453-%E5%B8%A6%E5%AE%BD%E4%BC%98%E5%8C%96%E6%80%BB%E7%BB%93)4.5.3 带宽优化总结

带宽瓶颈的优化策略是 **“少读，少写，高效利用缓存”**。通过优化数据格式、减少冗余绘制和提高内存访问局部性，可以将瓶颈从 DDR 内存转移到 GPU 核心的 ALU 单元。

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#454-%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)4.5.4 最佳实践

Android官网的 [管理顶点数据](https://developer.android.com/games/optimize/vertex-data-management) 一文，讲解了如何通过 **顶点数据压缩** 和 **顶点流拆分** 的方式来降低GPU带宽，非常有参考价值。

**1. 顶点数据压缩 (Vertex Data Compression)**

核心思想：通过减少单个顶点所需的存储空间，从而减少内存读取量。

**2. 顶点流拆分 (Vertex Stream Splitting)**

核心思想：将顶点属性分离到不同的缓存流中，以提高缓存局部性。

|概念|核心目标|优化方法|效果|
|:--|:--|:--|:--|
|**顶点流拆分**|将顶点数据（如位置、法线、UV）分离到多个独立的 **Vulkan Buffer** 中。|**分离属性：** 将**核心属性**（如位置、LOD 变化属性）放入一个流；将**次要属性**（如切线、骨骼权重、颜色）放入另一个流。|**提高缓存命中率：** GPU 在执行 Early Z Prepass 或 Shadow Pass 时，**只需要读取包含位置数据的流**，避免加载不必要的法线和 UV 数据，减少 **Load/Store Unit Issue**。|
|**数据访问**|GPU 按需访问所需的流。|**LOD/剔除优化：** 只需要读取最小数据集来完成剔除或渲染最简单的 LOD 级别。|**带宽节省：** 在某些渲染通道中，可以避免整个加载大块数据。|

**简而言之：**

- **顶点数据压缩** 告诉 GPU **少读点** 数据（数据更小）。
    
- **顶点流拆分** 告诉 GPU **只读需要的** 数据（数据更局部化）。
    

两者结合可以有效解决 GPU 性能分析中常见的 **`Vertex Memory Efficiency` 低** 和 **`External read bytes/cy` 高** 的问题。

## [](https://km.woa.com/articles/show/644635?kmref=article_recommend#46-%E5%85%B6%E4%BB%96%E4%BC%98%E5%8C%96%E6%89%8B%E6%AE%B5)4.6 其他优化手段

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#461-compute-shader-skinning)4.6.1 Compute Shader Skinning

**Compute Shader Skinning** 是指将 **蒙皮计算 (Skinning)** 提前到 **Compute Shader** 中执行，是现代高性能 Vulkan 程序，尤其是涉及复杂角色动画和大量角色的场景中，公认的、最高效的优化手段之一。

这个策略的本质是 **将 GPU 渲染瓶颈转化为可并行化的 Compute 瓶颈**。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#1-%E6%8F%90%E9%AB%98-gpu-%E6%A0%B8%E5%BF%83%E5%88%A9%E7%94%A8%E7%8E%87)1. 提高 GPU 核心利用率

**传统 Vertex Shader Skinning** 蒙皮计算在 **Vertex Shader** 中执行。Vertex Shader 的执行是串行依赖于 Draw Call 的顶点流的，效率受限于几何体的处理速度和流水线。

**Compute Shader 模式** 蒙皮计算在 Compute Shader 中，可以利用 GPU 的所有 Compute 核心进行大规模并行。并且可以利用**异步计算 (Asynchronous Compute)** 机制并发执行Skinning任务和渲染任务，实现多帧并行。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#2-%E4%BC%98%E5%8C%96%E6%B8%B2%E6%9F%93%E6%B5%81%E6%B0%B4%E7%BA%BF-geometry-processing)2. 优化渲染流水线 (Geometry Processing)

**传统模式** 顶点着色器既要负责蒙皮计算（大量矩阵乘法），又要负责标准几何体变换。这增加了 **顶点着色器** 的 **ALU 负载** 和 **寄存器压力**。

**Compute Shader 模式** 首先计算蒙皮后的新顶点数据，存储在 **Storage Buffer** 中，渲染 Pass 中的 **Vertex Shader** 变得非常简单，它只需要读取这个预先计算好的 Storage Buffer，执行标准的投影变换即可。

**优势：** 大幅降低了 **渲染 Pass** 中 Vertex Shader 的复杂度，从而降低了 **几何体瓶颈**，并释放了 ALU 资源，用于更重要的片段着色。

总而言之，**Compute Shader Skinning** 将一个串行化、高开销的 Vertex Shader 任务，转化为了一个高度可控、可并行化的 GPU 计算任务，从而提高了整个渲染管线的吞吐量。

### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#462-%E5%BC%82%E6%AD%A5%E8%AE%A1%E7%AE%97-ac)4.6.2 异步计算 (AC)

异步计算 (Asynchronous Compute) 是一种充分利用现代 GPU 硬件并行性的技术，它允许 GPU **同时执行图形渲染和通用计算任务**，从而提高硬件利用率和整体吞吐量。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#1-%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%86%EF%BC%9A%E5%A4%9A%E9%98%9F%E5%88%97)1. 核心原理：多队列

Vulkan 允许应用程序查询 GPU 硬件支持的 **队列族 (Queue Families)**，通常包括：

|队列类型|作用|任务示例|
|:--|:--|:--|
|**Graphics Queue (图形队列)**|处理所有渲染相关的命令。|`vkCmdDraw*`, `vkCmdBeginRenderPass`|
|**Compute Queue (计算队列)**|处理通用计算任务，与图形渲染解耦。|`vkCmdDispatch` (例如 Skinning、粒子物理计算)|
|**Transfer Queue (传输队列)**|处理纯粹的数据传输命令（CPU → GPU 或 GPU 内部）。|`vkCmdCopyBuffer`, `vkCmdCopyImage`|

**异步计算** 的实现就是通过 **同时向 Graphics Queue 和 Compute Queue 提交不相关的任务**，让它们在 GPU 的不同执行单元上并行运行。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#2-%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%88%B6%EF%BC%9A%E6%98%BE%E5%BC%8F%E5%90%8C%E6%AD%A5)2. 实现机制：显式同步

由于任务是并行执行的，如果一个任务的结果是另一个任务的输入（例如，Compute Pass 计算蒙皮后的顶点，Render Pass 需要渲染这些顶点），就需要显式同步。

#### [](https://km.woa.com/articles/show/644635?kmref=article_recommend#3-%E5%BC%82%E6%AD%A5%E8%AE%A1%E7%AE%97%E7%9A%84%E4%BC%98%E5%8A%BF)3. 异步计算的优势

- **提高硬件利用率：** 避免 GPU 核心闲置。例如，当 Graphics Queue 在等待渲染前端（如 ZS/Blend Unit）时，Compute Queue 可以利用空闲的 ALU 单元来执行 Skinning 或后处理计算。
    
- **隐藏延迟：** 将计算密集型任务（如蒙皮）从渲染流水线中分离，使其与图形渲染并行运行，从而隐藏了计算的延迟。
    

异步计算可以在 Vulkan 中实现 **最大化 GPU 吞吐量**，并更进一步的在 GPU 侧实现 **多帧并行渲染**。