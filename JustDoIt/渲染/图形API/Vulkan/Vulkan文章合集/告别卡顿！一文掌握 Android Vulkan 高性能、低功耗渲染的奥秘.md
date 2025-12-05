
# 1 概述


在移动设备上实现高性能、低功耗的图形渲染，是现代游戏和应用开发面临的核心挑战。Vulkan 作为新一代图形 API，为 Android 开发者提供了对 GPU 硬件前所未有的底层控制权，从而解锁了巨大的性能潜力。然而，这种控制权也带来了更高的复杂性和调优难度。

本文旨在为 Vulkan 开发者提供一套系统化的性能优化方法论。我们将首先解析移动端 GPU（如 分块渲染器）的工作原理，奠定优化基础。随后，我们将详细指导如何利用专业的 性能分析工具 和 关键 GPU 指标 科学地定位瓶颈。

文章的核心将围绕三大性能瓶颈展开深入讨论：CPU 瓶颈、GPU 计算瓶颈和内存带宽瓶颈。我们将为每一种限制提供具体的优化思路和最佳实践。此外，本文还将涵盖对 内存资源的高效管理 以及在移动环境中至关重要的 功耗优化策略。

通过掌握这些知识，我们将能够精准识别性能瓶颈的主导因素，并运用 Vulkan 的强大能力，在 Android 平台上构建出性能卓越、延迟更低、电池续航更持久的下一代图形应用。


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

线程束被分配给一个或多个 **Shader Core** 执行。在 Shader Core 内部，所有线程束中的线程都会**同时执行相同的指令**（即 SIMT）。**一个 Shader Core 在同一时间（硬件时钟周期 / 指令周期层面）只能处理一个线程束（warp）**。

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
    
- **数量：** 由输入数据决定。例如，如果有  个顶点，就有  个顶点着色线程。


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

**Mali Pixels：** 比如现在测试机 GPU 是 850MHZ 共 3 核心，分辨率是 2280X1080，如果目标帧率是60的话，我们就能算出每秒能处理的最多像素数 (850MHZ X3)/(2280 X 1080 X 60) = 17。也就是说或是这台手机要想流畅运行60帧的话，留给每个像素的 GPU 时钟周期必须小于17，当然这还包括前面处理顶点的时间周期。上图中我们的采样周期是1秒，也就是说1秒产生了153M的像素。

**Mali Overdraw:** 1秒产生了153M个像素，理想情况下每个像素只绘制一次。但实时上是不太可能的，比如Alpha blend会在同一个像素上Blend多次，这就是我们说的OverDraw（实际Fragment的数量除以最终产生的像素数），上图中的值是2.98threads，大概每个像素被过度绘制2.98倍。

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

那么 Mali Core Warps 和 Mail Core Throughout这两个指标有什么区别呢？简单来说：

- Mali Core Warps 衡量的是 工作总量 (Volume)。
- Mali Core Throughput 衡量的是 工作效率 (Efficiency)。

如果Fragment Warps高，说明像素处理工作量大，可能是 Overdraw 或 高分辨率。 如果Non-fragment Warps高，说明几何体处理工作量大，可能是 顶点数过多 或 Draw Call 过多。

如果Cycles/fragment thread 高，说明片段着色器过于复杂，需要大量 ALU 周期。 如果Cycles/non-fragment thread高，说明顶点着色器过于复杂，需要大量 ALU 周期。

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