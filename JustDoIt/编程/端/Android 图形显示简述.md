
`Android` 图形 `Graphic` 和显示 `Display` 是两个独立的部分，这里放在一起简述；介绍了图像和显示相关的基本概念，比如 `BufferQueue` 生产者消费者模型， `Surface/SurfaceFlinger` 图形合成等等。

## 概念

应用开发者可通过两种方式将图像绘制到屏幕上：使用 `Canvas` 或 `OpenGL` ：

- `android.graphics.Canvas` 是一个 `2D` 图形 `API` ， `Canvas API` 通过一个名为 `OpenGLRenderer` 的绘制库实现硬件加速，该绘制库将 `Canvas` 运算转换为 `OpenGL` 运算，以便它们可以在 `GPU` 上执行。从 `Android 4.0` 开始，硬件加速的 `Canvas` 默认情况下处于启用状态
- 除了 `Canvas`，开发者渲染图形的另一个主要方式是使用 `OpenGL ES` 直接渲染到 `Surface` 。 `Android` 在 `Android.opengl` 软件包中提供了 `OpenGL ES` 接口

### EGL

先熟悉 `Android` 平台图形处理 `API` 的标准：

- `OpenGL`  
    是由 `SGI` 公司开发的一套 `3D` 图形软件接口标准，由于具有体系结构简单合理、使用方便、与操作平台无关等优点， `OpenGL` 迅速成为 `3D` 图形接口的工业标准，并陆续在各种平台上得以实现。
- `OpenGL ES`  
    是由 `khronos` 组织根据手持及移动平台的特点，对 `OpenGL 3D` 图形 `API` 标准进行裁剪定制而形成的。
- `Vulkan`  
    是由 `khronos` 组织在 2016 年正式发布的，是 `OpenGL ES` 的继任者。 `API` 是轻量级、更贴近底层硬件 `close-to-the-metal` 的接口，可使 `GPU` 驱动软件运用多核与多线程 `CPU` 性能。

`OpenGL ES` 定义了一个渲染图形的 `API` ，但没有定义窗口系统。为了让它能够适合各种平台，它将与知道如何通过操作系统创建和访问窗口的库结合使用。而在 `Android` 中，这个库被称为 `EGL` ；也就是说 `EGL` 主要是适配系统和关联窗口属性。如果要绘制纹理多边形，应使用 `OpenGL ES` 调用；如果要在屏幕上进行渲染，应使用 `EGL` 调用。  
`OpenGL ES` 是 `Android` 绘图 `API` ，但 `OpenGL ES` 是平台通用的，在特定设备上使用需要一个中间层做适配， `Android` 中这个中间层就是 `EGL` 。

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-android-egl.jpg)

### `Surface` & `SurfaceFlinger`

无论开发者使用什么渲染 `API`，一切内容都会渲染到 `Surface` 。 `Surface` 表示缓冲队列中的生产者，而缓冲队列通常会被 `SurfaceFlinger` 消耗。在 `Android` 平台上创建的每个窗口都由 `Surface` 提供支持。所有被渲染的可见 `Surface` 都被 `SurfaceFlinger` 合成到显示部分。它们遵循生产者 / 消费者模型：

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-ape_fwk_graphics.png)

- 图像流生产者  
    图像流生产者可以是生成图形缓冲区以供消耗的任何内容。例如 `OpenGL ES, Canvas 2D, mediaserver` 视频解码器。
- 图像流消费者  
    图像流的最常见消费者是 `SurfaceFlinger` ，该系统服务会消耗当前可见的 `Surface` ，并使用窗口管理器中提供的信息将它们合成到显示部分。 `SurfaceFlinger` 是可以修改所显示部分内容的唯一服务。 `SurfaceFlinger` 使用 `OpenGL` 和 `Hardware Composer` 来合成一组 `Surface` 。  
    其他 `OpenGL ES` 应用也可以消耗图像流，例如相机应用会消耗相机预览图像流。非 `GL` 应用也可以是消费者，例如 `ImageReader` 类。

### `WMS`

窗口管理器（ WindowManagerServices），控制窗口的 `Android` 系统服务，它是视图容器。窗口总是由 `Surface` 提供支持。该服务会监督生命周期、输入和聚焦事件、屏幕方向、转换、动画、位置、变形、 `Z-Order` 以及窗口的其他许多方面。窗口管理器会将所有窗口元数据发送到 `SurfaceFlinger` ，以便 `SurfaceFlinger` 可以使用该数据在显示部分合成 `Surface` 。

### `FrameBuffer`

`FrameBuffer` 帧缓冲驱动，它是 `Linux` 的一种驱动程序接口。 `Linux` 是工作在保护模式下，所以用户态进程是无法像 `DOS` 那样使用显卡 `BIOS` 里提供的中断调用来实现直接写屏， `Linux` 抽象出 `FrameBuffer` 这个设备来供用户态进程实现直接写屏。 `FrameBuffer` 机制模仿显卡的功能，将显卡硬件结构抽象掉，可以通过 `FrameBuffer` 的读写直接对显存进行操作。用户可以将 `FrameBuffer` 看成是显示内存的一个映像，将其映射到进程地址空间之后，就可以直接进行读写操作，而写操作可以立即反应在屏幕上。这种操作是抽象的统一的。用户不必关心物理显存的位置、换页机制等等具体细节，这些都是由 `FrameBuffer` 设备驱动来完成的。但 `FrameBuffer` 本身不具备任何运算数据的能力，就只好比是一个暂时存放水的水池。 `CPU` 将运算后的结果放到这个水池, 水池再将结果流到显示器，中间不会对数据做处理。应用程序也可以直接读写这个水池的内容在这种机制下，尽管 `FrameBuffer` 需要真正的显卡驱动的支持，但所有显示任务都有 `CPU` 完成，因此 `CPU` 负担很重。

在开发者看来， `FrameBuffer` 本质上是一块显示缓存，往显示缓存中写入特定格式的数据就意味着向屏幕输出内容。所以说 `FrameBuffer` 就是一块白板。例如对于初始化为 16 位色的 `FrameBuffer` 来说， `FrameBuffer` 中的两个字节代表屏幕上一个点，从上到下，从左至右，屏幕位置与内存地址是顺序的线性关系。  
帧缓存可以在系统存储器 (内存) 的任意位置，视频控制器通过访问帧缓存来刷新屏幕。帧缓存也叫刷新缓存 `FrameBuffer` 或 `RefreshBuffer` ，这里的帧 `Frame` 是指整个屏幕范围。帧缓存有个地址，是在内存里。我们通过不停的向 `FrameBuffer` 中写入数据，显示控制器就自动的从 `FrameBuffer` 中取数据并显示出来。全部的图形都共享内存中同一个帧缓存。

`FrameBuffer` 帧缓冲实际上包括两个不同的方面：

- `Frame` ：帧，就是指一幅图像，在屏幕上看到的那幅图像就是一帧
- `Buffer` ：缓冲，就是一段存储区域，可这个区域存储的是帧

`FrameBuffer` 就是一个存储图形 / 图像帧数据的缓冲。`Linux` 内核提供了统一的 `Framebuffer` 显示驱动，设备节点 `/dev/graphics/fb*` 或者 `/dev/fb*` ，以 `fb0` 表示第一个 `Monitor` ，当前实现中只用到了一个显示屏。这个虚拟设备将不同硬件厂商实现的真实设备统一在一个框架下，这样应用层就可以通过标准的接口进行图形 / 图像的输入和输出了：

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-linux-framebuffer.png)

从上图中可以看出，应用层通过标准的 `ioctl, mmap` 等系统调用，就可以操作显示设备，用起来非常方便。这里 `mmap` 把设备中的显存映射到用户空间的，在这块缓冲上写数据，就相当于在屏幕上绘画。

### `Gralloc`

`Gralloc` 的含义为是 `Graphics Alloc` 图形分配 。 `Android` 系统在硬件抽象层中提供了一个 `Gralloc` 模块，封装了对 `Framebuffer` 的所有访问操作。

`Gralloc` 模块符合 `Android` 标准的 `HAL` 架构设计；它分为 `fb` 和 `gralloc` 两个设备：前者负责打开内核中的 `Framebuffer` 、初始化配置，以及提供 `post, setSwapInterval` 等操作；后者则管理帧缓冲区的分配和释放。上层只能通过 `Gralloc` 访问帧缓冲区，这样一来就实现了有序的封装保护。

`Gralloc` 图形内存分配器，分配图像生产者请求的内存。它不仅仅是在原生堆上分配内存的另一种方法；在某些情况下，分配的内存可能并非缓存一致，或者可能完全无法从用户空间访问。分配的性质由用法标记确定，这些标记包括以下属性：

- 从软件 `CPU` 访问内存的频率
- 从硬件 `GPU` 访问内存的频率
- 是否将内存用作 `OpenGL ES: GLES` 纹理
- 视频编码器是否会使用内存

例如如果格式指定为 `RGBA 8888` 像素，并且指明将从软件访问缓冲区（这意味着应用将直接触摸像素），则分配器必须按照 `R-G-B-A` 的顺序为每个像素创建 4 个字节的缓冲区。相反如果指明仅从硬件访问缓冲区且缓冲区作为 `GLES` 纹理，则分配器可以执行 `GLES` 驱动程序所需的任何操作 - `BGRA` 排序、非线性搅和布局、替代颜色格式等。允许硬件使用其首选格式可以提高性能。某些值在特定平台上无法组合，例如视频编码器标记可能需要 `YUV` 像素，因此将无法添加软件访问权并指定 `RGBA 8888` 。

`Gralloc` 分配器返回的句柄可以通过 `Binder` 在进程之间传递。

### `HWC`

`HWC: Hardware Composer` 硬件混合渲染器，显示子系统的硬件抽象实现。 `SurfaceFlinger` 可以将某些合成工作委托给 `Hardware Composer`，以分担 `OpenGL` 和 `GPU` 上的工作量。 `SurfaceFlinger` 只是充当另一个 `OpenGL ES` 客户端。因此在 `SurfaceFlinger` 将一个或两个缓冲区合成到第三个缓冲区中的过程中，它会使用 `OpenGL ES` 。这样使合成的功耗比通过 `GPU` 执行所有计算更低。
`Hardware Composer HAL` 则进行另一半的工作，并且是所有 `Android` 图形渲染的核心。 `Hardware Composer` 必须支持事件，其中之一是 `VSYNC`（另一个是支持即插即用 `HDMI` 的热插拔 `hotplug` ）。

### VSYNC 垂直刷新

先介绍几个概念：

显卡处理图像的帧速率和屏幕刷新频率是相互独立的，当两者不一致时会出现 `tearing` 问题，为了解决不一致的问题，引入了 `Vsync` 信号：当整个屏幕刷新完毕，即一个垂直刷新周期完成，会有短暂的空白期，等待定期同步信号 `VSync` 信号，收到后才开始下一次屏幕刷新；所以 `VSync` 中的 `V` 指的是垂直刷新中的垂直 `Vertical` 。  
`Vsync` 技术意味着，显卡显示性能极限被限制在屏幕刷新率以内了：在系统显卡处理的 `FPS` 高于屏幕刷新率时，显卡会将一部分时间浪费在等待上；因为没有可用的内存用于绘制，显卡需要等待 `Vsync` 信号才能绘制下一帧。

- 单缓存缓存模型  
    理想的情况是帧率和刷新频率相等，每绘制一帧，屏幕显示一帧，如下图所示；但是如果不一致，就会出现 `tearing` 。
    
    ![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-vsync-single-buffer.png)
    
- 双重缓存 `Double Buffer`  
    两个缓存区分别为 `Back Buffer` 和 `Frame Buffer` 。 `GPU` 向 `Back Buffer` 中写数据，屏幕从 `Frame Buffer` 中读数据。当屏幕刷新完成后产生 `VSync` 信号，此时将数据从 `Back Buffer` 复制到 `Frame Buffer`，可认为该复制操作在瞬间完成；复制完后显示设备开始显示这帧数据，同时通知 `CPU/GPU` 绘制下一帧图像。
    
    ![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-vsync-double-buffer.png)
    
      
    但是当 `GPU/CPU` 绘制一帧的时间超过了 `Vsync` 时，屏幕刷新从 `Frame Buffer` 取到的数据仍然是上一帧数据，即两个 `Vsync` 周期显示同一帧数据，我们称为发生了掉帧 `Dropped Frame, Skipped Frame, Jank` 现象。  
    
    ![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-vsync-double-buffer-jank.png)
    
- 三重缓存 `Triple Buffer`  
    在双重缓存模型中，当 `Jank` 现象出现时， `GPU/CPU` 此时都处于闲置状态，所以引入了三重缓存的概念：在 `Jank` 时， `GPU/CPU` 在第三个 `Buffer` 中绘制数据：  
    
    ![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-vsync-triple-buffer.png)
    
      
    需要注意的是，第三个缓存并不是总是存在的，只要当需要的时候才会创建；而且也无法完全解决 `Jank` 现象，但是能缓解。  
    
    ![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-vsync-triple-buffer-jank.png)
    

### 60Hz 和 16 ms

从上面解释帧速率时提到，虽然人眼感知生理的极限 `85fps` ，但达到 `60fps` 时动画就已经有很好的体验，不会出现卡顿和迟滞现象；而最为关键的是 `60Hz` 是美国交流电的频率，如果屏幕刷新频率能够匹配交流电的频率就可以有效的预防屏幕中出现滚动条；所以：

- `60Hz` 的屏幕刷新率或者 `60fps` 的帧率，是人眼能够感知到比较流畅的数值
- `1000ms/60=16ms` ， `16ms` 是指 `GPU/CPU` 在绘制图形时，必须在这个刷新频率内绘制完成，否则会出现丢帧现象

### `BufferQueue`

实现了整个生产者消费者模型。 `BufferQueues` 是 `Android` 图形组件之间的粘合剂。它们是一对队列，可以调解缓冲区从生产者到消费者的固定周期。一旦生产者移交其缓冲区， `SurfaceFlinger` 便会负责将所有内容合成到显示部分。  
`BufferQueue` 永远不会复制缓冲区内容（移动如此多的数据是非常低效的操作）；相反**缓冲区始终通过句柄进行传递**。

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-bufferqueue.png)

`BufferQueue` 包含将图像流生产者与图像流消费者结合在一起的逻辑。图像生产者的一些示例包括由相机 `HAL` 或 `OpenGL ES` 游戏生成的相机预览。图像消费者的一些示例包括 `SurfaceFlinger` 或显示 `OpenGL ES` 流的另一个应用，如显示相机取景器的相机应用。  
`BufferQueue` 是将缓冲区池与队列相结合的数据结构，它使用 `Binder IPC` 在进程之间传递缓冲区。生产者接口，或者您传递给想要生成图形缓冲区的某个人的内容，即是 `IGraphicBufferProducer` （ `SurfaceTexture` 的一部分）。 `BufferQueue` 通常用于渲染到 `Surface` ，并且与 `GL` 消费者及其他任务一起消耗内容。 `BufferQueue` 可以在三种不同的模式下运行：

- 类同步模式  
    默认情况下， `BufferQueue` 在类同步模式下运行，在该模式下，从生产者进入的每个缓冲区都在消费者那退出。在此模式下不会舍弃任何缓冲区。如果生产者速度太快，创建缓冲区的速度比消耗缓冲区的速度更快，它将阻塞并等待可用的缓冲区。
- 非阻塞模式  
    `BufferQueue` 还可以在非阻塞模式下运行，在此类情况下，它会生成错误，而不是等待缓冲区。在此模式下也不会舍弃缓冲区。这有助于避免可能不了解图形框架的复杂依赖项的应用软件出现潜在死锁现象。
- 舍弃模式  
    `BufferQueue` 可以配置为丢弃旧缓冲区，而不是生成错误或进行等待。例如，如果对纹理视图执行 `GL` 渲染并尽快绘制，则必须丢弃缓冲区。

为了执行这项工作的大部分环节， `SurfaceFlinger` 就像另一个 `OpenGL ES` 客户端一样工作。例如当 `SurfaceFlinger` 正在积极地将一个缓冲区或两个缓冲区合成到第三个缓冲区中时，它使用的是 `OpenGL ES` 。

### 数据流

`Android` 图形管道数据流如下图所示：

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/01113-android-graphics-display-graphics_pipeline.png)

左侧的对象是生成图形缓冲区的渲染器，如主屏幕、状态栏和系统界面。 `SurfaceFlinger` 是合成器，而硬件混合渲染器是制作器。

### 组件小结

- 低级别组件
    - `BufferQueue` 和 `gralloc` 。 `BufferQueue` 将可生成图形数据缓冲区的组件（生产者）连接到接受数据以便进行显示或进一步处理的组件（消费者）。通过供应商专用 `HAL` 接口实现的 `gralloc` 内存分配器将用于执行缓冲区分配任务。
    - `SurfaceFlinger, Hardware Composer` 和虚拟显示屏。 `SurfaceFlinger` 接受来自多个源的数据缓冲区，然后将它们进行合成并发送到显示屏。 `Hardware Composer HAL (HWC)` 确定使用可用硬件合成缓冲区的最有效的方法，虚拟显示屏使合成输出可在系统内使用（录制屏幕或通过网络发送屏幕）。
    - `Surface, Canvas, SurfaceHolder` 。 `Surface` 可生成一个通常由 `SurfaceFlinger` 使用的缓冲区队列。当渲染到 `Surface` 上时，结果最终将出现在传送给消费者的缓冲区中。 `Canvas API` 提供一种软件实现方法（支持硬件加速），用于直接在 `Surface` 上绘图（ `OpenGL ES` 的低级别替代方案）。与视图有关的任何内容均涉及到 `SurfaceHolder` ，其 `API` 可用于获取和设置 `Surface` 参数（如大小和格式）。
    - `EGLSurface, OpenGL ES` 。 `OpenGL ES (GLES)` 定义了用于与 `EGL` 结合使用的图形渲染 `API` 。 `EGI` 是一个规定如何通过操作系统创建和访问窗口的库（要绘制纹理多边形，请使用 `GLES` 调用；要将渲染放到屏幕上，请使用 `EGL` 调用）。 `ANativeWindow` ，它是 `Java Surface` 类的 `C/C++` 等价类，用于通过原生代码创建 `EGL` 窗口 `Surface` 。
    - `Vulkan` 。 `Vulkan` 是一种用于高性能 `3D` 图形的低开销、跨平台 `API` 。与 `OpenGL ES` 一样， `Vulkan` 提供用于在应用中创建高质量实时图形的工具。 `Vulkan` 的优势包括降低 `CPU` 开销以及支持 `SPIR-V` 二进制中间语言。
- 高级别组件
    - `SurfaceView` 和 `GLSurfaceView` 。 `SurfaceView` 结合了 `Surface` 和 `View` 。 `SurfaceView` 的 `View` 组件由 `SurfaceFlinger` （而不是应用）合成，从而可以通过单独的线程 / 进程渲染，并与应用界面渲染隔离。 `GLSurfaceView` 提供帮助程序类来管理 `EGL` 上下文、线程间通信以及与 `Activity` 生命周期的交互（但使用 `GLES` 时并不需要 `GLSurfaceView` ）。
    - `SurfaceTexture` 。 `SurfaceTexture` 将 `Surface` 和 `GLES` 纹理相结合来创建 `BufferQueue` ，而应用是 `BufferQueue` 的消费者。当生产者将新的缓冲区排入队列时，它会通知应用。应用会依次释放先前占有的缓冲区，从队列中获取新缓冲区并执行 `EGL` 调用，从而使 `GLES` 可将此缓冲区作为外部纹理使用。 `Android 7.0` 增加了对安全纹理视频播放的支持，以便用户能够对受保护的视频内容进行 `GPU` 后处理。
    - `TextureView` 。 `TextureView` 结合了 `View` 和 `SurfaceTexture` 。 `TextureView` 对 `SurfaceTexture` 进行包装，并负责响应回调以及获取新的缓冲区。在绘图时， `TextureView` 使用最近收到的缓冲区的内容作为其数据源，根据 `View` 状态指示，在它应该渲染的任何位置和以它应该采用的任何渲染方式进行渲染。 `View` 合成始终通过 `GLES` 来执行，这意味着内容更新可能会导致其他 `View` 元素重绘。

高级别组件可以直接在 `APP` 中使用。

## `Buffer/Window` 体系

### 代码速查表

system/core/libcutils/include/cutils/native_handle.h
hardware/qcom/display/libgralloc/gralloc_priv.h
frameworks/native/libs/nativebase/include/nativebase/nativebase.h
frameworks/native/libs/nativewindow/include/system/window.h
frameworks/native/libs/ui/include/ui/ANativeObjectBase.h
frameworks/native/libs/ui/include/ui/GraphicBuffer.h
frameworks/native/libs/gui/include/gui/Surface.h

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#native-handle-buffer-handle-t "native_handle/buffer_handle_t")`native_handle/buffer_handle_t`

先看 `native_handle` 这个结构体的定义：

// native_handle.h
typedef struct native_handle
{
    int version;        /* sizeof(native_handle_t) */
    int numFds;         /* number of file-descriptors at &data[0] */
    int numInts;        /* number of ints at &data[numFds] */
#if defined(__clang__)
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wzero-length-array"
#endif
    int data[0];        /* numFds + numInts ints */
#if defined(__clang__)
#pragma clang diagnostic pop
#endif
} native_handle_t;

typedef const native_handle_t* buffer_handle_t;

`native_handle` 这个结构体，描述了一个数据结构，其中最关键的是 `data[0]` ，它是一个长度为 0 的数组，即 `native_handle` 是一个柔性数组。在标准的 `C/C++` 中, 长度为 0 的数组是不被允许的，编译时会产生错误！长度为 0 的数组是 `C/C++` 的扩展，需要当前编译器支持这个扩展。从头文件注释中也可以看出，当使用的是 `clang` 编译器时，才会定义 `data[0]` 并且忽略数组为 0 的警告。

从 `C/C++` 中柔性数组的用途来看， `native_handle` 表示的是一个不定长数据结构，实际意义指向连续分配的内存空间（除了 `native_handle` 之外）代表的数据结构（通常是 `private_handle_t` ）。这里这么做，是因为显示系统和每家实现平台相关度很高， `native_handle` 定义一个通用的数据结构，至于显示系统如何显示，每家自己去实现对应的 `private_handle_t` 。

`native_handle` 结构体中的注释写的很清楚， `numFds` 表示被指向数据结构包含几个文件描述符； `numInts` 表示被指向数据结构长度是多少个整型；有了这两个信息后，内存分配就很容易了，参考 `native_handle_create` 的源码实现：

// native_handle.c
native_handle_t* native_handle_create(int numFds, int numInts) {
    if (numFds < 0 || numInts < 0 || numFds > kMaxNativeFds 
            || numInts > kMaxNativeInts) {
        errno = EINVAL;
        return NULL;
    }

    // 给 native_handle 分配连续的内存空间
    // 除了 native_handle 自身所占空间，还包含被指向数据结构的长度  
    size_t mallocSize = sizeof(native_handle_t) 
        + (sizeof(int) * (numFds + numInts));
    native_handle_t* h = malloc(mallocSize);
    if (h) {
        h->version = sizeof(native_handle_t);
        h->numFds = numFds;
        h->numInts = numInts;
    }
    return h;
}

小结： `native_handle, native_handle_t` 表示一个不定长数据结构，而 `buffer_handle_t` 表示指向 `native_handle` 的指针。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#private-handle-t "private_handle_t")`private_handle_t`

`private_handle_t` 描述的是一块缓存，因为和实现平台高度相关，我这里选取高通平台，先看头文件定义：

// gralloc_priv.h
#ifdef __cplusplus
struct private_handle_t : public native_handle {
#else
    struct private_handle_t {
        native_handle_t nativeHandle;
#endif
        enum {
            PRIV_FLAGS_FRAMEBUFFER        = 0x00000001,
            ...
        };

        // file-descriptors
        int     fd;
        int     fd_metadata;          // fd for the meta-data
        // ints
        int     magic;
        int     flags;
        unsigned int  size;
        unsigned int  offset;
        int     bufferType;
        uint64_t base __attribute__((aligned(8)));
        unsigned int  offset_metadata;
        // The gpu address mapped into the mmu.
        uint64_t gpuaddr __attribute__((aligned(8)));
        int     format;
        int     width;   // holds aligned width of the actual buffer allocated
        int     height;  // holds aligned height of the  actual buffer allocated
        uint64_t base_metadata __attribute__((aligned(8)));
        int unaligned_width;   // holds width client asked to allocate
        int unaligned_height;  // holds height client asked to allocate

#ifdef __cplusplus
        static const int sNumFds = 2;
        static inline int sNumInts() {
            return ((sizeof(private_handle_t) - sizeof(native_handle_t)) /
                    sizeof(int)) - sNumFds;
        }
        static const int sMagic = 'gmsm';
        ...
}

`private_handle_t` 描述了缓存区使用的文件描述符 `fd, fd_metadata` 、大小、偏移量、基地址、长宽、格式、没有对齐的长宽等等，而 `sNumFds` 对应 `nativeHandle.numFds` ； `sNumInts` 对应 `nativeHandle.numInts` ，即除了文件描述符之外，该数据结构的长度。  
小结： `private_handle_t` 在各个模块之间传递的时候很不方便，而如果用 `native_handle` 的来传递，就可以消除平台的差异性。一个简单示意图描述两者的关系：

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0109-android-camera-5-hal-buffer_handle_t.png)

至此，我们可以简单的理解为 `native_handle, native_handle_t, private_handle_t, buffer_handle_t` 表示的是同一块内存。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#ANativeWindowBuffer "ANativeWindowBuffer")`ANativeWindowBuffer`

先了解 `android_native_base_t` 数据结构的定义：

typedef struct android_native_base_t
{
    /* a magic value defined by the actual EGL native type */
    int magic;

    /* the sizeof() of the actual EGL native type */
    int version;

    void* reserved[4];

    /* reference-counting interface */
    void (*incRef)(struct android_native_base_t* base);
    void (*decRef)(struct android_native_base_t* base);
} android_native_base_t;

`android_native_base_t` 中 `incRef/decRef` 主要功能是：为了把派生类和 `Android` 所有 `class` 的老祖宗 `RefBase` 联系起来所预留的函数指针。

再看 `ANativeWindowBuffer` 数据结构的定义：

// nativebase.h
typedef struct ANativeWindowBuffer
{
#ifdef __cplusplus
    ANativeWindowBuffer() {
        common.magic = ANDROID_NATIVE_BUFFER_MAGIC;
        common.version = sizeof(ANativeWindowBuffer);
        memset(common.reserved, 0, sizeof(common.reserved));
    }

    // Implement the methods that sp<ANativeWindowBuffer> expects so that it
    // can be used to automatically refcount ANativeWindowBuffer's.
    void incStrong(const void* /*id*/) const {
        common.incRef(const_cast<android_native_base_t*>(&common));
    }
    void decStrong(const void* /*id*/) const {
        common.decRef(const_cast<android_native_base_t*>(&common));
    }
#endif

    struct android_native_base_t common;

    int width;
    int height;
    int stride;
    int format;
    int usage_deprecated;
    uintptr_t layerCount;

    void* reserved[1];

    const native_handle_t* handle;
    uint64_t usage;

    // we needed extra space for storing the 64-bits usage flags
    // the number of slots to use from reserved_proc depends on the
    // architecture.
    void* reserved_proc[8 - (sizeof(uint64_t) / sizeof(void*))];
} ANativeWindowBuffer_t;

typedef struct ANativeWindowBuffer ANativeWindowBuffer;

// Old typedef for backwards compatibility.
typedef ANativeWindowBuffer_t android_native_buffer_t;

`ANativeWindowBuffer` 中使用了 `native_handle_t` 指针，同时该结构体中也有长宽、格式、步进等基本描述信息；也就是 `ANativeWindowBuffer` 描述的是一块 `Window` 相关的缓存区。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#ANativeWindow "ANativeWindow")`ANativeWindow`

`ANativeWindow` 数据结构的定义：

// window.h
struct ANativeWindow
{
#ifdef __cplusplus
    ANativeWindow()
        : flags(0), minSwapInterval(0), maxSwapInterval(0), xdpi(0), ydpi(0)
    {
        common.magic = ANDROID_NATIVE_WINDOW_MAGIC;
        common.version = sizeof(ANativeWindow);
        memset(common.reserved, 0, sizeof(common.reserved));
    }

    /* Implement the methods that sp<ANativeWindow> expects so that it
       can be used to automatically refcount ANativeWindow's. */
    void incStrong(const void* /*id*/) const {
        common.incRef(const_cast<android_native_base_t*>(&common));
    }
    void decStrong(const void* /*id*/) const {
        common.decRef(const_cast<android_native_base_t*>(&common));
    }
#endif

    struct android_native_base_t common;

    /* flags describing some attributes of this surface or its updater */
    const uint32_t flags;

    /* min swap interval supported by this updated */
    const int   minSwapInterval;

    /* max swap interval supported by this updated */
    const int   maxSwapInterval;

    /* horizontal and vertical resolution in DPI */
    const float xdpi;
    const float ydpi;

    /* Some storage reserved for the OEM's driver. */
    intptr_t    oem[4];

    int     (*setSwapInterval)(struct ANativeWindow* window,
                int interval);
    int     (*dequeueBuffer_DEPRECATED)(struct ANativeWindow* window,
                struct ANativeWindowBuffer** buffer);
    int     (*lockBuffer_DEPRECATED)(struct ANativeWindow* window,
                struct ANativeWindowBuffer* buffer);
    int     (*queueBuffer_DEPRECATED)(struct ANativeWindow* window,
                struct ANativeWindowBuffer* buffer);
    int     (*query)(const struct ANativeWindow* window,
                int what, int* value);
    int     (*perform)(struct ANativeWindow* window,
                int operation, ... );
    int     (*cancelBuffer_DEPRECATED)(struct ANativeWindow* window,
                struct ANativeWindowBuffer* buffer);
    int     (*dequeueBuffer)(struct ANativeWindow* window,
                struct ANativeWindowBuffer** buffer, int* fenceFd);
    int     (*queueBuffer)(struct ANativeWindow* window,
                struct ANativeWindowBuffer* buffer, int fenceFd);
    int     (*cancelBuffer)(struct ANativeWindow* window,
                struct ANativeWindowBuffer* buffer, int fenceFd);
};

 /* Backwards compatibility: use ANativeWindow (struct ANativeWindow in C).
  * android_native_window_t is deprecated.
  */
typedef struct ANativeWindow android_native_window_t __deprecated;

从数据结构定义中可以看出， `ANativeWindow` 和窗口属性相关，它表示的是一个底层实现的窗口，定义的各种函数指针都是对 `ANativeWindowBuffer` 内存的操作；而 `fenceFd` 可以看成这个 `buffer` 的锁。

小结：不管是 `ANativeWindow, ANativeWindowBuffer` 它们都包含 `android_native_base_t` 结构体，但是都没有对 `incRef, decRef` 赋值；可以认为 `ANativeWindow, ANativeWindowBuffer` 为抽象数据结构。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#ANativeObjectBase-%E6%A8%A1%E6%9D%BF "ANativeObjectBase 模板")`ANativeObjectBase` 模板

`ANativeObjectBase` 是一个模板类，定义如下：

// ANativeObjectBase.h
template <typename NATIVE_TYPE, typename TYPE, typename REF,
        typename NATIVE_BASE = android_native_base_t>
class ANativeObjectBase : public NATIVE_TYPE, public REF
{
public:
    // Disambiguate between the incStrong in REF and NATIVE_TYPE
    void incStrong(const void* id) const {
        REF::incStrong(id);
    }
    void decStrong(const void* id) const {
        REF::decStrong(id);
    }

protected:
    typedef ANativeObjectBase<NATIVE_TYPE, TYPE, REF, NATIVE_BASE> BASE;
    ANativeObjectBase() : NATIVE_TYPE(), REF() {
        NATIVE_TYPE::common.incRef = incRef;
        NATIVE_TYPE::common.decRef = decRef;
    }
    static inline TYPE* getSelf(NATIVE_TYPE* self) {
        return static_cast<TYPE*>(self);
    }
    static inline TYPE const* getSelf(NATIVE_TYPE const* self) {
        return static_cast<TYPE const *>(self);
    }
    static inline TYPE* getSelf(NATIVE_BASE* base) {
        return getSelf(reinterpret_cast<NATIVE_TYPE*>(base));
    }
    static inline TYPE const * getSelf(NATIVE_BASE const* base) {
        return getSelf(reinterpret_cast<NATIVE_TYPE const*>(base));
    }
    static void incRef(NATIVE_BASE* base) {
        ANativeObjectBase* self = getSelf(base);
        self->incStrong(self);
    }
    static void decRef(NATIVE_BASE* base) {
        ANativeObjectBase* self = getSelf(base);
        self->decStrong(self);
    }
};

} // namespace android
#endif // __cplusplus

`ANativeObjectBase` 模板类的主要作用就是实现 `incRef, decRef` 引用计数，以及父类子类的类型转换。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#GraphicBuffer "GraphicBuffer")`GraphicBuffer`

先看 `GraphicBuffer` 的头文件定义：

// GraphicBuffer.h
class GraphicBuffer
    : public ANativeObjectBase<ANativeWindowBuffer, GraphicBuffer, RefBase>,
      public Flattenable<GraphicBuffer>
{...}

`GraphicBuffer` 使用 `ANativeObjectBase` 模板，即 `GraphicBuffer` 就是 `ANativeWindowBuffer` 的一种具体实现；而 `ANativeWindowBuffer.common` 成员的两个函数指针 `incRef, decRef` 指向了 `GraphicBuffer` 的另一个基类 `RefBase` 的 `incStrong, decStrong` ；而 `ANativeWindowBuffer` 可以看做是把 `buffer_handle_t` 包了一层，所以 `GraphicBuffer` 也是指向的一块缓存区。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#Surface "Surface")`Surface`

`Surface` 的头文件定义：

// Surface.h
class Surface
    : public ANativeObjectBase<ANativeWindow, Surface, RefBase>
{
    ...
protected:
    ...
    struct BufferSlot {
        sp<GraphicBuffer> buffer;
        Region dirtyRegion;
    };
    BufferSlot mSlots[NUM_BUFFER_SLOTS];
    ...
}

`Surface` 也使用了 `ANativeObjectBase` 模板，即 `Surface` 就是 `ANativeWindow` 的一种具体实现，同样也继承了 `RefBase` 实现引用计数。另外成员数据结构 `BufferSlot` 是对 `GraphicBuffer` 的包装，而 `mSlots` 数组表示每个 `Surface` 中包含 `NUM_BUFFER_SLOTS` 个 `GraphicBuffer` 缓存。  
而 `Surface` 的构造函数中，也将 `ANativeWindow` 的函数指针进行了赋值：

// Surface.cpp
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, 
    bool controlledByApp)
      : mGraphicBufferProducer(bufferProducer),
        mCrop(Rect::EMPTY_RECT),
        mBufferAge(0),
        mGenerationNumber(0),
        mSharedBufferMode(false),
        mAutoRefresh(false),
        mSharedBufferSlot(BufferItem::INVALID_BUFFER_SLOT),
        mSharedBufferHasBeenQueued(false),
        mQueriedSupportedTimestamps(false),
        mFrameTimestampsSupportsPresent(false),
        mEnableFrameTimestamps(false),
       mFrameEventHistory(std::make_unique<ProducerFrameEventHistory>()){
    // Initialize the ANativeWindow function pointers.
    ANativeWindow::setSwapInterval  = hook_setSwapInterval;
    ANativeWindow::dequeueBuffer    = hook_dequeueBuffer;
    ANativeWindow::cancelBuffer     = hook_cancelBuffer;
    ANativeWindow::queueBuffer      = hook_queueBuffer;
    ANativeWindow::query            = hook_query;
    ANativeWindow::perform          = hook_perform;
    ...
}

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E5%B0%8F%E7%BB%93 "小结")小结

- `native_handle/native_handle_t` 是 `private_handle_t` 的抽象表示方法，消除平台相关性；方便 `private_handle_t` 所表示的缓存区可以在 `Android` 各个层次之间传递；而 `buffer_handle_t` 是指向他们的指针
- `ANativeWindowBuffer` 将 `buffer_handle_t` 进行了包装；`ANativeWindow, ANativeWindowBuffer` 都继承于 `android_native_base_t` ，它定义了引用计数两个函数指针；可以认为 `ANativeWindow, ANativeWindowBuffer` 为抽象数据结构，表示窗口和其对应缓存
- `GraphicBuffer, Surface` 都使用了模版类 `ANativeObjectBase` ，都继承了 `RefBase` 实现 `incRef, decRef` 引用计数；它们是具体的实现类，即实现具体的窗口缓存和窗口
- `Surface` 的成员 `BufferSlot mSlots[NUM_BUFFER_SLOTS];` 可以看作是 `sp<GraphicBuffer>` 类型的数组；也就是说每个 `Surface` 中都包含有 `NUM_BUFFER_SLOTS` 个 `GraphicBuffer`
- `Surface, GraphicBuffer` 是图形显示系统的高层类，后续主要围绕这两个类来介绍；一个代表窗口，一个代表窗口对应的缓存

[Buffer/Window 体系缓存，查看大图](https://upload-images.jianshu.io/upload_images/606437-20bbb0ebba7fd244.png)

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-surface-buffer.png)

`BufferQueue` 中的 `Buffer` 对象，我们用的都是 `GraphicBuffer` 。 `Surface` 是 `Andorid` 窗口的描述，是 `ANativeWindow` 的实现；同样 `GraphicBuffer` 是 `Android` 中图形 `Buffer` 的描述，是 `ANativeWindowBuffer` 的实现。而一个窗口可以有多个 `Buffer` 。

## [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#libui-%E5%BA%93 "libui 库")`libui` 库

`libui.so` 库主要是 `GraphicBuffer` 缓存相关的代码，包含缓存分配，映射当当前进程等等，而 `IAllocator, IMapper` 具体是在 `HAL` 中实现的。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E4%BB%A3%E7%A0%81%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84 "代码目录结构")代码目录结构

`frameworks/native/libs/ui` 目录结构：

ui
├── Android.bp
├── ColorSpace.cpp
├── DebugUtils.cpp
├── Fence.cpp
├── FenceTime.cpp
├── FrameStats.cpp
├── Gralloc2.cpp
├── GraphicBufferAllocator.cpp
├── GraphicBuffer.cpp
├── GraphicBufferMapper.cpp
├── HdrCapabilities.cpp
├── include
│   └── ui
│       ├── ANativeObjectBase.h
│       ├── BufferQueueDefs.h
│       ├── ColorSpace.h
│       ├── DebugUtils.h
│       ├── DisplayInfo.h
│       ├── DisplayStatInfo.h
│       ├── Fence.h
│       ├── FenceTime.h
│       ├── FloatRect.h
│       ├── FrameStats.h
│       ├── Gralloc2.h
│       ├── GraphicBufferAllocator.h
│       ├── GraphicBuffer.h
│       ├── GraphicBufferMapper.h
│       ├── HdrCapabilities.h
│       ├── PixelFormat.h
│       ├── Point.h
│       ├── Rect.h
│       ├── Region.h
│       └── UiConfig.h
├── MODULE_LICENSE_APACHE2
├── NOTICE
├── PixelFormat.cpp
├── Rect.cpp
├── Region.cpp
├── tests
│   ├── Android.bp
│   ├── colorspace_test.cpp
│   └── Region_test.cpp
├── tools
│   ├── Android.bp
│   └── lutgen.cpp
└── UiConfig.cpp

4 directories, 42 files

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#GraphicBufferAllocator-GraphicBufferMapper "GraphicBufferAllocator/GraphicBufferMapper")`GraphicBufferAllocator/GraphicBufferMapper`

它们两个都是包装类，包装了 `IAllocator, IMapper` ，而这两个类都是在 `HAL Gralloc2` 中实现的。

- `GraphicBufferAllocator`  
    缓存分配，包装了 `IAllocator` 类。
- `GraphicBufferMapper`  
    缓存映射到当前进程，包装了 `IMapper` 类。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#GraphicBuffer-1 "GraphicBuffer")`GraphicBuffer`

`GraphicBuffer` 继承 `ANativeWindowBuffer` ，并持有 `GraphicBufferMapper` 映射对应的缓存区。

class GraphicBuffer : 
    public ANativeObjectBase<ANativeWindowBuffer, GraphicBuffer,RefBase>,
    public Flattenable<GraphicBuffer>
{
public:
    static sp<GraphicBuffer> from(ANativeWindowBuffer *);
    ...
private:
    GraphicBufferMapper& mBufferMapper;
    ssize_t mInitCheck;
    uint64_t mId;
    uint32_t mGenerationNumber;
};

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#Fence-%E6%9C%BA%E5%88%B6 "Fence 机制")`Fence` 机制

`Fence` 是一种同步机制，主要用于 `GraphicBuffer` 的同步。它主要被用来处理跨硬件的情况，尤其是 `CPU, GPU, HWC` 之间的同步，另外它还可以用于多个时间点之间的同步。 `GPU` 编程和纯 `CPU` 编程一个很大的不同是它是异步的，也就是说当我们调用 `GL command` 返回时这条命令并不一定完成了，只是把这个命令放在本地的 `command buffer` 里，而 `Fence` 机制就是解决这些同步问题的。  
`Fence` 顾名思义就是把先到的拦住，等后来的，两者步调一致了再往前走。抽象地说，`Fence` 包含了同一或不同时间轴上的多个时间点，只有当这些点同时到达时 `Fence` 才会被触发。 `Fence` 可以由硬件实现 `Graphic driver`，也可以由软件实现 `Android kernel` 中的 `sw_sync` 。

`Fence` 的主要实现代码路径：

frameworks/native/libs/ui/Fence.cpp
system/core/libsync/sync.c
kernel/drivers/base/sync.c
frameworks/native/libs/gui/SyncFeatures.cpp

总得来说， `kernel driver` 部分是同步的主要实现，`libsync` 是对 `driver` 接口的封装， `Fence` 是对 `libsync` 的进一步的 `C++` 封装。 `Fence` 会被作为 `GraphicBuffer` 的附属随着 `GraphicBuffer` 在生产者和消费间传输； `SyncFeatures` 用以查询系统支持的同步机制。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#DisplayInfo-%E6%98%BE%E7%A4%BA%E4%BF%A1%E6%81%AF "DisplayInfo 显示信息")`DisplayInfo` 显示信息

// DisplayInfo.h
struct DisplayInfo {
    uint32_t w{0};                      // 屏幕的宽
    uint32_t h{0};                      // 屏幕的高
    float xdpi{0};                      // 屏幕 x 方向每英寸的像素点
    float ydpi{0};                      // 屏幕 y 方向每英寸的像素点
    float fps{0};                       // FPS 屏幕的刷新率
    float density{0};                   // 屏幕的密度
    uint8_t orientation{0};             // 屏幕的旋转方式
    bool secure{false};                 // 屏幕是否是安全的
    nsecs_t appVsyncOffset{0};          // App 的 Vsync 的偏移
    nsecs_t presentationDeadline{0};    // 显示的最后时间
};

/* Display orientations as defined in Surface.java 
    and ISurfaceComposer.h. */
enum {
    DISPLAY_ORIENTATION_0 = 0,
    DISPLAY_ORIENTATION_90 = 1,
    DISPLAY_ORIENTATION_180 = 2,
    DISPLAY_ORIENTATION_270 = 3
};

}; // namespace android

`DisplayInfo` 结构体包含了显示屏幕的基本信息：

- 屏幕分辨率 `Resolution`  
    屏幕的宽高是用分辨率 `Resolution` 来描述的，也就是有多少个像素点。屏幕宽度，即屏幕横向可以显示多少个像素点；屏幕高度，即屏幕纵向可以显示多少给像素点。平常所说的 `720P: 1080x720` 屏幕，即横向可以显示 1080 个像素点，纵向可以显示 720 个像素点。如下为常见屏幕分辨率：  
    
    ![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-resolution.png)
    
- 屏幕 `DPI`  
    `DPI: Dots Per Inch` 每英寸点数，是一个度量单位，表示屏幕每英寸上有多少个物理点。常见屏幕物理大小，是用英寸来描述屏幕对角线的长度，比如 `IPhone X` 的大小 5.8 寸，即屏幕对角线长度为 5.8 英寸。* _标准 `DPI` 为 `160dpi` *_，人类视网膜级通常为 `300dpi` 。  
    `PPI: Pixel Per Inch` 每英寸像素，也是度量单位，表示每英寸显示多少个像素。通常情况下 `DPI, PPI` 设为相同，表示每个物理点显示一个像素；但是好一点的显示器，可能 `DPI` 比 `PPI` 大，即一个像素由多个物理点来显示。
- 密度 `Density`  
    `DIP: Density Independent Pixels` 设备无关像素，通常简写为 `DP=DIP`，请注意 `DPI` 做好区分。 `DP` 表示这个像素的数值是和设备无关的，那实际转换时怎么转换呢？  
    `Density` 密度，实际是一个缩放因子，它表示当前设备实际 `DPI` 和标准 `DPI` 的比例值；比如设备实际 `DPI` 为 `320dpi` ，那么 `density=320/160=2` ，即 `density` 为 2 。有了 `density` 之后， `dp, px` 可以使用公式来转换 `px=density*dp` 。  
    所以我们在 `APP` 布局设计中，所有显示设置的距离，通常使用 `dp` 来计算，来规避不同屏幕特性。
- 屏幕刷新率 `FPS`  
    这里屏幕刷新率使用 `FPS` 来表示，不是 `Hz` ，表示屏幕每秒能显示多少帧数据；通常为 60 fps ，即 16 ms 刷新一次。
- 屏幕旋转方向 `orientations`  
    手机默认竖屏，0 表示竖屏， 180 表示横屏。
- 屏幕安全性 `secure`  
    这主要是用于 `DRM` 数字版权保护时，确保显示的设备是安全的，以防止 `DRM` 的内容被在显示的过程中被截取，只有安全的设备才能显示 `DRM` 的内容。 `Android` 默认所有的非虚拟显示都是安全的。
- `appVsyncOffset, presentationDeadline`  
    这两个都和 `Vsync` 有关； `appVsyncOffset` 是一个偏移量，在系统或硬件 `Vsync` 的基础上做一些偏移； `presentationDeadline` 表示，一帧数据必现在这个时间内显示出来。

## [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#libgui-%E5%BA%93 "libgui 库")`libgui` 库

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E4%BB%A3%E7%A0%81%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84-1 "代码目录结构")代码目录结构

`frameworks/native/libs/gui` 目录结构：

gui/
├── Android.bp
├── BitTube.cpp
├── BufferItemConsumer.cpp
├── BufferItem.cpp
├── bufferqueue
│   └── 1.0
│       ├── B2HProducerListener.cpp
│       └── H2BGraphicBufferProducer.cpp
├── BufferQueueConsumer.cpp
├── BufferQueueCore.cpp
├── BufferQueue.cpp
├── BufferQueueProducer.cpp
├── BufferSlot.cpp
├── CleanSpec.mk
├── ConsumerBase.cpp
├── CpuConsumer.cpp
├── DisplayEventReceiver.cpp
├── FrameTimestamps.cpp
├── GLConsumer.cpp
├── GuiConfig.cpp
├── IConsumerListener.cpp
├── IDisplayEventConnection.cpp
├── IGraphicBufferConsumer.cpp
├── IGraphicBufferProducer.cpp
├── include
│   ├── gui
│   │   ├── BufferItemConsumer.h
│   │   ├── BufferItem.h
│   │   ├── bufferqueue
│   │   │   └── 1.0
│   │   │       ├── B2HProducerListener.h
│   │   │       └── H2BGraphicBufferProducer.h
│   │   ├── BufferQueueConsumer.h
│   │   ├── BufferQueueCore.h
│   │   ├── BufferQueueDefs.h
│   │   ├── BufferQueue.h
│   │   ├── BufferQueueProducer.h
│   │   ├── BufferSlot.h
│   │   ├── ConsumerBase.h
│   │   ├── CpuConsumer.h
│   │   ├── DisplayEventReceiver.h
│   │   ├── FrameTimestamps.h
│   │   ├── GLConsumer.h
│   │   ├── GuiConfig.h
│   │   ├── IConsumerListener.h
│   │   ├── IDisplayEventConnection.h
│   │   ├── IGraphicBufferConsumer.h
│   │   ├── IGraphicBufferProducer.h
│   │   ├── IProducerListener.h
│   │   ├── ISurfaceComposerClient.h
│   │   ├── ISurfaceComposer.h
│   │   ├── OccupancyTracker.h
│   │   ├── StreamSplitter.h
│   │   ├── SurfaceComposerClient.h
│   │   ├── SurfaceControl.h
│   │   ├── Surface.h
│   │   └── view
│   │       └── Surface.h
│   └── private
│       └── gui
│           ├── BitTube.h
│           ├── ComposerService.h
│           ├── LayerState.h
│           └── SyncFeatures.h
├── IProducerListener.cpp
├── ISurfaceComposerClient.cpp
├── ISurfaceComposer.cpp
├── LayerState.cpp
├── OccupancyTracker.cpp
├── StreamSplitter.cpp
├── SurfaceComposerClient.cpp
├── SurfaceControl.cpp
├── Surface.cpp
├── SyncFeatures.cpp
├── tests
│   ├── ...
└── view
    └── Surface.cpp

11 directories, 96 files

其中 `H2B, B2H` 表示 `Framework Buffer` 和 `HAL` 层数据结构的相互转换；实际上代表的是同一样东西，方便各层内部使用。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#IGraphicBufferProducer-IProducerListener-%E7%94%9F%E4%BA%A7%E8%80%85 "IGraphicBufferProducer/IProducerListener 生产者")`IGraphicBufferProducer/IProducerListener` 生产者

`IGraphicBufferProducer` 是生产者接口，实现了 `IInterface` 可以用于跨进程通信。

// IGraphicBufferProducer.h
class IGraphicBufferProducer : public IInterface
{
public:
    DECLARE_HYBRID_META_INTERFACE(GraphicBufferProducer, 
        HGraphicBufferProducer)
    ...
    // 根据指定参数申请一块 Buffer ，索引值为 slot ，同步为 fence
    // 从 BufferQueue 中出队一块缓存 GraphicBuffer
    virtual status_t dequeueBuffer(int* slot, sp<Fence>* fence, 
        uint32_t w, uint32_t h,
        PixelFormat format, uint64_t usage, uint64_t* outBufferAge,
        FrameEventHistoryDelta* outTimestamps) = 0;
    // 获取 slot 位置的 GraphicBuffer
    virtual status_t requestBuffer(int slot, sp<GraphicBuffer>* buf) = 0;
    // 客户端已经向 slot 位置的 Buffer 填充完数据 
    // IGraphicBufferProducer 得到 Buffer 的输入信息，
    // slot 这块缓存 GraphicBuffer 进入队列 BufferQueue
    virtual status_t queueBuffer(int slot, const QueueBufferInput& input,
            QueueBufferOutput* output) = 0;
    // 释放 slot 位置的 GraphicBuffer
    virtual status_t detachBuffer(int slot) = 0;
    virtual status_t detachNextBuffer(sp<GraphicBuffer>* outBuffer,
            sp<Fence>* outFence) = 0;
    // 根据指定的 buffer 获取 slot
    virtual status_t attachBuffer(int* outSlot,
            const sp<GraphicBuffer>& buffer) = 0;
    // 释放 slot 位置的 buffer
    virtual status_t cancelBuffer(int slot, const sp<Fence>& fence) = 0;
    ...
    // 客户端根据 api 类型，连接 IGraphicBufferProducer ，
    // 客户端得到缓存的相关信息 QueueBufferOutput
    virtual status_t connect(const sp<IProducerListener>& listener,
            int api, bool producerControlledByApp, 
            QueueBufferOutput* output) = 0;
    // 断开连接
    virtual status_t disconnect(int api, 
        DisconnectMode mode = DisconnectMode::Api) = 0;
    // 获取消费者名称
    virtual String8 getConsumerName() const = 0;
    ...
};

`IProducerListener` 是 `IGraphicBufferProducer` 对应的回调接口。

// IProducerListener.h
class ProducerListener : public virtual RefBase
{
public:
    ProducerListener() {}
    virtual ~ProducerListener();

    virtual void onBufferReleased() = 0; // Asynchronous
    virtual bool needsReleaseNotify() = 0;
};

class IProducerListener : public ProducerListener, public IInterface
{
public:
    DECLARE_META_INTERFACE(ProducerListener)
};

class BnProducerListener : public BnInterface<IProducerListener>
{
public:
    virtual status_t onTransact(uint32_t code, const Parcel& data,
            Parcel* reply, uint32_t flags = 0);
    virtual bool needsReleaseNotify();
};

class DummyProducerListener : public BnProducerListener
{
public:
    virtual ~DummyProducerListener();
    virtual void onBufferReleased() {}
    virtual bool needsReleaseNotify() { return false; }
};

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#IGraphicBufferConsumer-IConsumerListener-%E6%B6%88%E8%B4%B9%E8%80%85 "IGraphicBufferConsumer/IConsumerListener 消费者")`IGraphicBufferConsumer/IConsumerListener` 消费者

`IGraphicBufferConsumer` 是消费者接口，实现了 `IInterface` 可以用于跨进程通信。

// IGraphicBufferConsumer.h
class IGraphicBufferConsumer : public IInterface {
public:
    DECLARE_META_INTERFACE(GraphicBufferConsumer)
    ...
    // 从 BufferQueue 中获取一块准备好了的缓存 GraphicBuffer
    virtual status_t acquireBuffer(BufferItem* buffer, 
        nsecs_t presentWhen, uint64_t maxFrameNumber = 0) = 0;
    // 释放 slot 位置的缓存
    virtual status_t detachBuffer(int slot) = 0;
    // 根据指定的 GraphicBuffer 获取 slot
    virtual status_t attachBuffer(int* outSlot, 
        const sp<GraphicBuffer>& buffer) = 0;
    // 移除指定 slot 位置的缓存
    virtual status_t releaseBuffer(int buf, uint64_t frameNumber, 
        EGLDisplay display, EGLSyncKHR fence, 
        const sp<Fence>& releaseFence) = 0;
    ...
    // 连接一个消费者进入 BufferQueue
    virtual status_t consumerConnect(const sp<IConsumerListener>& consumer,
                                     bool controlledByApp) = 0;
    // 从 BufferQueue 断开连接
    virtual status_t consumerDisconnect() = 0;
    ...
};

`IConsumerListener` 是 `IGraphicBufferConsumer` 对应的回调接口：

// IConsumerListener.h
class ConsumerListener : public virtual RefBase {
public:
    ConsumerListener() {}
    virtual ~ConsumerListener();

    virtual void onDisconnect() {} /* Asynchronous */
    virtual void onFrameAvailable(const BufferItem& item) 
        = 0; /* Asynchronous */
    virtual void onFrameReplaced(const BufferItem& /* item */)
        {} /* Asynchronous */
    virtual void onBuffersReleased() = 0; /* Asynchronous */
    virtual void onSidebandStreamChanged() = 0; /* Asynchronous */
    virtual void addAndGetFrameTimestamps(const NewFrameEventsEntry* 
        /*newTimestamps*/, FrameEventHistoryDelta* /*outDelta*/) {}
};

class IConsumerListener : public ConsumerListener, public IInterface {
public:
    DECLARE_META_INTERFACE(ConsumerListener)
};

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#BufferItem "BufferItem")`BufferItem`

`BufferItem` 描述了一块缓存 `GraphicBuffer` ，以及位置 `mSlot` ，同步 `mFence` 等等信息。

// BufferItem.h
class BufferItem : public Flattenable<BufferItem> {
    ...
    sp<GraphicBuffer> mGraphicBuffer;
    sp<Fence> mFence;
    Rect mCrop;
    ...
    android_dataspace mDataSpace;
    uint64_t mFrameNumber;
    int mSlot;
    ...
}

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#BufferSlot "BufferSlot")`BufferSlot`

`BufferSlot` 记录了当前 `slot` 位置的缓存 `GraphicBuffer` ，以及对应状态， `EGL` 相关信息。

// BufferSlot.h
struct BufferSlot {

    BufferSlot()
    : mGraphicBuffer(nullptr),
      mEglDisplay(EGL_NO_DISPLAY),
      mBufferState(),
      mRequestBufferCalled(false),
      mFrameNumber(0),
      mEglFence(EGL_NO_SYNC_KHR),
      mFence(Fence::NO_FENCE),
      mAcquireCalled(false),
      mNeedsReallocation(false) {
    }

    sp<GraphicBuffer> mGraphicBuffer;
    EGLDisplay mEglDisplay;
    BufferState mBufferState;
    bool mRequestBufferCalled;
    uint64_t mFrameNumber;
    EGLSyncKHR mEglFence;
    sp<Fence> mFence;
    bool mAcquireCalled;
    bool mNeedsReallocation;
};

`BufferQueueDefs` 中定义了一个 `BufferSlot` 的数组结构类型 `SlotsType` 。

// BufferQueueDefs.h
namespace android {
    class BufferQueueCore;

    namespace BufferQueueDefs {
        typedef BufferSlot SlotsType[NUM_BUFFER_SLOTS];
    } // namespace BufferQueueDefs
} // namespace android

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#BufferQueueCore "BufferQueueCore")`BufferQueueCore`

`BufferQueueCore` 是生产者消费模型的核心，如下是几个重要的成员变量：

// BufferQueueCore.h
class BufferQueueCore : public virtual RefBase {
public:
    ...
    typedef Vector<BufferItem> Fifo;
private:
    ...
    String8 mConsumerName;
    sp<IConsumerListener> mConsumerListener;
    sp<IProducerListener> mConnectedProducerListener;

    BufferQueueDefs::SlotsType mSlots;
    Fifo mQueue;
    ...
}

- `mQueue`  
    是一个新建先出队列，存储了一队 `BufferItem` 数据，即一组缓存区。
- `mSlots`  
    一个数组，保存了 `BufferSlot` 数据，每个 `slot` 位置对应一个缓存。
- `mConsumerListener`  
    当前生产消费模型中的，消费者回调接口。
- `mConnectedProducerListener`  
    当前生产消费模型中的，生产者回调接口。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#BufferQueueProducer-BufferQueueConsumer-%E7%94%9F%E4%BA%A7%E8%80%85-%E6%B6%88%E8%B4%B9%E8%80%85%E5%AE%9E%E7%8E%B0%E7%B1%BB "BufferQueueProducer/BufferQueueConsumer 生产者/消费者实现类")`BufferQueueProducer/BufferQueueConsumer` 生产者 / 消费者实现类

`BufferQueueProducer` 是 `IGraphicBufferConsumer` 的实现类，实现了生产者对应的功能。

// BufferQueueProducer.h
class BufferQueueProducer : public BnGraphicBufferProducer,
                            private IBinder::DeathRecipient {
    ...
private:
    ...
    sp<BufferQueueCore> mCore;
    // This references mCore->mSlots.
    BufferQueueDefs::SlotsType& mSlots;
    String8 mConsumerName;
    ...
    sp<Fence> mLastQueueBufferFence;
    Rect mLastQueuedCrop;
    uint32_t mLastQueuedTransform;
}

`BufferQueueProducer` 中持有 `BufferQueueCore` 对象； `mSlots` 指向 `mCore->mSlots` ；同时保持了生产消费模型中，对应消费者的名称。

`BufferQueueConsumer` 是 `IGraphicBufferConsumer` 的实现类，实现了消费者对应的功能。

// BufferQueueConsumer.h
class BufferQueueConsumer : public BnGraphicBufferConsumer {
    ...
private:
    sp<BufferQueueCore> mCore;
    // This references mCore->mSlots.
    BufferQueueDefs::SlotsType& mSlots;
    String8 mConsumerName;
}

`BufferQueueConsumer` 中持有 `BufferQueueCore` 对象； `mSlots` 指向 `mCore->mSlots` 。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#BufferQueue-%E6%A8%A1%E5%9E%8B "BufferQueue 模型")`BufferQueue` 模型

`BufferQueue` 类是 `Android` 中所有图形处理操作的核心。它的作用很简单：将生成图形数据缓冲区的一方（生产者）连接到接受数据以进行显示或进一步处理的一方（消费者）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 `BufferQueue` 。  
基本用法很简单：生产者请求一个可用的缓冲区 `dequeueBuffer` ，并指定一组特性，包括宽度、高度、像素格式和用法标记；生产者填充缓冲区并将其返回到队列 `queueBuffer` 。随后消费者获取该缓冲区 `acquireBuffer` ，并使用该缓冲区的内容。当消费者操作完毕后，将该缓冲区返回到队列 `releaseBuffer` 。  
最新的 Android 设备支持 “同步框架”，这使得系统能够在与可以异步处理图形数据的硬件组件结合使用时提高工作效率。例如，生产者可以提交一系列 `OpenGL ES` 绘制命令，然后在渲染完成之前将输出缓冲区加入队列。该缓冲区伴有一个栅栏，当内容准备就绪时，栅栏会发出信号。当该缓冲区返回到空闲列表时，会伴有第二个栅栏，因此消费者可以在内容仍在使用期间释放该缓冲区。该方法缩短了缓冲区通过系统时的延迟时间，并提高了吞吐量。  
队列的一些特性（例如可以容纳的最大缓冲区数）由生产者和消费者联合决定。但是 `BufferQueue` 负责根据需要分配缓冲区。除非特性发生变化，否则将会保留缓冲区；例如，如果生产者请求具有不同大小的缓冲区，则系统会释放旧的缓冲区，并根据需要分配新的缓冲区。  
生产者和消费者可以存在于不同的进程中； `BufferQueue` 永远不会复制缓冲区内容（移动如此多的数据是非常低效的操作），缓冲区始终通过句柄进行传递。

// BufferQueue.h
class BufferQueue {
public:
    ...
    typedef ::android::ConsumerListener ConsumerListener;

    // ProxyConsumerListener 是 ConsumerListener 弱引用实现
    class ProxyConsumerListener : public BnConsumerListener {
    public:
        explicit ProxyConsumerListener(const wp<ConsumerListener>& 
            consumerListener);
        ~ProxyConsumerListener() override;
        void onDisconnect() override;
        void onFrameAvailable(const BufferItem& item) override;
        void onFrameReplaced(const BufferItem& item) override;
        void onBuffersReleased() override;
        void onSidebandStreamChanged() override;
        void addAndGetFrameTimestamps(
                const NewFrameEventsEntry* newTimestamps,
                FrameEventHistoryDelta* outDelta) override;
    private:
        // 弱引用
        wp<ConsumerListener> mConsumerListener;
    };

    // BufferQueue manages a pool of gralloc memory slots to be used by
    // producers and consumers. allocator is used to allocate all the
    // needed gralloc buffers.
    static void createBufferQueue(
            sp<IGraphicBufferProducer>* outProducer,
            sp<IGraphicBufferConsumer>* outConsumer,
            bool consumerIsSurfaceFlinger = false);

    BufferQueue() = delete; // Create through createBufferQueue
};

`BufferQueue` 的头文件定义很简单：

- 定义了一个 `ConsumerListener` 的弱引用
- 整个类只有一个函数 `createBufferQueue` ，它将参数中的 `IGraphicBufferProducer, IGraphicBufferConsumer` 消费者关联起来
- 没有构造函数，只能通过 `createBufferQueue` 来创建对象
- `BufferQueue` 中并不包含队列数据结构来存储缓存，仅仅连接了生产者、消费者两者的关系

来看 `createBufferQueue` 的具体实现：

// BufferQueue.cp
void BufferQueue::createBufferQueue(
        sp<IGraphicBufferProducer>* outProducer,
        sp<IGraphicBufferConsumer>* outConsumer,
        bool consumerIsSurfaceFlinger) {
    ...
    sp<BufferQueueCore> core(new BufferQueueCore());
    sp<IGraphicBufferProducer> producer(
        new BufferQueueProducer(core, consumerIsSurfaceFlinger));
    ...
    sp<IGraphicBufferConsumer> consumer(new BufferQueueConsumer(core));
    ...
    *outProducer = producer;
    *outConsumer = consumer;
}

上面代码删掉了 `LOG` 打印及空判断，整个代码流程非常简单：

- 新建 `BufferQueueCore core`
- 由 `core` 新建生产者 `IGraphicBufferProducer`
- 由 `core` 新建消费者 `IGraphicBufferConsumer`

也就是说 `BufferQueueCore` 是最终的纽带，保存了生产者消费者对应的缓存区，连接了两者的关系。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#Surface-1 "Surface")`Surface`

`Surface` 代表着窗口，它包含一个生产者 `IGraphicBufferProducer` 用来填充缓存，也就是窗口中用来显示在屏幕上的内容，`mSlot` 数组表示可以有多个缓存区。

// Surface.h
class Surface
    : public ANativeObjectBase<ANativeWindow, Surface, RefBase>
{
public:
    ...
    sp<IGraphicBufferProducer> getIGraphicBufferProducer() const;
    String8 getConsumerName() const;
    ...
protected:
    ...
    struct BufferSlot {
        sp<GraphicBuffer> buffer;
        Region dirtyRegion;
    };
    sp<IGraphicBufferProducer> mGraphicBufferProducer;
    BufferSlot mSlots[NUM_BUFFER_SLOTS];
    uint32_t mReqWidth;
    uint32_t mReqHeight;
    PixelFormat mReqFormat;
    ...
    // must be used from the lock/unlock thread
    sp<GraphicBuffer>           mLockedBuffer;
    sp<GraphicBuffer>           mPostedBuffer;
    ...
private:
    ...
}

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#ISurfaceComposer-ISurfaceComposerClient "ISurfaceComposer/ISurfaceComposerClient")`ISurfaceComposer/ISurfaceComposerClient`

`ISurfaceComposerClient/ISurfaceComposer` 都继承了 `IInterface` ，它们俩分别代表 `Surface` 合成的客户端和服务端，具体在 `SurfaceFlinger` 服务进程中实现， `libgui` 中只对合成能力（函数）做了定义。

// ISurfaceComposerClient.h
class ISurfaceComposerClient : public IInterface {
public:
    DECLARE_META_INTERFACE(SurfaceComposerClient)
    ...
    virtual status_t createSurface(const String8& name, uint32_t w, 
        uint32_t h, PixelFormat format, uint32_t flags, 
        const sp<IBinder>& parent, uint32_t windowType,
        uint32_t ownerUid, sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp) = 0;
    virtual status_t destroySurface(const sp<IBinder>& handle) = 0;
    virtual status_t clearLayerFrameStats(const sp<IBinder>& handle)
        const = 0;
    virtual status_t getLayerFrameStats(const sp<IBinder>& handle, 
        FrameStats* outStats) const = 0;
};

// ISurfaceComposer.h
class ISurfaceComposer: public IInterface {
public:
    DECLARE_META_INTERFACE(SurfaceComposer)
    ...
    virtual sp<ISurfaceComposerClient> createConnection() = 0;
    virtual sp<ISurfaceComposerClient> createScopedConnection(
            const sp<IGraphicBufferProducer>& parent) = 0;
    virtual sp<IDisplayEventConnection> createDisplayEventConnection(
            VsyncSource vsyncSource = eVsyncSourceApp) = 0;
    virtual sp<IBinder> createDisplay(const String8& displayName,
            bool secure) = 0;
    virtual void destroyDisplay(const sp<IBinder>& display) = 0;
    ...
    virtual status_t captureScreen(const sp<IBinder>& display,
            const sp<IGraphicBufferProducer>& producer,
            Rect sourceCrop, uint32_t reqWidth, uint32_t reqHeight,
            int32_t minLayerZ, int32_t maxLayerZ,
            bool useIdentityTransform,
            Rotation rotation = eRotateNone) = 0;
}

如下是 `ISurfaceComposerClient` 相关的类图结构：

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/01113-android-graphics-display-ISurfaceComposerClient.png)

先看 `ComposerService` ，它代表着 `SurfaceFlinger` 服务端，头文件定义如下：

// ComposerService.h
class ComposerService : public Singleton<ComposerService>
{
    sp<ISurfaceComposer> mComposerService;
    sp<IBinder::DeathRecipient> mDeathObserver;
    Mutex mLock;

    ComposerService();
    void connectLocked();
    void composerServiceDied();
    friend class Singleton<ComposerService>;
public:

    // Get a connection to the Composer Service.  This will block until
    // a connection is established.
    static sp<ISurfaceComposer> getComposerService();
};

查看 `connectLocked, getComposerService` 两个函数的实现：

// SurfaceComposerClient.cpp
void ComposerService::connectLocked() {
    const String16 name("SurfaceFlinger");
    while (getService(name, &mComposerService) != NO_ERROR) {
        usleep(250000);
    }
    assert(mComposerService != NULL);
    ...
}

/*static*/ sp<ISurfaceComposer> ComposerService::getComposerService() {
    ComposerService& instance = ComposerService::getInstance();
    Mutex::Autolock _l(instance.mLock);
    if (instance.mComposerService == NULL) {
        ComposerService::getInstance().connectLocked();
        assert(instance.mComposerService != NULL);
        ALOGD("ComposerService reconnected");
    }
    return instance.mComposerService;
}

`connectLocked` 连接过程就是等待 `SurfaceFlinger` 服务启动后并获取它； `getComposerService` 直接返回已经连接成功的实例 `mComposerService` 。

再看 `SurfaceComposerClient` ，可以将它理解为应用端，是 `SurfaceFlinger` 服务的客户端，它将建立和 `SurfaceFlinger` 服务的通信，头文件定义如下：

// SurfaceComposerClient.h
class SurfaceComposerClient : public RefBase
{
public:
    // Return the connection of this client
    sp<IBinder> connection() const;
    ...
    //! Create a surface
    sp<SurfaceControl> createSurface(
            const String8& name,// name of the surface
            uint32_t w,         // width in pixel
            uint32_t h,         // height in pixel
            PixelFormat format, // pixel-format desired
            uint32_t flags = 0, // usage flags
            SurfaceControl* parent = nullptr, // parent
            uint32_t windowType = 0, 
            uint32_t ownerUid = 0 // UID of the task
    );
    status_t    destroySurface(const sp<IBinder>& id);

    //! Create a virtual display
    static sp<IBinder> createDisplay(const String8& displayName,
        bool secure);
    //! Destroy a virtual display
    static void destroyDisplay(const sp<IBinder>& display);
    ...

private:
    virtual void onFirstRef();
    Composer& getComposer();
    ...
    sp<ISurfaceComposerClient>  mClient;
    Composer&                   mComposer;
    wp<IGraphicBufferProducer>  mParent;
};

`mClient` 实际对应的是 `SurfaceFlinger` 进程中的 `Client.cpp` ，查看 `SurfaceComposerClient::onFirstRef` 源码：

// SurfaceComposerClient.cpp
void SurfaceComposerClient::onFirstRef() {
    sp<ISurfaceComposer> sm(ComposerService::getComposerService());
    if (sm != 0) {
        auto rootProducer = mParent.promote();
        sp<ISurfaceComposerClient> conn;
        conn = (rootProducer != nullptr) ? 
                sm->createScopedConnection(rootProducer) :
                sm->createConnection();
        if (conn != 0) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}

// SurfaceFlinger.cpp
sp<ISurfaceComposerClient> SurfaceFlinger::createScopedConnection(
        const sp<IGraphicBufferProducer>& gbp) {
    if (authenticateSurfaceTexture(gbp) == false) {
        return nullptr;
    }
    const auto& layer = (static_cast<MonitoredProducer*>(
        gbp.get()))->getLayer();
    if (layer == nullptr) {
        return nullptr;
    }

   return initClient(new Client(this, layer));
}

`SurfaceComposerClient` 中有一个重要功能就是创建 `Surface` ，对应源码：

// SurfaceComposerClient.cpp
sp<SurfaceControl> SurfaceComposerClient::createSurface(
        const String8& name,
        uint32_t w,
        uint32_t h,
        PixelFormat format,
        uint32_t flags,
        SurfaceControl* parent,
        uint32_t windowType,
        uint32_t ownerUid)
{
    sp<SurfaceControl> sur;
    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IBinder> parentHandle;
        sp<IGraphicBufferProducer> gbp;

        if (parent != nullptr) {
            parentHandle = parent->getHandle();
        }
        status_t err = mClient->createSurface(name, w, h, format, 
                flags, parentHandle,
                windowType, ownerUid, &handle, &gbp);
        ALOGE_IF(err, "SurfaceComposerClient::createSurface error..");
        if (err == NO_ERROR) {
            sur = new SurfaceControl(this, handle, gbp);
        }
    }
    return sur;
}

最终会调用 `SurfaceFlinger` 中的 `mClient` 来创建 `Layer, IGraphicBufferProducer` ；而具体的 `Surface` 则由 `SurfaceControl` 来创建。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#SurfaceControl "SurfaceControl")`SurfaceControl`

`SurfaceControl` 持有创建的 `Surface` 的强引用，头文件定义：

// SurfaceControl.h
class SurfaceControl : public RefBase
{
    ...
private:
    ...
    sp<SurfaceComposerClient>   mClient;
    sp<IBinder>                 mHandle;
    sp<IGraphicBufferProducer>  mGraphicBufferProducer;
    mutable sp<Surface>         mSurfaceData;
};
}

`mHandle` 指向 `SurfaceFlinger` 创建的 `Layer` 。而 `SurfaceControl::createSurface` 直接 `new` 了一个 `Surface` 对象。

// SurfaceControl.cpp
sp<Surface> SurfaceControl::generateSurfaceLocked() const
{
    // This surface is always consumed by SurfaceFlinger, so the
    // producerControlledByApp value doesn't matter; using false.
    mSurfaceData = new Surface(mGraphicBufferProducer, false);

    return mSurfaceData;
}

sp<Surface> SurfaceControl::createSurface() const
{
    Mutex::Autolock _l(mLock);
    return generateSurfaceLocked();
}

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#IDisplayEventConnection-%E6%98%BE%E7%A4%BA%E8%BF%9E%E6%8E%A5 "IDisplayEventConnection 显示连接")`IDisplayEventConnection` 显示连接

`IDisplayEventConnection` 继承了 `IInterface` ，客户端 `APP` 通过它向服务端 `SurfaceFlinger` 发送刷新请求。

class IDisplayEventConnection : public IInterface {
public:
    DECLARE_META_INTERFACE(DisplayEventConnection)

    /*
     * stealReceiveChannel() returns a BitTube to receive events from. 
     *Only the receive file descriptor of outChannel will be initialized,
     * and this effectively "steals" the receive channel from the remote
     * end (such that the remote end can only use its send channel).
     */
    virtual status_t stealReceiveChannel(gui::BitTube* outChannel) = 0;

    /*
     * setVsyncRate() sets the vsync event delivery rate. 
     * A value of 1 returns every vsync event.
     * A value of 2 returns every other event, etc. 
     * A value of 0 returns no event unless
     * requestNextVsync() has been called.
     */
    virtual status_t setVsyncRate(uint32_t count) = 0;

    /*
     * requestNextVsync() schedules the next vsync event. 
     * It has no effect if the vsync rate is > 0.
     */
    virtual void requestNextVsync() = 0; // Asynchronous
};

`IDisplayEventConnection` 的类图结构

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-create-IDisplayEventConnection.png)

`IDisplayEventConnection` 的具体实现是在 `SurfaceFlinger` 进程中的 `EventThread::Connection` ；`DisplayEventReceiver` 持有该实例。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E5%B0%8F%E7%BB%93-1 "小结")小结

- `libgui` 中一共提供了 4 组 `IInterface` 接口： `IGraphicBufferProducer/IProducerListener, IGraphicBufferConsumer/IConsumerListener, ISurfaceComposerClient/ISurfaceComposer, IDisplayEventConnection`
- `IGraphicBufferProducer/IProducerListener` 生产者模型
- `IGraphicBufferConsumer/IConsumerListener` 消费者模型
- `ISurfaceComposerClient/ISurfaceComposer` 提供创建 `Surface` 的功能，及相关管理
- `IDisplayEventConnection` 提供了 `APP` 客户端请求服务端 `SurfaceFlinger` 刷新的接口

## [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#SurfaceFlinger "SurfaceFlinger")`SurfaceFlinger`

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E4%BB%A3%E7%A0%81%E9%80%9F%E6%9F%A5%E8%A1%A8-1 "代码速查表")代码速查表

surfaceflinger/
├── Android.bp
├── Android.mk
├── Barrier.h
├── Client.cpp
├── Client.h
├── clz.h
├── Colorizer.h
├── DdmConnection.cpp
├── DdmConnection.h
├── DisplayDevice.cpp
├── DisplayDevice.h
├── DisplayHardware
│   ├── ComposerHal.cpp
│   ├── ComposerHal.h
│   ├── DisplaySurface.h
│   ├── FramebufferSurface.cpp
│   ├── FramebufferSurface.h
│   ├── HWC2.cpp
│   ├── HWC2.h
│   ├── HWComposerBufferCache.cpp
│   ├── HWComposerBufferCache.h
│   ├── HWComposer.cpp
│   ├── HWComposer.h
│   ├── HWComposer_hwc1.cpp
│   ├── HWComposer_hwc1.h
│   ├── PowerHAL.cpp
│   ├── PowerHAL.h
│   ├── VirtualDisplaySurface.cpp
│   └── VirtualDisplaySurface.h
├── DisplayUtils.cpp
├── DisplayUtils.h
├── DispSync.cpp
├── DispSync.h
├── Effects
│   ├── Daltonizer.cpp
│   └── Daltonizer.h
├── EventControlThread.cpp
├── EventControlThread.h
├── EventLog
│   ├── EventLog.cpp
│   ├── EventLog.h
│   └── EventLogTags.logtags
├── EventThread.cpp
├── EventThread.h
├── ExSurfaceFlinger
│   ├── ExLayer.cpp
│   ├── ExLayer.h
│   ├── ExSurfaceFlinger.cpp
│   ├── ExSurfaceFlinger.h
│   ├── ExVirtualDisplaySurface.cpp
│   └── ExVirtualDisplaySurface.h
├── FrameTracker.cpp
├── FrameTracker.h
├── GpuService.cpp
├── GpuService.h
├── Layer.cpp
├── LayerDim.cpp
├── LayerDim.h
├── Layer.h
├── LayerRejecter.cpp
├── LayerRejecter.h
├── LayerVector.cpp
├── LayerVector.h
├── main_surfaceflinger.cpp
├── MessageQueue.cpp
├── MessageQueue.h
├── MODULE_LICENSE_APACHE2
├── MonitoredProducer.cpp
├── MonitoredProducer.h
├── RenderEngine
│   ├── Description.cpp
│   ├── Description.h
│   ├── GLES20RenderEngine.cpp
│   ├── GLES20RenderEngine.h
│   ├── GLExtensions.cpp
│   ├── GLExtensions.h
│   ├── Mesh.cpp
│   ├── Mesh.h
│   ├── ProgramCache.cpp
│   ├── ProgramCache.h
│   ├── Program.cpp
│   ├── Program.h
│   ├── RenderEngine.cpp
│   ├── RenderEngine.h
│   ├── Texture.cpp
│   └── Texture.h
├── StartPropertySetThread.cpp
├── StartPropertySetThread.h
├── SurfaceFlingerConsumer.cpp
├── SurfaceFlingerConsumer.h
├── SurfaceFlinger.cpp
├── SurfaceFlinger.h
├── SurfaceFlinger_hwc1.cpp
├── surfaceflinger.rc
├── SurfaceInterceptor.cpp
├── SurfaceInterceptor.h
├── Transform.cpp
└── Transform.h

9 directories, 117 files

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#surfaceflinger-%E8%BF%9B%E7%A8%8B "surfaceflinger 进程")`surfaceflinger` 进程

`SurfaceFlinger` 是以独立进程运行的，进程名为 `surfaceflinger` ，对应的 `rc` 文件如下：

// surfaceflinger.rc
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    onrestart restart zygote
    writepid /dev/stune/foreground/tasks
    socket pdx/system/vr/display/client     stream 0666 system graphics u:object_r:pdx_display_client_endpoint_socket:s0
    socket pdx/system/vr/display/manager    stream 0666 system graphics u:object_r:pdx_display_manager_endpoint_socket:s0
    socket pdx/system/vr/display/vsync      stream 0666 system graphics u:object_r:pdx_display_vsync_endpoint_socket:s0

`surfaceflinger` 服务属于核心类 `core`，当 `surfaceflinger` 重启时会触发 `zygote` 的重启。接下来看进程 `main` 方法对应文件为：

// main_surfaceflinger.cpp
int main(int, char**) {
    // 启动 IAllocator, DisplayService 两个服务
    startHidlServices();

    signal(SIGPIPE, SIG_IGN);

    ProcessState::self()->setThreadPoolMaxThreadCount(4);
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();

    // 1. new SurfaceFlinger 实例
    sp<SurfaceFlinger> flinger = 
        DisplayUtils::getInstance()->getSFInstance();

    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
    set_sched_policy(0, SP_FOREGROUND);
    if (cpusets_enabled()) set_cpuset_policy(0, SP_SYSTEM);

    // 2. SurfaceFlinger 初始化
    flinger->init();

    // 3. 发布 SurfaceFlinger, GpuService 两个服务
    // publish surface flinger
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), 
        flinger, false);

    // publish GpuService
    sp<GpuService> gpuservice = new GpuService();
    sm->addService(String16(GpuService::SERVICE_NAME),gpuservice,false);

    struct sched_param param = {0};
    param.sched_priority = 2;
    if (sched_setscheduler(0, SCHED_FIFO, ¶m) != 0) {
        ALOGE("Couldn't set SCHED_FIFO");
    }

    // run surface flinger in this thread
    // 4. SurfaceFlinger 无限循环等待事件
    flinger->run();

    return 0;
}

`surfaceflinger` 进程的 `main` 函数中主要做了 4 件事：

- 通过 `DisplayUtils` 创建 `SurfaceFlinger` 对象
- `SurfaceFlinger` 对象调用 `init` 方法，实现初始化
- `SurfaceFlinger` 向系统注册 `Binder` 服务
- `SurfaceFlinger` 对象调用 `run` 方法，该方法是一个 `do-while` 无限循环

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B "初始化流程")初始化流程

[SurfaceFlinger 初始化流程，查看大图](https://upload-images.jianshu.io/upload_images/606437-92984e2a12a8b3d0.png)

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-SurfaceFlinger-init.png)

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#DisplayDevice-%E6%98%BE%E7%A4%BA%E8%AE%BE%E5%A4%87 "DisplayDevice 显示设备")`DisplayDevice` 显示设备

// DisplayDevice.h
class DisplayDevice : public LightRefBase<DisplayDevice>
{
public:
    ...
    enum DisplayType {
        DISPLAY_ID_INVALID = -1,
        DISPLAY_PRIMARY     = HWC_DISPLAY_PRIMARY,      // 主显
        DISPLAY_EXTERNAL    = HWC_DISPLAY_EXTERNAL,     // 外显
        DISPLAY_VIRTUAL     = HWC_DISPLAY_VIRTUAL,      // 虚显
        NUM_BUILTIN_DISPLAY_TYPES = HWC_NUM_PHYSICAL_DISPLAY_TYPES,
    };
    ...
}

显示设备有三种类型：主显、外显、虚显；每添加一个显示屏，都会创建一个 `DisplayDevice` 。

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#Layer-%E5%B1%82 "Layer 层")`Layer` 层

`Layer` 是 `SurfaceFlinger` 进行合成的基本操作单元。`Layer` 在应用请求创建 `Surface` 的时候在 `SurfaceFlinger` 内部创建，因此一个 `Surface` 对应一个 `Layer` 。每个 `Layer` 包含常见属性：

- `Z order`
- `Alpha value from 0 to 255`
- `visibleRegion`
- `crop region`
- `transformation: rotate 0, 90, 180, 270: flip H, V: scale`

当多个 `Layer` 进行合成的时候，并不是整个 `Layer` 的空间都会被完全显示，根据这个 `Layer` 最终的显示效果，一个 `Layer` 可以被划分成很多的 `Region` ， 在 `SurfaceFlinger` 中定义了以下几种类型：

- `TransparantRegion` ：完全透明的区域，在它之下的区域将被显示出来
- `OpaqueRegion` ：完全不透明的区域，是否显示取决于它上面是否有遮挡或是否透明
- `VisibleRegion` ：可见区域，包括完全不透明无遮挡区域或半透明区域；即 `visibleRegion = Region - above OpaqueRegion.`
- `CoveredRegion` ：被遮挡区域，在它之上，有不透明或半透明区域
- `DirtyRegion` ：可见部分改变区域，包括新的被遮挡区域，和新的露出区域

头文件定义：

// Layer.h
class Layer : public SurfaceFlingerConsumer::ContentsChangedListener {
public:
    ...
    // regions below are in window-manager space
    Region visibleRegion;
    Region coveredRegion;
    Region visibleNonTransparentRegion;
    Region surfaceDamageRegion;

    // Layer serial number.  This gives layers an explicit ordering, so we
    // have a stable sort order when their layer stack and Z-order are
    // the same.
    int32_t sequence;
    ...
private:
    ...
    // constants
    sp<SurfaceFlingerConsumer> mSurfaceFlingerConsumer;
    sp<IGraphicBufferProducer> mProducer;
    uint32_t mTextureName;      // from GLES
    bool mPremultipliedAlpha;
    String8 mName;
    String8 mTransactionName; // A cached version of "TX - " + mName for systraces
    PixelFormat mFormat;
    ...
    FenceTimeline mAcquireTimeline;
    FenceTimeline mReleaseTimeline;
    ...
    // The mesh used to draw the layer in GLES composition mode
    mutable Mesh mMesh;
    // The texture used to draw the layer in GLES composition mode
    mutable Texture mTexture;
    Vector<BufferItem> mQueueItems;
    // Child list about to be committed/used for editing.
    LayerVector mCurrentChildren;
    // Child list used for rendering.
    LayerVector mDrawingChildren;

    wp<Layer> mCurrentParent;
    wp<Layer> mDrawingParent;
}

`SurfaceFlinger` 接收所有 `Surface` 作为输入，根据 `Z-Order`， 透明度，大小，位置等参数，计算出每个 `Surface` 在最终合成图像中的位置，然后交由 `HWComposer, OpenGL` 生成最终的显示 `Buffer` , 然后显示到特定的显示设备上。  
`Layer` 的合成分为两种，离线合成和在线合成：

- 离线合成  
    先将所有图层画到一个最终层 `FrameBuffer` 上，再将 `FrameBuffer` 送到 `LCD` 显示。由于合成 `FrameBuffer` 与送 `LCD` 显示一般是异步的（线下生成 `FrameBuffer` ，需要时线上的 `LCD` 去取），因此叫离线合成。
- 在线合成  
    不使用 `FrameBuffer` ，在 `LCD` 需要显示某一行的像素时，用显示控制器将所有图层与该行相关的数据取出，合成一行像素送过去。只有一个图层时，又叫 `Overlay` 技术。  
    由于省去合成 `FrameBuffer` 时读图层，写 `FrameBuffer` 的步骤，大幅降低了内存传输量，减少了功耗，但这个需要硬件支持。

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-layer-composer.jpg)

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#Surface-%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B%E5%9B%BE "Surface 创建流程图")`Surface` 创建流程图

由客户端 `SurfaceComposerClient` 发起创建流程，然后由服务端 `SurfaceFlinger` 创建对应的 `Layer` ，而 `Layer` 在被引用时会创建生产者消费者模型的 `BufferQueue` ，然后再由客户端将拿到的结果传入 `SurfaceControl` ，最后直接实例化一个 `Surface` 。

创建生产者消费者模型 `BufferQueue` 的关键代码：

// Layer.cpp
void Layer::onFirstRef() {
    // Creates a custom BufferQueue for SurfaceFlingerConsumer to use
    sp<IGraphicBufferProducer> producer;
    sp<IGraphicBufferConsumer> consumer;
    BufferQueue::createBufferQueue(&producer, &consumer, true);
    mProducer = new MonitoredProducer(producer, mFlinger, this);
    mSurfaceFlingerConsumer = 
        new SurfaceFlingerConsumer(consumer, mTextureName, this);
    mSurfaceFlingerConsumer->setConsumerUsageBits(getEffectiveUsage(0));
    mSurfaceFlingerConsumer->setContentsChangedListener(this);
    mSurfaceFlingerConsumer->setName(mName);

    if (mFlinger->isLayerTripleBufferingDisabled()) {
        mProducer->setMaxDequeuedBufferCount(2);
    }

    const sp<const DisplayDevice> hw(mFlinger->getDefaultDisplayDevice());
    updateTransformHint(hw);
}

[创建 Surface 流程，查看大图](https://upload-images.jianshu.io/upload_images/606437-645921566ffd77c6.png)

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-create-surface.png)

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#VSYNC-%E5%9E%82%E7%9B%B4%E5%88%B7%E6%96%B0-1 "VSYNC 垂直刷新")`VSYNC` 垂直刷新

`Android` 中有 2 种 `VSync` 信号：屏幕产生的硬件 `VSync` 和由 `SurfaceFlinger` 将其转成的软件 `Vsync` 信号；软件 `Vsync` 后者经由 `Binder` 传递给 `Choreographer` 。  
`Vsync` 信号可将某些事件同步到显示设备的刷新周期。应用总是在 `VSYNC` 边界上开始绘制，而 `SurfaceFlinger` 总是在 `VSYNC` 边界上进行合成。这样可以消除卡顿，并提升图形的视觉表现。 `HWComposer` 对象创建过程，会注册一些回调方法；当硬件产生 `VSYNC` 信号时，则会回调 `HWC2::ComposerCallbackBridge::onVsync` 方法，然后逐级回调，下图是整个回调流程图：

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-VsyncCallback.png)

- 硬件 `Vsync` 信号发送过来，一路执行到 `DispSyncThread.updateModel` 方法中调用 `mCond.signal` ，唤醒 `DispSyncThread` 线程
- `DispSyncThread` 线程中执行 `EventThread::onVSyncEvent` 中调用 `mCondition.broadcast` 唤醒 `EventThread` 线程
- `EventThread` 线程中执行 `DisplayEventReceiver::sendEvents` 方法，会调用 `BitTube::sendObjects` ；在 `MessageQueue::setEventThread` 中，我们设置了 `BitTube` 事件的回调，当收到数据会触发 `MQ.cb_eventReceiver` ；根据 `Handler` 消息机制，进入 `SurfaceFlinger` 主线程
- `SurfaceFlinger` 主线程进入到 `MesageQueue的handleMessage` ，最终调用 `SurfaceFlinger::handleMessageRefresh`

### [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E5%AE%A2%E6%88%B7%E7%AB%AF%E9%80%9A%E7%9F%A5-SurfaceFlinger-%E5%88%B7%E6%96%B0 "客户端通知 SurfaceFlinger 刷新")客户端通知 `SurfaceFlinger` 刷新

`BufferQueueProducer::queueBuffer` 函数中会调用 `listener->onFrameAvailable` ，而这最终会触发服务端的 `Layer::onFrameAvailable` ，从而通知 `SurfaceFlinger` 合成图像。我们先看 `onFrameAvailable` 接口的继承关系：

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-onFrameAvailable-class-uml.png)

从 `Surface` 创建流程中贴出的 `Layer::onFirstRef` 代码中可以看到，在 `Layer` 中设置了 `mSurfaceFlingerConsumer->setContentsChangedListener` 监听事件，所以 `BufferQueueProducer::queueBuffer` 会触发 `Layer::onFrameAvailable` 事件，下面是完整的请求流程。  
客户端在 `BufferQueue` 中生产完图像数据后，通知 `SurfaceFlinger` 刷新界面的流程图：

[客户端通知 SurfaceFlinger 刷新，查看大图](https://upload-images.jianshu.io/upload_images/606437-4f8a5a419984488c.png)

![](https://raw.githubusercontent.com/redspider110/blog-images/master/_images/0113-android-graphics-display-requestNextVsync.png)

## [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E5%90%8E%E7%BB%AD "后续")后续

- `WMS`
- `Layer` 合成流程
- 结合 `Camera` 熟悉图形显示中 `Buffer` 相关流程
- `Choreographer` 及掉帧分析
    - [Choreographer 简析](https://www.jianshu.com/p/dd32ec35db1d)
- 详述 `SurfaceTexture, SurfaceView, GLSurfaceView` 等区别
- 工具 `dumpsys SurfaceFlinger` 输出的 `LOG` 分析
- `screencap` 命令及源码分析
- `Systrace` 性能工具分析
    - [Systrace 官网](https://source.android.google.cn/devices/tech/debug/systrace)

## [](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#%E5%8F%82%E8%80%83%E6%96%87%E6%A1%A3 "参考文档")参考文档

- [Android 图形显示 - 官网](https://source.android.google.cn/devices/graphics)
- [Android 中 native_handle private_handle_t 的关系](https://blog.csdn.net/ear5cm/article/details/45458683)
- **[夕阳风 - 图形显示系列文章](https://www.jianshu.com/u/f92447ae8445)**
- **[SurfaceFlinger 系列](http://windrunnerlihuan.com/2017/04/27/Android-SurfaceFlinger-%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-%E4%BA%8C-SurfaceFlinger%E6%A6%82%E8%BF%B0/)**
- [理解 VSYNC](https://blog.csdn.net/zhaizu/article/details/51882768)
- [dp, dpi, px, density 的关系](http://www.cnblogs.com/yaozhongxiao/archive/2014/07/14/3842908.html)
- [gityuan: SurfaceFlinger 相关](http://gityuan.com/2017/02/18/surface_flinger_2/)
- [阿拉神农：深入理解 Surface 系统](https://blog.csdn.net/innost/article/details/47208337)
- [图解 Surface, SurfaceFlinger 关系](https://blog.csdn.net/freekiteyu/article/details/79483406)
- [Google Grafika](https://github.com/google/grafika)
- [图形引擎的核心 - BufferQueue](https://blog.csdn.net/u013928208/article/details/82999075)
- [surfaceflinger 框架 - 流程图大全](https://blog.csdn.net/xisuzun7960/article/details/81212721)
- [BufferQueue 介绍](https://blog.csdn.net/armwind/article/details/73436532)
- [GraphicBuffer 同步机制 - Fence](https://blog.csdn.net/jinzhuojun/article/details/39698317/)

全文完

本文由 [简悦 SimpRead](http://ksria.com/simpread) 优化，用以提升阅读体验

使用了 全新的简悦词法分析引擎 beta，[点击查看](http://ksria.com/simpread/docs/#/%E8%AF%8D%E6%B3%95%E5%88%86%E6%9E%90%E5%BC%95%E6%93%8E)详细说明

[概念](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-0)[EGL](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-1)[Surface 和 SurfaceFlinger](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-2)[WMS: WindowManagerServices](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-3)[FrameBuffer](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-4)[Gralloc](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-5)[HWC](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-6)[VSYNC 垂直刷新](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-7)[60Hz 和 16 ms](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-8)[BufferQueue](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-9)[数据流](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-10)[组件小结](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-11)[Buffer/Window 体系](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-12)[代码速查表](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-13)[native_handle/buffer_handle_t](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-14)[private_handle_t](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-15)[ANativeWindowBuffer](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-16)[ANativeWindow](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-17)[ANativeObjectBase 模板](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-18)[GraphicBuffer](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-19)[Surface](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-20)[小结](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-21)[libui 库](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-22)[代码目录结构](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-23)[GraphicBufferAllocator/GraphicBufferMapper](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-24)[GraphicBuffer](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-25)[Fence 机制](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-26)[DisplayInfo 显示信息](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-27)[libgui 库](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-28)[代码目录结构](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-29)[IGraphicBufferProducer/IProducerListener 生产者](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-30)[IGraphicBufferConsumer/IConsumerListener 消费者](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-31)[BufferItem](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-32)[BufferSlot](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-33)[BufferQueueCore](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-34)[BufferQueueProducer/BufferQueueConsumer 生产者 / 消费者实现类](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-35)[BufferQueue 模型](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-36)[Surface](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-37)[ISurfaceComposer/ISurfaceComposerClient](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-38)[SurfaceControl](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-39)[IDisplayEventConnection 显示连接](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-40)[小结](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-41)[SurfaceFlinger](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-42)[代码速查表](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-43)[surfaceflinger 进程](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-44)[初始化流程](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-45)[DisplayDevice 显示设备](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-46)[Layer 层](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-47)[Surface 创建流程图](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-48)[VSYNC 垂直刷新](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-49)[客户端通知 SurfaceFlinger 刷新](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-50)[后续](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-51)[参考文档](https://redspider110.github.io/2019/04/17/0113-android-graphics-display/#sr-toc-52)