## GLSurfaceView 简介

### Surface 与 SurfaceFlinger

在Android系统中，无论开发者调用什么渲染API，一切内容都会渲染到Surface上。在Android平台上创建的每个窗口都由Surface提供支持。 用户通过在Surface这张画布上绘制生成图形数据，这些数据会被BufferQueue传输给SurfaceFlinger，所有被渲染的可见Surface都被SurfaceFlinger合成到屏幕。Surface表示缓冲区队列中的生产者，而缓冲区队列会被消费者SurfaceFlinger消耗掉。

Android 通过其框架 API 和原生开发套件 (NDK) 来支持 OpenGL。Android 框架中有如下两个基本类，用于通过 OpenGL ES API 来创建和操控图形：`GLSurfaceView` 和 `GLSurfaceView.Renderer`。也就是说我们想在Android中使用OpenGL ES的最简单的方法就是将我们要显示的图像渲染到 `GLSurfaceView` 上，但它只是一个展示类，至于怎么渲染到上面我们就要自己去实现 `GLSurfaceView.Renderer` 来生成我们自己的渲染器从而完成渲染操作。

### GLSurfaceView

SurfaceView 在 View 的基础上创建了独立的 Surface，拥有 SurfaceHolder 来管理它的 Surface，渲染的工作可以不在主线程中做。可以通过 SurfaceHolder 得到 Canvas，在单独的线程中，利用 Canvas 绘制需要显示的内容，然后更新到 Surface 上。

GLSurfaceView 继承自 SurfaceView，它主要是在 SurfaceView 的基础上加入了 EGL 的管理，并自带了一个 GLThread 的绘制线程，绘制的工作直接通过 OpenGL 在绘制线程进行，不会堵塞主线程，绘制的结果输出到 SurfaceView 所提供的 Surface 上，这使得 GLSurfaceView 也拥有了 OpenGLES 所提供的图形处理能力，通过它定义的 Render 接口，使更改具体的 Render 的行为非常灵活，只需要将实现了渲染函数的 Renderer 的实现类设置给 GLSurfaceView 既可。也就是说它是对 SurfaceView 的再次封装，为了方便我们在安卓中使用 OpenGL。

#### GLSurfaceView 常用方法
- setEGLContextClientVersion: 设置 OpenGL ES 的版本，3.0 则设置 3
- onPause: 暂停渲染，最好在 Activity、Fragment 的 onPause 方法内调用，减少不必要的性能开销，避免不必要的崩溃
- onResume: 恢复渲染
- setRender: 设置渲染器
- setRenderMode: 设置渲染模式，有连续渲染和按需渲染两种模式，按需渲染需要调用下面的 requestRender 方法才会去渲染
- requestRender: 请求渲染，请求异步线程进行渲染，调用后不会立即进行渲染。渲染会回调到 Renderer 接口的 onDrawFrame() 方法
- queueEvent: 插入一个 Runnable 任务到后台渲染线程上执行。相应的，渲染线程中可以通过 Activity 的 runOnUIThread 的方法来传递事件给主线程去执行

#### GLSurfaceView.Renderer

此接口定义了在 GLSurfaceView 中绘制图形所需的方法。您必须将此接口的一个实现作为单独的类提供，并使用 GLSurfaceView.setRenderer() 将其附加到您的 GLSurfaceView 实例。

GLSurfaceView.Renderer 接口要求您实现以下方法:
- onSurfaceCreated(): 系统会在创建 GLSurface 时调用一次此方法。使用此方法可执行仅需发生一次的操作，例如设置 OpenGL 环境参数或初始化 OpenGL 图形对象。
- onDrawFrame(): 完成绘制工作，每一帧图像的渲染都要在这里完成，系统会在每次重新绘制 GLSurfaceView 时调用此方法。请将此方法作为绘制（和重新绘制）图形对象的主要执行点。
- onSurfaceChanged(): 系统会在 GLSurfaceView 几何图形发生变化（包括 GLSurfaceView 大小发生变化或设备屏幕方向发生变化）时调用此方法。例如，系统会在设备屏幕方向由纵向变为横向时调用此方法。使用此方法可响应 GLSurfaceView 容器中的更改。

使用 GLSurfaceView 和 GLSurfaceView.Renderer 为 OpenGL ES 建立容器视图后，您便可以开始使用 `android.opengl` 调用 OpenGL API。此软件包提供了 OpenGL ES 3.0/3.1 类的接口。
    - `GLES30`
    - `GLES31`
    - `GLES31Ext` ([Android Extension Pack](https://developer.android.google.cn/guide/topics/graphics/opengl#aep))

### GLSurfaceView 应用

```
package android.opengl;
/**
 * An implementation of SurfaceView that uses the dedicated surface for
 * displaying OpenGL rendering.
 */
 public class GLSurfaceView extends SurfaceView implements SurfaceHolder.Callback2 {
 }
```

GPU 加速：GLSurfaceView 的效率是 SurfaceView 的 30 倍以上，SurfaceView 使用画布 Canvas 进行绘制，GLSurfaceView 利用 GPU 加速提高了绘制效率。View 的绘制 onDraw(Canvas canvas) 使用 Skia 渲染引擎渲染，而 GLSurfaceView 的渲染器 Renderer 的 onDrawFrame(GL10 gl) 使用 OpenGL 绘制引擎进行渲染。

GLSurfaceView 的特性:
- 管理一个 surface，这个 surface 就是一块特殊的内存，能直接排版到 android 的视图 view 上。
- 管理一个 EGL Display，它能让 OpenGL 把内容渲染到 surface 上。
- 用户自定义渲染器 Renderer。
- 让渲染器在独立的变成里运作 (与 UI 线程分离)。
- 支持按需渲染 (on-demand) 和连续渲染(continuous)。
- 可以封装、跟踪并且排查渲染器的问题。

#### OpenGL 实现一个绿色的 activity

这里是全屏渲染成绿色，所以不会涉及到顶点着色器和片段着色器。首选现在 AndroidManifest.xml 文件中声明使用 OpenGL ES 的版本：
<uses-feature android:glEsVersion="0x00030000" android:required="true" />
```xml
<uses-feature android:glEsVersion="0x00030000" android:required="true" />
```

```Java
public class MainActivity extends AppCompatActivity {
    private GLSurfaceView mGlSurfaceView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mGlSurfaceView = new MyGLSurfaceView(this);
        setContentView(mGlSurfaceView);
    }
}

class MyGLSurfaceView extends GLSurfaceView {
    private final MyGLRenderer renderer;
    public MyGLSurfaceView(Context context) {
        super(context);
        // 设置opengl es的版本，现在应该没有app还要支持4.3一下系统了，直接用3.0就可以
        setEGLContextClientVersion(3);
        renderer = new MyGLRenderer();
        setRenderer(renderer);
    }
}

class MyGLRenderer implements GLSurfaceView.Renderer {
    public void onSurfaceCreated(GL10 unused, EGLConfig config) {
        // Set the background frame color RGBA
        GLES30.glClearColor(0.0f, 1.0f, 0.0f, 0.0f);
    }
    public void onDrawFrame(GL10 unused) {
        // Redraw background color
        // glClearColor设置好清除颜色，glClear利用glClearColor设置好的清除颜色来设置颜色缓冲区的颜色
        GLES30.glClear(GLES31.GL_COLOR_BUFFER_BIT);
    }

    public void onSurfaceChanged(GL10 unused, int width, int height) {
        GLES30.glViewport(0, 0, width, height);
    }
}
```

#### RenderMode

GLSurfaceView 默认采用的是 `RENDERMODE_CONTINUOUSLY` 连续渲染的方式，刷新的帧率是 60FPS，16ms 就重绘一次，可以通过 mGLView.setRenderMode() 更改其渲染方式为 `RENDERMODE_WHEN_DIRTY`，表示按需渲染，在 surfaceCreate 的时候会绘制一次，之后只有在调用 requestRender 或者 onResume 等方法主动请求重绘时才会进行渲染，如果界面不需要频繁的刷新最好使用 `RENDERMODE_WHEN_DIRTY`，这样可以降低 CPU 和 GPU 的活动。

#### 状态处理

使用 GLSurfaceView 需要注意程序的生命周期，Activity 及 Fragment 会有暂停和恢复等状态，GLSurfaceView 也需根据这些状态来做相应的处理，GLSurfaceView 具有 onResume 和 onPause 两个同 Activity 及 Fragment 中的生命周期同名的方法，在 Activity 或者 Fragment 中的 onResume 和 onPause 方法中，需要主动调用 GLSurfaceView 的实例的这两个方法，这样能使 OpenGLES 的内部线程做出正确的判断，从而保证应用程序的稳定性。

#### GLSurfaceView 的事件处理

为了处理事件，需要继承 GLSurfaceView 类并重载它的事件方法，但是由于 GLSurfaceView 是多线程的，渲染器在独立的渲染线程中，需要使用 Java 的跨线程机制与渲染器进行通讯，GLSurfaceView 提供了 queueEvent(Runnable runnable) 方法就是一种相对简单的操作，queueEvent() 方法被安全的用于在 UI 线程和渲染线程之间进行交流通信。这块在注释里面写了:

```
 * To handle an event you will typically subclass GLSurfaceView and override the
 * appropriate method, just as you would with any other View. However, when handling
 * the event, you may need to communicate with the Renderer object
 * that's running in the rendering thread. You can do this using any
 * standard Java cross-thread communication mechanism. In addition,
 * one relatively easy way to communicate with your renderer is
 * to call
 * {@link #queueEvent(Runnable)}. For example:
 * <pre class="prettyprint">
 * class MyGLSurfaceView extends GLSurfaceView {
 *
 *     private MyRenderer mMyRenderer;
 *
 *     public void start() {
 *         mMyRenderer = ...;
 *         setRenderer(mMyRenderer);
 *     }
 *
 *     public boolean onKeyDown(int keyCode, KeyEvent event) {
 *         if (keyCode == KeyEvent.KEYCODE_DPAD_CENTER) {
 *             queueEvent(new Runnable() {
 *                 // This method will be called on the rendering
 *                 // thread:
 *                 public void run() {
 *                     mMyRenderer.handleDpadCenter();
 *                 }});
 *             return true;
 *         }
 *         return super.onKeyDown(keyCode, event);
 *     }
 *}
```

### SurfaceTexture

说到 GLSurfaceView 就一定要提一下 SurfaceTexture。

和 SurfaceView 功能类似，区别是 SurfaceTexure 可以不显示在界面中。使用 OpenGl 对图片流进行美化，添加水印，滤镜这些操作的时候我们都是通过 SurfaceTexre 去处理，处理完之后再通过 GlSurfaceView 显示。

缺点是可能会导致个别帧的延迟。本身管理着 BufferQueue, 所以内存消耗会多一点。

SurfaceTexture 从图像流（来自 Camera 预览，视频解码，GL 绘制场景等）中获得帧数据，当调用 updateTexImage() 时，根据内容流中最近的图像更新 SurfaceTexture 对应的 GL 纹理对象，接下来，就可以像操作普通 GL 纹理一样操作它了。

SurfaceTexture 可以将 Surface 中最近的图像数据更新到 GL Texture 中。通过 GL Texture 我们就可以拿到视频帧，然后直接渲染到 GLSurfaceView 中。

通过 **setOnFrameAvailableListener(listener)** 可以向 SurfaceTexture 注册监听事件，当 Surface 有新的图像可用时，调用 SurfaceTexture 的 **updateTexImage()** 方法将图像内容更新到 GL Texture 中，然后做绘制操作。

SurfaceTexture 中的 attachToGLContext() 和 detachToGLContext() 可以让多个 GL context 共享同一个内容源。

SurfaceTexture 对象可以在任何线程上创建。

updateTexImage() 只能在包含纹理对象的 OpenGL ES 上下文的线程上调用。 在任意线程上调用 frame-available 回调函数，不与 updateTexImage() 在同一线程上出现。


## GLSurfaceView 源码分析

https://github.com/CharonChui/AndroidNote/blob/master/VideoDevelopment/OpenGL/3.GLSurfaceView%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md

