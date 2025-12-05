## Tips

如果 Instruments 有关于某个函数的原始源代码的更多信息，函数名会显示成黑色，否则为灰色。

Heaviest Stack Trace 中，每个符号行都会标注「binary of symbol」，本质是告诉你：**这个消耗 CPU / 能耗的符号，属于哪个二进制文件**。
- 如果 binary 是你的 App 名称 → 性能瓶颈在**自己的代码**，优先优化；
- 如果 binary 是系统框架（如 UIKitCore）→ 瓶颈在系统 API，需检查「是否频繁调用昂贵系统方法」「参数是否合理」（比如频繁刷新 UI、过度调用 `CoreData` 持久化）；
- 如果 binary 是第三方库 → 需检查「第三方库版本是否有性能问题」「调用方式是否错误」（比如大量异步请求未节流）。
- 如果「binary of symbol」显示为 `<redacted>`、`unknown` 或内存地址，说明**符号表缺失**。通常是dSYM 文件缺失/不匹配、 App 开启了代码混淆、设备端二进制被系统保护（如系统库）等原因。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1764755767.png)


按住 ==Option== 键点击展开三角形，能一键展开全部调用树。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Adobe%20Express%20-%2003-01-expand-call-tree-with-option.gif)


在调用树上右键可以选择「Open in Source Viewer」或「Reveal in Xcode」，可以打开堆栈对应的代码。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202512032151993.png)

