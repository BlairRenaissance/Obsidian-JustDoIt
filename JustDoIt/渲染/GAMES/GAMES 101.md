## Lecture 05~06 Rasterization 光栅化

### 三角形网格（Triangle Meshes）

- 三角形是最基础的多边形
- 三角形的重心插值计算简单
- 三个点唯一确定一个平面
- 三角形一定是凸多边形，内外定义清晰易于求解

---
### 频域（Frequency Domain）和时域（Time Domain）解释锯齿/走样

频域和时域

- 频域：描述信号在频率方面特性的一种坐标系
- 时域：描述数学函数或物理信号对时间关系的一种坐标系（图像信号的空间分布也可视为一种时间关系）
- 时域的卷积等于频域的乘积，反之，时域的乘积也等于频域的卷积
- 注：在图像处理中，频率是指图像中像素值变化的速度或图案变化的速率，它衡量的是单位空间内亮度变化的程度。

**傅里叶级数展开与傅里叶变换**

- 傅里叶级数展开：任意周期函数都可以写成一系列正弦和余弦函数的线性组合以及一个常数项
- 傅里叶变换：正变换会将时域图转换为频域图，逆变换会将频域图转换为时域图
- 图片本身可以视为一张傅里叶时域图，转换为频域图时默认认为图片边缘进行了平铺重复的拼接

**滤波（Filtering）：等价于卷积/加权平均**

- 低通滤波：低频信号通过，实现模糊效果
- 高通滤波：高频信号通过，实现边缘检测/图像锐化效果
- 带通滤波：指定范围频率信号通过
- 时域上的卷积核越大，转换到频域上后中心点越小（卷积核越大越模糊，相当于更极致的低通滤波）
![400](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1754989711.png)
![400](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1754989751.png)

**采样**

- 采样是单一频谱的复制粘贴（普通函数能转换成一段频谱，采样点的脉冲函数傅里叶变换后仍然是规则脉冲，采样是时域卷积，相当于频域的相乘）
- 采样率不够导致频谱混叠是走样产生的根本原因
- 砍掉高频信息就能有效避免频谱混叠，砍掉高频信息显然就是变模糊的过程

![500](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1754989809.png)
![300](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1754989845.png)

---
### 抗锯齿（AntiAliasing）

基于超采样的抗锯齿

- SSAA 超采样抗锯齿（Super-Sampling AA）

	- 严格超采样（比如采用2x2的4倍超采），然后降分辨率到目标分辨率
	- 质量极高，开销极大，实时渲染中通常不会使用

- MSAA 多重采样抗锯齿（Multi-Sampling AA）

	- 对SSAA的近似和改进
	- 只在几何边缘（多边形边界）进行超采样。即对同一个几何体，只进行一次采样
	-  存在问题：只对几何边缘的锯齿有效，对纹理锯齿、阴影锯齿等颜色边界不产生效果。
	-  存在问题：延迟渲染管线（Deffered Shading）中无法使用MSAA

- TAA 时间抗锯齿（Temporal AA）

	- 复用历史帧的信息进行混合，达成“超采样”
	- 存在问题：对于运动物体会产生拖影（Ghosting）现象

基于后处理的抗锯齿

- FXAA 快速近似抗锯齿（Fast Approximate AA）

	- 基于亮度差异检测边缘，然后进行模糊处理
	- 优点：开销很低，兼容性好（对几何、纹理、阴影锯齿均有效）
	- 缺点：可能模糊非锯齿的边界信息（比如UI，文字）

- MLAA 形态抗锯齿（Morphological AA）

	- 形态学分析边缘几何模式，然后进行处理
	- 更智能的边缘处理，开销较大

- SMAA 亚像素形态抗锯齿（Subpixel Morphological AA）

	- 基于MLAA的优化改进，支持亚像素分析
	- 优点：可处理亚像素精度，画质最佳且可以保留高频信息
	- 缺点：实现复杂，需要结合TAA解决时间性锯齿

基于深度学习的抗锯齿

- DLSS 深度学习超采样（Deep Learning Super Sampling）：NVIDIA的超采样技术
- FSR 保真度超分辨率（FidelityFX Super Resolution）：AMD的超采样技术

---
## Lecture 07~09 Shading 着色

### Blinn-Phong 光照模型

- 将光照分为环境光（Ambient）、漫反射光（Diffuse）和镜面高光（Specular），对复杂光照的近似和简化

![](https://pic1.zhimg.com/v2-90c6004c0527438e0910943f6bc6a524_1440w.jpg)

**漫反射 Diffuse Reflection**

![](https://pic3.zhimg.com/v2-54a9946078e56b9df65d54d6c609264e_1440w.jpg)

光照方向l，法线n，观察方向v

- 兰伯特Lambert光照模型：一种漫反射光照模型
- 核心理论为漫反射强度与入射光与法线夹角相关（即为入射光越接近垂直入射，漫反射越强），与观察角度无关
$$Idiffuse​=Ilight​⋅kdiffuse​⋅max(0,n⋅l)⋅attenuation$$
- 公式中分别为漫反射光照，光源光照，漫反射系数（材质决定），光照和法线夹角余弦（不取负值），光照衰减
- 课程中使用 $\dfrac{1}{d^2}$ 的衰减系数，但是实际使用时往往使用 $\dfrac{1}{a_0 + a_1 \cdot d + a_2 \cdot d^2}$  作为衰减系数，便于进行细微调整

**高光 Specular Highlight**

- 当观察方向与光照反射方向接近时，认为能够接收到高光反射
- Phong式光照模型中利用光照方向 l 和法线 n 计算出反射光方向，并和观察方向进行比较
- Blinn-Phong光照模型使用光照方向 l 和观察方向 v 的半程向量 h 和法线 n 进行比较，因为半程向量比反射方向好算太多
- 其中半角向量满足 $\mathbf{h} = \dfrac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|}$
$$I_{\text{specular}} = I_{\text{light}} \cdot k_{\text{specular}} \cdot \max(0, \mathbf{n} \cdot \mathbf{h})^{\text{shininess}} \cdot \text{attenuation}$$
- 公式中分别为高光，光源光照，高光系数（材质决定），法线与半角向量夹角余弦（不取负值），光泽度（调整高光范围，因为一次方的cos的变化过慢，使得高光面积很大），光照衰减

**环境光 Ambient Lighting**

- Blinn-Phong光照模型中的环境光是恒定不变的常数，与光源方向、法线、观察方向都无关。主要是为了保证始终有一定的光照会照亮物体

---
### 着色频率（Shading Frequncies）

![](https://pic1.zhimg.com/v2-c46ba3f86560b9687270f7df959f32a0_1440w.jpg)

从左到右增加着色频率，从上到下增加模型面数

逐三角面着色（也称Flat Shading）

- 每一个面根据面法线执行一次光照计算，结果应用到整个三角面
- 计算量极低，每个面只进行一次光照计算

逐顶点着色（也称Gouraud Shading）

- 每个顶点根据顶点法线执行一次光照计算，面内部光照通过插值计算获得
- 计算量较低，每个面平均进行一次光照计算（并不严谨，实际受顶点数和面数影响），同时有插值运算开销

逐像素/片元着色（也称Phong Shading）

- 每个像素根据自身法线（法线根据顶点法线插值获得）独立进行光照计算并应用于自身
- 计算量较高，像素法线根据顶点法线插值获取，每个像素都进行一次光照计算

计算量大小并不绝对，当面数无限增加（比如数亿个面的高模模型）超过屏幕像素数量时，逐三角面着色反而效果最好计算量最大。

---
### 纹理映射（Texture Mapping）

- 3D模型的表面能够映射到2D纹理图像中
- 纹理也就是常说的贴图，顶点信息中通常带有纹理坐标（uv）信息
- 片元的纹理坐标通过三角形重心插值获取，再依据纹理坐标在纹理中采样获得最终的纹理颜色
- 纹理具有多种类型，最常见的就是漫反射纹理，其他的还有法线纹理、AO纹理、高光纹理、立方体纹理等

---
**重心坐标（Barycentric Coordinates）**

- 三角形插值时的计算方式
- 三角形内部的点都可以使用三个顶点的线性组合进行表示，且满足如下性质
- 重心坐标的问题：重心坐标在投影过程中会发生变化，所以需要插值如深度等信息时，不可以用投影后的点进行插值，必须对3D空间中原始的位置进行插值运算

> [!Warning] 
> 重心坐标不具有投影不变性

---
**纹理映射**

- 当纹理过小时，多个屏幕像素映射到纹理中同一纹素中，导致产生纹理锯齿
- 当纹理过大时，较少的屏幕像素映射到较多的纹素中，使得纹理像素信息丢失，可能产生摩尔纹、画面断裂等问题

解决方案

- 纹理过小时：

	- 最邻近算法（Nearest）：不进行处理，未落到纹理像素中心的坐标，直接使用所处像素的中心的颜色值
	- 双线性插值（Bilinear）：对于未落到纹理像素中心的坐标，根据位置取周围4个纹理像素进行水平和竖直的线性插值获得最后的颜色值
	- 双立方插值（Bicubic）：使用更多的纹理像素（比如16个）进行插值运算，以获得更好的效果

- 纹理过大时：

	- 使用Mipmap技术，生成一系列的低分辨率图像（增加1/3的内存空间）
	- 启用各向异性过滤（Anisotropic Filtering）

---
**Mipmap**

![](https://picx.zhimg.com/v2-f71a6e913ea61eb8ebbd338bd16ab4f1_1440w.jpg)

- 基于原图像生成一系列的低分辨率图像（增加1/3的内存空间）
- Mipmap只能进行方形的近似范围查询
- 查询Mipmap层级的计算：

	- 计算出当前像素在纹理中占据大小的近似值：取周围临近的像素，计算和当前像素的uv坐标差值，取最大的值认为是当前像素在纹理中占据大小的近似正方形边长 L
	- 计算出应该查询的Mipmap层级：由于Mipmap以2倍的速度缩小，计算 $D = \log_2 L$ 即可

- 三线性插值：如果计算出的层数不为整数，对上下两个层级进行两次查询，然后在层间进行一次线性插值即可

![](https://pica.zhimg.com/v2-6670b1a0c9faeff8d6b59eee59577368_1440w.jpg)

---
**各向异性过滤（Anisotropic Filtering）**

- Mipmap只能进行方形的范围查询，对于狭长的范围查询效果不好，这时可以通过各向异性过滤解决
- 各向异性过滤在Mipmap基础上生成额外生成了一系列横向压缩和纵向压缩的图像（总计增加3倍内存空间）
- 使得在范围查询上的效果更好

![](https://pic2.zhimg.com/v2-2bbfbafd4c23aacce0d33e3430b07b67_1440w.jpg)

---
### 纹理应用（Application of Texture）

环境贴图（Environment Map）

- 使用一张纹理/贴图实现周围的环境光照（比如天空盒就常用立方体贴图实现）
- 近似认为环境光是无限远的，改变物体位置不会改变环境映射的结果（不考虑遮挡的前提下）
- 常用的有球面贴图（Spherical Map）和立方体贴图（Cube Map）

凹凸贴图/法线贴图（Bump Map / Normal Map）

- 区别

	- 凹凸贴图（Bump/Height map）存的是“高度 h(u,v)”（灰度），在着色时现场用高度的梯度去“临时”扰动法线。
	- 法线贴图（Normal map）存的是“三通道法线方向 n′(u,v)”（RGB），着色时解包 RGB→`[-1,1]`，重归一化，乘以 TBN 矩阵变到世界/视图空间，直接参与光照。
	
	- 可以通过凹凸贴图的深度变化计算出法线，但难以反算：
	    - 高度→法线：对高度做 Sobel/有限差分，再归一化即可（很多工具一键生成）。
	    - 法线→高度：通常不可逆（缺绝对高度且不一定可积），只能近似积分，易产生伪影。

- 共同点
	- 都不改变几何轮廓和真实几何位置，只改变光照反应（阴影明暗/高光），所以轮廓处仍然是平的。想改变轮廓，需要位移贴图/细分或视差遮挡映射。

- 常用的法线贴图通常为蓝色。因为法线贴图是基于顶点切线空间的，实际使用时需要根据顶点的切线（Tangent）和法线（Normal）计算出副切线（Bitangent），组成TBN矩阵（Tangent、Bitangent、Normal）。然后使用TBN矩阵与法线贴图采样结果相乘并归一化得到最后的法线
- 由于通常法线都是对于原法线的微调，所以Normal（即Blue通道）的值比较大，所以法线贴图通常呈现为蓝色

位移/置换贴图（Displacement Map）

- 实际改变模型的顶点位置，得到更加真实的高模效果（支持自遮挡阴影，投射阴影等）

![](https://pic2.zhimg.com/v2-1ab3c043d1e6a2a90f78f3388ea5b777_1440w.jpg)

程序纹理（Procedural Map）

- 常用于木纹纹理、岩石纹理、山脉、年轮等2D或3D纹理中
- 常常使用噪声函数，比如著名的Perlin柏林噪声
- 不会生成具体的纹理图，而是通过函数确定不同位置的渲染结果

环境光遮蔽纹理（AO Map）

- 环境光遮蔽（Ambient Occlusion）指的是由于周围物体的遮蔽导致的环境光减弱的现象（比如凹陷处更暗，凸出处更亮）
- 记录预定义的AO强度，实际使用时直接进行采样而不需要进行实时计算。保证在低性能条件下也有较好的AO效果

---
## Lecture 10~12 Geometry 几何

### 有向距离场SDF（Signed Distance Field）

怎么搞这么难受的术语啊... 明明就是直译的意思... 带符号距离... [2D distance functions](https://iquilezles.org/articles/distfunctions2d/)

长宽上等比（XY维度上量纲相同）的基础距离场图形可以由原点、目标点和距离构成。那么SDF的构建函数就是一个向量和一个距离标量的函数，比如：

- 圆形：`sdCircle(float2 pos, float d)`  
- 正方形：`sdSquare(float2 pos, float d)`  
- 正菱形：`sdRhombic(float2 pos, float d)`  

```
float sdCircle( vec2 p, float r )
{
    return length(p) - r;
}
```

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/202508121653205.png)


XY维度上量纲不同的距离场，SDF的构建函数长得更像中学时的图形定义：

- 椭圆形：`sdEllipse(float2 pos, float w, float h)`  
- 矩形：`sdRect(float2 pos, float w, float h)`  
- 菱形：`sdRhombus(float pos, float w, float h)` 
- 梯形：`sdTrapezoid(float2 pos, float up, float down, float h)`  
- 等腰三角形：`sdTriangleIsocaeles(float2 pos, float down, float h)`  
- 任意多边形：`sdAnything(float2 pos, float2[] points)`  
- 极坐标下任意正多边形：`sdPolygopn(float2 pos, float r)`  

