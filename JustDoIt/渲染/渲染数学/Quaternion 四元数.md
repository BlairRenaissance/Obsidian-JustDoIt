
正如[[Complex Number 复数]]是实数的二维扩展一样，四元数是复数的四维扩展。

## 疯帽子版四元数

### 向直线国居民解释复数

复数可以被描述为：1个实数 ➕ 1个实数乘 `i`。两个复数的相乘可以通过乘法分配律来计算。

那么复数相乘的几何📐意义呢？

对于我们这些更高维文明的居民来说，可以将 $z ⋅ w$  中的 $z$ 看作是某种函数。当函数作用于整个平面时：
- 原点(0, 0)会保持不动
- x轴方向的单位向量(1, 0)会变换到 $z$ 的位置
- $w$ 会变换到 $z ⋅ w$  的位置。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752153540.png)

按着(0,0)位置，将(1,0)拽到 $z$ 的位置。这个过程中进行了某种旋转和缩放。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752153673.png)

显然如果乘以落在复平面的单位圆上的数字，那么这种相乘就是一种纯旋转。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752154030.png)

但是，如何向直线国居民解释旋转这种二维操作呢？

欸？！其实旋转只涉及一个数字——角度！

可以将单位圆上的点立体投影到轴上：
- 从(-1,0)向外发射无数条射线，射线与单位圆的交点被投影到射线与纵轴的交点上。
- 显然(0, i) (0, -i)没有变化，(1, 0)被投影到了(0, 0)， (-1, 0)被投影到了无穷远。
- 实数在0～1范围内的值被投影到了线段内部，实数在-1～0范围内的值被投影到了线段外部。

==由此得到的一维直线投影就是 i 左乘了单位圆上所有单位向量的结果。==

因此旋转就变成了沿轴移动。

显然原点旋转 $90^\circ$ 就是沿轴移动到 $i$ ：$1·i = i$，旋转 $135^\circ$ 就是移动到 $-\frac{\sqrt{2}}{2} + \frac{\sqrt{2}}{2}i$ 。 

当 $i$ 也旋转90度，就移动到了无穷远：$i·i = -1$。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752154257.png)

---
### 向平面国居民解释三维变换

如何向平面国居民解释三维旋转呢？

3D 空间中的每个点都被描述为：1个实数 ➕ 1个实数乘 `i` ➕ 1个实数乘 `j`。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752151632.png)

居住在二维平面上的居民 Felix 无法理解立体单位球，但可以理解被投影到二维平面上的单位球：
-  实数作为纵轴，平面为 i j 平面。
- 从(0,0,-1)向外发射无数条射线，射线与单位球的交点被投影到射线与水平面的交点上。
- 显然(i,j,0)(-i,j,0)(-i,-j,0)(i,-j,0)构成的单位圆不变，(0,0,1) 被投影到了(0,0,0)。
- 实数在0～1范围内的值被投影到了单位圆内，实数在-1～0范围内的值被投影到了单位圆外。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752224177.png)

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752202561.png)

三维向量的旋转平面国居民也可以理解了。假设还是计算  $z ⋅ w$，还是将 z 看作某种函数，w 可以看作空间里一个点，那么 $z ⋅ w$ 其实是 w 旋转了一定角度z'➕缩放了|z|。

捏着🤏粉色(0,0,1)顶点旋转。根据上一章，iz 平面上（z为实数轴）的旋转可以看作沿 i 轴移动。==由此得到的二维平面投影就是 i 左乘了单位球上所有三维单位向量的结果。==

ij 平面上的旋转则直接在平面国居民本来的领土上转就完事了，相当于左乘某个实数向量，如$(-\sqrt{2},\sqrt{2})$。

 j 的左乘既可以看作捏着🤏粉色顶点沿 j 轴移动。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/%E5%BD%95%E5%B1%8F2025-07-11%2016.58.20.gif)

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752225209.png)

那么我们就能够得到粉色(0,0,1)顶点旋转z‘后，投影的平面。那么球上任意向量的位置都能在投影后的平面上找到，对应的都是旋转了z'后的点，包括向量w。

注意因为投影的仅仅是单位球，所以这里仅表示旋转。但缩放的部分很容易，平面国居民也能算出长度，一乘就行了。

---
### 向立体国居民解释四元数

如何向立体国居民解释四元数旋转呢？

显然整个三维空间是单位四维超球的投影。

4D 空间中的每个点都被描述为：1个实数 ➕ 1个实数乘 `i` ➕ 1个实数乘 `j`➕ 1个实数乘 `k`。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752202282.png)

四维超球投影到三维空间时，规律和之前一样：

- (0,0,0,1)会投影到原点。
- 就像实数为0时，直线国投影中 i 和 -i 不会发生变动，平面国投影中 ij 平面不会发生变动。立体国的投影下，实数为0的 ijk 的球面也不会发生变动。
- ==实数在0～1范围内的值被投影到了单位球内，实数在-1～0范围内的值被投影到了单位圆外。==

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752214891.png)

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752215095.png)

捏着🤏粉色(0,0,0,1)顶点旋转。根据上一章，iz 复平面上的旋转可以看作沿 i 轴移动（虽然我们画不出来z轴，这是一个四维平面）。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752229881.png)

同时 j 和 k 会对应发生旋转，遵循拇指朝向 i 轴的右手法则，旋转角是 iz 复平面上旋转的角度。（别问为什么了，记住吧...）

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752229988.png)

==由此得到的三维投影面就是 i 左乘了四维单位超球上所有单位向量的结果。==（如下图）

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/%E5%BD%95%E5%B1%8F2025-07-11%2015.59.16%20(2)_%E5%89%AF%E6%9C%AC.gif)

假设左乘 $-\frac{\sqrt{2}}{2} + \frac{\sqrt{2}}{2}i$，除了沿 i 轴移动，j 和 k 则是遵循拇指朝向 i 轴的右手法则旋转了$135^\circ$。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752226774.png)


> [!Tip]
> 显然我们一直在做的事情是计算出完成了四元数旋转后的基平面。

---
## 正常人版四元数
