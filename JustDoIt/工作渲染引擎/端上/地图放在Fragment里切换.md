
**Fragment 包装类**

地图提供 SupportMapFragment 容器包装 MapView 创建地图，里面内含生命周期的绑定调用。
public class **SupportMapFragment** extends [Fragment](https://developer.android.com/reference/androidx/fragment/app/Fragment.html)  

A Map component in an app. This fragment is the simplest way to place a map in an application. It's a wrapper around a view of a map to automatically handle the necessary life cycle needs. Being a fragment, this component can be added to an activity's layout file simply with the XML below.

```
<fragment
    class="com.google.android.gms.maps.SupportMapFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

**Fragment**

Fragment 是 Android3.0 后引入的一个新的 API，他出现的初衷是为了适应大屏幕的平板电脑， 当然现在他仍然是平板 APP UI 设计的宠儿，而且我们普通手机开发也会加入这个 Fragment， 我们可以把他看成一个小型的 Activity，又称 Activity 片段！想想，如果一个很大的界面，我们 就一个布局，写起界面来会有多麻烦，而且如果组件多的话是管理起来也很麻烦！而使用 Fragment 我们可以把屏幕划分成几块，然后进行分组，进行一个模块化的管理！从而可以更加方便的在 运行过程中动态地更新 Activity 的用户界面！另外 Fragment 并不能单独使用，他需要嵌套在 Activity 中使用，尽管他拥有自己的生命周期，但是还是会受到宿主 Activity 的生命周期的影响，比如 Activity 被 destory 销毁了，他也会跟着销毁！

下图是文档中给出的一个 Fragment 分别对应手机与平板间不同情况的处理图：

![](http://www.runoob.com/wp-content/uploads/2015/08/41442282.jpg)