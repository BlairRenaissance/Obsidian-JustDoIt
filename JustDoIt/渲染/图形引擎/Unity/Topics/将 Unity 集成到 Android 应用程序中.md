说明如何使用“Unity 作为库引入”功能将 Unity 运行时库集成到 Android 应用程序中。
[将 Unity 集成到 Android 应用程序中](https://docs.unity3d.com/cn/2021.2/Manual/UnityasaLibrary-Android.html)
[Using Unity as a library in native iOS/Android apps](https://forum.unity.com/threads/using-unity-as-a-library-in-native-ios-android-apps.685195/)


## Gralde结构

从Unity生成导出的每个 [Android Gradle 项目](https://docs.unity3d.com/cn/2021.2/Manual/android-gradle-overview.html)都具有以下结构：
- **unityLibrary** 模块中的一个库部分，可以集成到其他任何 Gradle 项目中。这包含 Unity 运行时和播放器数据。
- **launcher** 模块中的瘦启动器部分，其中包含应用程序的名称及其图标。这是一个可启动 Unity 的简单 Android 应用程序。您可以将此模块替换为自己的应用程序。

![](https://forum.unity.com/attachments/gradleoldnew-png.426991/)


## Gradle模版

Gradle templates configure how to build an Android application using Gradle. Each Gradle template represents a single Gradle project. Gradle projects can include, and depend on, other Gradle projects. [Gradle模板](https://docs.unity3d.com/cn/2021.2/Manual/gradle-templates.html)

![image.png](https://pic2.58cdn.com.cn/nowater/webim/big/n_v27eb639b0f95d4d668ba1f3dda89fe043.png)

## 集成到Android

要将 Unity 集成到另一个 Android Gradle 项目中，必须通过 _settings.gradle_ 文件将生成的 Android Gradle 项目的 **unityLibrary** 模块包含在您的 Android Unity 项目中。此[论坛](https://forum.unity.com/threads/integration-unity-as-a-library-in-native-android-app-version-2.751712/)（推荐）或[代码仓库](https://github.com/Unity-Technologies/uaal-example/blob/master/docs/android.md)演示了如何将 Unity 集成到 Android 应用程序中。


## Intent？

要控制播放器，请转发 Intent 以启动 Unity 活动并在必要时对其进行扩展。如需了解更多信息，请参阅有关 [Intent 和 Intent 过滤器 (Intents and Intent Filters)](https://developer.android.com/guide/components/intents-filters) 的 Android 开发者文档。还可以使用 UnityPlayer Java API。
