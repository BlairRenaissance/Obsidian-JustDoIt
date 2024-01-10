# Useful

超好用的OpenGL Doc： https://docs.gl/

# Vertex Buffer and Drawing a Triangle

**VBO** 其实就是一个Buffer，一个memory buffer，是GPU上的内存，在VRAM(Video RAM)中。所以其实就是将一些数据放进GPU's VRAM。

**DrawCall** 指令(which is basically a draw command)，是CPU完成了一些工作后，从CPU发出DrawCall指令通知GPU。大致是解释一下数据。

**Shader** 就是运行在显卡上的程序(a shader is just a bunch of code that we can write which runs on the GPU)。

**OpenGL** 实际上是一个状态机，它已经准备好了Data和Shader，这是它状态的一部分。控制绘制是通过变更状态进行，即设置要用的Data数据和Shader。OpenGL是上下文相关的，所以在OpenGL中生成的所有东西都会被分配一个唯一的标识符(OpenGL works as a state machine, which means that everything you generate is contextual. Everything you generate in OpenGL assigns a unique identify)。


```
void glVertexAttribPointer( GLuint  index,  
						    GLint   size,  
						    GLenum  type,  
						    GLboolean   normalized,  
						    GLsizei     stride,  
						    const GLvoid *  pointer);
```

- _`index`_ 是当前属性的索引值(Specifies the index of the generic vertex attribute to be modified)。比如顶点上分别有位置/纹理/法线，分别对应index 0/1/2。
- _`normalized`_ 标识是否访问数据时需要进行标准化。颜色值在Shader中使用时需要被标准化为0到1的float值，这一步标准化可以在CPU中完成，也可以偷懒让GPU帮忙做。
- _`stride`_ 是一个顶点所有属性的总字节长度(Specifies the byte offset between consecutive generic vertex attributes)。
- _`pointer`_ 是一个顶点内当前属性的字节偏移值。

永远不要忘记的重要一步，否则将得到黑屏：To enable and disable a generic vertex attribute array, call `glEnableVertexAttribArray` and `glDisableVertexAttribArray` with `index`.