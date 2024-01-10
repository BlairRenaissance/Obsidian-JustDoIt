
# 接口

```Java
/**  
 * 开关控制HD道路是否响应阴影（阴影总开关下的子开关项）  
 * 默认HD道路不响应阴影  
 * @param hdCastShadow  
 */  
public void switchHDRoadShadow(boolean hdCastShadow){  
    if( mMapEngineRef.get()!=null) {
	    mMapEngineRef.get().getMapJniWrapper().setPipe(
	    TXMapJniWrapper.TXMapEventType_SwicthHDRoadShadow, hdCastShadow ? 1 : 0);  
    }
}
```

接口调用示例

```Java
mMap.getStyleController().switchHDRoadShadow(ck.isChecked());
```

# 备注

1. 无法独立出立交桥概念，因此开关是控制HD道路是否响应阴影。
2. HD道路响应阴影开关仍受阴影总开关调控。如果阴影总开关为关闭状态，单独打开HD道路响应阴影开关不会投射阴影。
3. 已经生成的阴影图无法进行立即更新，因此打开开关后不对当前瓦片生效，是对新加载的瓦片生效，或重启生效。

