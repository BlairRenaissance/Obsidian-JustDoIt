

# 接口能力

支持 `setPosition / setAnchor / setAngle / setIcon / setScale`。

设置图片支持

```
options.setIcon(new BitmapDescriptor(R.drawable.cluster_group));
```

或

```
options.setIcon(new BitmapDescriptor(BitmapFactory.decodeResource(getResources(), R.drawable.cluster_group)));
```



# 回调

``` Java
@Override  
public void onBatchClick(List<BaseOverlay> elementId, TappedInfo elem, float screenX, float screenY) {  
  Integer subId = elem.getSubID();  StringBuilder sb = new StringBuilder();

  for (int i = 0; i < elementId.size(); i++) { 
    sb.append(elementId.get(i).getId());  sb.append("-"); 
    sb.append(elem.getSubID());  } // 【MarkerID – subID】  
}
```

注册监听后需要调用的回调是onBatchClick。

之前会直接返回被点击到的Overlay，现在也是，只多一个subID，当下点击到的Marker在设置BatchMarker时第几个传入subID就是多少。

回调中的拼接出的 unique ID 为【MarkerID – subID】。


---


例如：

某五角星的序号为8-95，MarkerID = 8，subID = 95。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Pasted%20image%2020230927021317.png)


---

某蓝色标签的序号为7-61，MarkerID = 7，subID = 61。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Pasted%20image%2020230927021430.png)



# 调整Marker次序


可调用引擎内原有接口设置Marker优先级。

```
/**  
 * 设置显示优先级  
 *  
 * zIndex越大越靠上显示  
 *  
 * @param priority  显示优先级  
 */  
public void setZIndex(int priority) {  
    mPriority = priority;  
    if (mJniWrapper != null) {  
        mJniWrapper.setMarkerPriority(mId, priority);  
    }  
}
```

例如：

```
mMarker.setZIndex(10);  
otherMarker.setZIndex(11);
```


