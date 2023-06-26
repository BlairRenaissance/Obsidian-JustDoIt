
### Unity main thread

Unity’s main thread executes all scripts and usually has a high CPU load. 

### Unity JobWorker threads

Unity’s worker threads execute jobs from both the core engine and those your application dispatches via the [C# Job System](https://docs.unity3d.com/cn/2021.2/Manual/JobSystem.html). 自链: [[Job System]]

### Unity render thread

Unity’s render thread interacts with the Graphics API if your project uses [multithreaded rendering](https://docs.unity3d.com/cn/2021.2/ScriptReference/Rendering.RenderingThreadingMode.MultiThreaded.html).

**Note**: If you use GraphicsJobs, JobWorker threads also interact with the Graphics API.