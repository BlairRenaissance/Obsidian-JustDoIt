## EGL简介

在OpenGL的设计中，OpenGL是不负责管理窗口的，窗口的管理交由各个设备自己完成。在Android平台上使用EGL提供本地平台对OpenGL的实现。OpenGL通过GPU进行渲染，但是我们的程序是运行在CPU上，要与GPU关联，这就需要通过EGL，它相当于Android上层应用于GPU通讯的中间层。

GLSurfaceView 主要就是在SurfaceView的基础上加入了EGL的管理，并自带了一个GLThread的绘制线程。

EGL提供了OpenGL ES与运行于计算机上的原生窗口系统(如Windows、Mac OSX)之间的一个“结合”层次。因为每个窗口系统都有不同的语言，所以EGL提供基本的不透明类型 EGLDisplay，该类型封装了所有系统相关性，用于和原生窗口系统接口。==任何使用EGL的应用程序前必须执行的第一个操作就是创建和初始化与本地EGL显示的连接。==

EGL是OpenGL和本地窗口系统（Native Window System）之间的通信接口，它的主要作用：
- 与设备的原生窗口系统通信；
- 查询绘图表面的可用类型和配置；
- 创建绘图表面；
- 在OpenGL ES 和其他图形渲染API之间同步渲染；
- 管理纹理贴图等渲染资源。 OpenGL ES 的平台无关性正是借助 EGL 实现的，EGL 屏蔽了不同平台的差异。

![image](https://raw.githubusercontent.com/CharonChui/Pictures/master/egl_interface_1.png?raw=true)

上图中：
- Display(EGLDisplay) 是对实际显示设备的抽象；
- Surface（EGLSurface）是对用来存储图像的内存区域 FrameBuffer 的抽象，包括 Color Buffer，Stencil Buffer，Depth Buffer；
- Context (EGLContext) 存储 OpenGL ES 绘图的一些状态信息； 在 Android 平台上开发 OpenGL ES 应用时，类 GLSurfaceView 已经为我们提供了对 Display , Surface , Context 的管理，即 **GLSurfaceView 内部实现了对 EGL 的封装**，可以很方便地利用接口 GLSurfaceView.Renderer 的实现，使用 OpenGL ES API 进行渲染绘制，很大程度上提升了 OpenGLES 开发的便利性。

本地窗口相关的 API 提供了访问本地窗口系统的接口，而 EGL 可以创建渲染表面 EGLSurface ，同时提供了图形渲染上下文 EGLContext，用来进行状态管理，接下来 OpenGL ES 就可以在这个渲染表面上绘制。

EGL为双缓冲工作模式(Double Buffer)，既有一个Back Frame Buffer和一个Front Frame Buffer，正常绘制的目标都是Back Frame Buffer，绘制完成后再调用 `eglSwapBuffer` API，将绘制完毕的FrameBuffer交换到Front Frame Buffer并显示出来。

补充说明： 应用程序使用单缓冲绘图时可能会存在图像闪烁的问题。这是因为生成的图像不是一下子被绘制出来的，而是按照从左到右，由上到下逐像素的绘制而成的。最终图像不是在瞬间显示给用户，而是通过一步一步生成的，这会导致渲染的结果很不真实。为了规避这些问题，就应用了双缓冲渲染模式，前缓冲保存着最终输出的图像，它会在屏幕上显示；  
而所有的渲染指令都会在后缓冲上绘制。当所有的渲染指令执行完毕后，我们交换前缓冲和后缓冲，这样图像就立即呈现出来，刚才提到的不真实感就消除了。

## EGL使用流程

要在Android平台实现OpenGL渲染，需要完成一系列的EGL操作，主要为下面几步(后面分析GLSurfaceView源码的时候也是这样来实现的):

1. 获取显示设备(EGL Display)
    获取将要用于显示的设备，有些系统具有多个显示器，会存在多个display，在Android上通过调用EGL10的 `eglGetDisplay(Object native_display)` 方法获得EGLDisplay对象，通常传入的参数为`EGL10.EGL_DEFAULT_DISPLAY`。
    
2. 初始化EGL
    调用EGL10的 `egInitialize(EGLDisplay display, int[] major_minor)` 方法完成初始化操作。display参数即为上一步获取的对象，major_minor传入的是一个int数据，通常传入的是一个大小为2的数据。
    
3. 选择Config配置
    调用EGL10的 `eglChooseConfig(EGLDisplay display, int[] attire_list, EGLConfig[] configs, int config_size, int[] num_config)` 方法，参数3用于存放输出的configs，参数4指定最多输出多少个config，参数5由EGL系统写入，表明满足attributes的config一共有多少个。
    
4. 创建EGL Context
    `eglCreateContext(EGLDisplay display, EGLConfig config, EGLContext share_context, int[] attrib_list);` 参数1即为上面获取的Display，参数2为上一步chooseConfig传入的configs，share_context 是否有context共享，共享的context之间亦共享所有数据，通常设置为EGL_NO_CONTEXT代表不共享。attrib_list 为int数组 {EGL_CONTEXT_CLIENT_VERSION, 2,EGL10.EGL_NONE }; 中间的2代表的是OpenGL ES的版本。
    
5. 创建EGLSurface
    `eglCreateWindowSurface(EGLDisplay display, EGLConfig config, Object native_window, int[] attrib_list);` 参数1、2均为上述步骤得到的结果，参数3为上层创建的用于绘制内容的surface对象，参数4常设置为null。
    
6. 设置OpenGL的渲染环境
    `eglMakeCurrent(EGLDisplay display, EGLSurface draw, EGLSurface read, EGLContext context);` 该方法的参数意义很明确，该方法在异步线程中被调用，该线程也会被成为GL线程，一旦设定后，所有OpenGL渲染相关的操作都必须放在该线程中执行。

通过上述操作，就完成了EGL的初始化设置，便可以进行OpenGL的渲染操作。所有EGL命令都是以 `egl` 前缀开始，对组成命令名的每个单词使用首字母大写(如 `eglCreateWindowSurface`)可以参考Khronos Group的EGL官方文档：[https://www.khronos.org/registry/EGL/](https://www.khronos.org/registry/EGL/) 。