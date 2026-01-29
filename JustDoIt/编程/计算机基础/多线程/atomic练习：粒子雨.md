
这个项目叫 **“ASCII 粒子雨引擎”**。

它不需要配置 OpenGL 或 DirectX，只需要一个 C++ 编译器（VS、Xcode 或 VS Code 均可）。虽然它是跑在控制台（Terminal）里的，但它模拟了游戏引擎最核心的 **“逻辑-渲染分离”** 架构。

做完这个，你将彻底掌握：

1. **预分配内存**（解决 Vector 扩容崩溃）。
    
2. **逻辑与渲染分离**（模拟 60FPS 游戏循环）。
    
3. **原子同步**（解决画面撕裂）。

---
### 项目目标

在黑框框里，让一堆字符像《黑客帝国》代码雨或者下雪一样不断下落。

- **线程 A（逻辑）**：计算每个雪花的新位置（下落）。
    
- **线程 B（渲染）**：把当前所有雪花的位置画出来。

### 阶段一：搭建“必崩”的脚手架

首先，我们写一个没有保护的多线程版本。

**任务 1：准备数据结构**

不要用 `vector` 的 `push_back`，我们要用**预分配**方案（之前学到的教训）。

**运行观察**：

你会发现雪花有时会“闪烁”，有时位置很诡异，甚至如果有更复杂的数据，字符会乱跳。因为渲染线程在读第 1 个粒子时，逻辑线程可能已经把第 100 个粒子更新了两遍了，导致画面上的粒子**时间线不统一**（这就是撕裂）。

---
### 阶段二：加入“原子锁”进行修复

现在，请你运用学到的 **Acquire / Release** 知识来改造它。

**挑战任务**：

1. 定义一个 `std::atomic<bool> g_dataReady{false};`。
    
2. **逻辑线程**：更新完所有粒子后，设置标记为 `true`（Release），然后稍微等待一下让渲染线程有机会画。
    
3. **渲染线程**：只有当标记为 `true`（Acquire）时才开始画，画完后把标记设回 `false`。

**代码修改提示**：

```C++
// 逻辑线程
if (!g_dataReady.load(std::memory_order_relaxed)) {
    // 更新所有粒子...
    // 提交！
    g_dataReady.store(true, std::memory_order_release);
}

// 渲染线程
if (g_dataReady.load(std::memory_order_acquire)) {
    // 渲染...
    // 渲染完了，告诉逻辑线程可以算下一帧了
    g_dataReady.store(false, std::memory_order_relaxed);
}
```

> [!example]- 点击展开代码：通过“原子锁”分隔逻辑线程与渲染线程操作
> ```cpp
> #include <iostream>
> #include <thread>
> #include <chrono>
> #include "AtomicTest.h"
> 
> const int WIDTH = 40;
> const int HEIGHT = 20;
> const int MAX_PARTICLES = 100;
> 
> struct Particle {
>     int x, y;
>     int speed;
>     bool active;
> };
> 
> // 全局数据（共享内存）
> static Particle g_particles[MAX_PARTICLES]; // 预分配，绝对安全！
> static std::atomic<bool> g_dataReady{false};
> 
> void UpdateThread() {
>     while (true) {
>         while(g_dataReady.load(std::memory_order_acquire)){
>             std::this_thread::yield();
>         }
> 
>         for (int i = 0; i < MAX_PARTICLES; ++i) {
>             if (g_particles[i].active) {
>                 g_particles[i].y += g_particles[i].speed;
>                 // 如果掉出屏幕，重置到顶部
>                 if (g_particles[i].y >= HEIGHT) g_particles[i].y = 0;
>             }
>         }
> 
>         g_dataReady.store(true, std::memory_order_release);
>     }
> }
> 
> void RenderThread() {
>     char buffer[HEIGHT][WIDTH]; // 画布
> 
>     while (true) {
>         while (!g_dataReady.load(std::memory_order_acquire)) {
>             std::this_thread::yield();
>         }
> 
>         // 1. 清空画布
>         for(int y=0; y<HEIGHT; ++y)
>             for(int x=0; x<WIDTH; ++x) buffer[y][x] = ' ';
> 
>         for (int i = 0; i < MAX_PARTICLES; ++i) {
>             if (g_particles[i].active) {
>                 int px = g_particles[i].x;
>                 int py = g_particles[i].y;
>                 if (px >= 0 && px < WIDTH && py >= 0 && py < HEIGHT) {
>                     buffer[py][px] = '*'; // 画个雪花
>                 }
>             }
>         }
> 
>         g_dataReady.store(false, std::memory_order_release);
> 
>         // 3. 打印这一帧（为了不闪瞎眼，稍微停顿一下）
>         system("clear"); // 清屏
> 
>         for(int y=0; y<HEIGHT; ++y) {
>             for(int x=0; x<WIDTH; ++x) std::cout << buffer[y][x];
>             std::cout << "\n";
>         }
> 
>         std::this_thread::sleep_for(std::chrono::milliseconds(33)); // ~30 FPS
>     }
> }
> 
>   
> 
> int main(){
>     std::srand(std::time(nullptr));
>     
>     for(int i = 0; i < MAX_PARTICLES; ++i) {
>         g_particles[i].x = std::rand() % WIDTH;
>         g_particles[i].y = std::rand() % HEIGHT;
>         g_particles[i].speed = 1 + (std::rand() % 2); // 速度 1 或 2
>         g_particles[i].active = true; // 关键！让它活过来
>     }
>     
>     std::thread update(UpdateThread);
>     std::thread render(RenderThread);
> 
>     update.join();
>     render.join();    
> 
>     return 0;
> }
> ```

---
### 阶段三：双缓冲（Double Buffering）

上面的阶段二有一个缺点：逻辑线程在等渲染线程画完（串行了），这浪费了 CPU。

真正的高手（也就是你）要实现**无锁双缓冲**。

**架构设计**：

1. 准备**两个**数组：`Particle g_bufferA[MAX]` 和 `Particle g_bufferB[MAX]`。
    
2. 一个原子指针 `std::atomic<Particle*> g_currentRenderBuffer;` 指向最新的那个。
    
3. **逻辑线程**：一直在后台写“那个没人看的 Buffer”，写好了就用原子操作**交换指针**。
    
4. **渲染线程**：永远只读指针指向的那个 Buffer。

**这才是真正的游戏引擎写法。** 

> [!example]- 点击展开代码：双缓冲分隔逻辑线程与渲染线程操作
> ```cpp
> #include <iostream>
> #include <thread>
> #include <chrono>
> #include "DoubleBufferTest.h"
> 
> // 其实类似三缓冲了，因为渲染线程还有一个自己的私有缓冲区
> 
> const int WIDTH = 40;
> const int HEIGHT = 20;
> const int MAX_PARTICLES = 100;
> 
> struct Particle {
>     int x, y;
>     int speed;
>     bool active;
> };
> 
> // 全局数据（共享内存）
> Particle g_frontbuffer[MAX_PARTICLES]; // 预分配，绝对安全！
> Particle g_backbuffer[MAX_PARTICLES]; // 预分配，绝对安全！
> 
> std::atomic<Particle*> g_currentBuffer{g_frontbuffer};
> std::atomic<bool> g_renderLock{false};
> 
> // 建模线程
> 
> void UpdateThread() {
>     // 不假设一定是 backbuffer，而是看看当前 Render 正在用谁
>     Particle* currentRender = g_currentBuffer.load();
> 
>     // 如果 Render 用的是 A，我就用 B；如果 Render 用 B，我就用 A。
>     // 这样无论全局变量怎么改，这里永远是正确的“互斥”状态。
>     Particle* backBuffer = (currentRender == g_frontbuffer) ? g_backbuffer : g_frontbuffer;
> 
>     while (true) {
>         for (int i = 0; i < MAX_PARTICLES; ++i) {
>             if (backBuffer[i].active) {
>                 backBuffer[i].y += backBuffer[i].speed;
>                 if (backBuffer[i].y >= HEIGHT) backBuffer[i].y = 0;
>             }
>         }
> 
>         while(g_renderLock.load(std::memory_order_acquire)){
>             std::this_thread::yield();
>         }
> 
>         backBuffer = g_currentBuffer.exchange(backBuffer, std::memory_order_acq_rel);
> 
>         std::this_thread::sleep_for(std::chrono::milliseconds(16));
>     }
> }
> 
> // 渲染线程
> void RenderThread() {
>     char buffer[HEIGHT][WIDTH]; // 画布
>     Particle localCopy[MAX_PARTICLES];
> 
>     while (true) {
>         g_renderLock.store(true, std::memory_order_release);
>         
>         Particle* currentParticle = g_currentBuffer.load(std::memory_order_acquire);
>         
>         std::memcpy(&localCopy, currentParticle, MAX_PARTICLES * sizeof(Particle));
>         
>         g_renderLock.store(false, std::memory_order_release);
> 
>         // 1. 清空画布
>         for(int y=0; y<HEIGHT; ++y)
>             for(int x=0; x<WIDTH; ++x) buffer[y][x] = ' ';
> 
>         for (int i = 0; i < MAX_PARTICLES; ++i) {
>             if (localCopy[i].active) {
>                 int px = localCopy[i].x;
>                 int py = localCopy[i].y;
>                 if (px >= 0 && px < WIDTH && py >= 0 && py < HEIGHT) {
>                     buffer[py][px] = '*'; // 画个雪花
>                 }
>             }
>         }
> 
>         // 3. 打印这一帧（为了不闪瞎眼，稍微停顿一下）
>         system("clear"); // 清屏
>         for(int y=0; y<HEIGHT; ++y) {
>             for(int x=0; x<WIDTH; ++x) std::cout << buffer[y][x];
>             std::cout << "\n";
>         }
> 
>         std::this_thread::sleep_for(std::chrono::milliseconds(33)); // ~30 FPS
>     }
> }
> 
> int main(){
>     std::srand(std::time(nullptr));
>     for(int i = 0; i < MAX_PARTICLES; ++i) {
>         g_frontbuffer[i].x = std::rand() % WIDTH;
>         g_frontbuffer[i].y = std::rand() % HEIGHT;
>         g_frontbuffer[i].speed = 1 + (std::rand() % 2); // 速度 1 或 2
>         g_frontbuffer[i].active = true; // 关键！让它活过来
>         g_backbuffer[i] = g_frontbuffer[i];
>     }
> 
>     std::thread update(UpdateThread);
>     std::thread render(RenderThread);
> 
>     update.join();
>     render.join();
> 
>     return 0;
> }
> ```

---
### 阶段四：工程化通用做法

在实际的游戏引擎（如 Unreal 或 Unity 底层）中，通常不会直接存指针，而是存 **Index (0 或 1)**。这样数学上更严谨，逻辑也更清晰。

```C++
// 全局
Particle g_buffers[2][MAX_PARTICLES]; // 二维数组：g_buffers[0] 和 g_buffers[1]
std::atomic<int> g_renderIndex{0};    // 0 或 1

void UpdateThread() {
    // 渲染线程用的是 g_renderIndex，那我就用 1 - g_renderIndex
    // 比如渲染用 0，我就用 1；渲染用 1，我就用 0。
    int myIndex = 1 - g_renderIndex.load(); 

    while (true) {
        // 操作 g_buffers[myIndex] ...
        
        // 交换时，只要把索引反转并发布即可
        // (这里需要配合原子操作逻辑调整，通常是更新 g_renderIndex)
    }
}
```

> [!example]- 点击展开代码：原子变量改用索引
> ```cpp
> #include <iostream>
> #include <thread>
> #include <chrono>
> #include "DoubleBufferTest.h"
> 
> // 其实类似三缓冲了，因为渲染线程还有一个自己的私有缓冲区
> 
> const int WIDTH = 40;
> const int HEIGHT = 20;
> const int MAX_PARTICLES = 100;
> 
> struct Particle {
>     int x, y;
>     int speed;
>     bool active;
> };
> 
> // 全局数据（共享内存）
> Particle g_buffer[2][MAX_PARTICLES];
>   
> std::atomic<int> g_currentBuffer{0};
> std::atomic<bool> g_renderLock{false};
> 
> // 建模线程
> 
> void UpdateThread() {
>     int currentIndex = 1 - g_currentBuffer.load();
>     while (true) {
>         for (int i = 0; i < MAX_PARTICLES; ++i) {
>             if (g_buffer[currentIndex][i].active) {
>                 g_buffer[currentIndex][i].y += g_buffer[currentIndex][i].speed;
>                 // 如果掉出屏幕，重置到顶部
>                 if (g_buffer[currentIndex][i].y >= HEIGHT) g_buffer[currentIndex][i].y = 0;
>             }
>         }
> 
>         while(g_renderLock.load(std::memory_order_acquire)){
>             std::this_thread::yield();
>         }
> 
>         currentIndex = g_currentBuffer.exchange(currentIndex, std::memory_order_acq_rel);
> 
>         std::this_thread::sleep_for(std::chrono::milliseconds(16));
>     }
> }
> 
> // 渲染线程
> void RenderThread() {
>     char buffer[HEIGHT][WIDTH]; // 画布
>     Particle localCopy[MAX_PARTICLES];
>     
>     while (true) {
>         g_renderLock.store(true, std::memory_order_release);
>         
>         int currentIndex = g_currentBuffer.load(std::memory_order_acquire);
>         
>         std::memcpy(&localCopy, g_buffer[currentIndex], MAX_PARTICLES * sizeof(Particle));
> 
>         g_renderLock.store(false, std::memory_order_release);
> 
>         // 1. 清空画布
>         for(int y=0; y<HEIGHT; ++y)
>             for(int x=0; x<WIDTH; ++x) buffer[y][x] = ' ';
> 
>         for (int i = 0; i < MAX_PARTICLES; ++i) {
>             if (localCopy[i].active) {
>                 int px = localCopy[i].x;
>                 int py = localCopy[i].y;
>                 if (px >= 0 && px < WIDTH && py >= 0 && py < HEIGHT) {
>                     buffer[py][px] = '*'; // 画个雪花
>                 }
>             }
>         }
> 
>         // 3. 打印这一帧（为了不闪瞎眼，稍微停顿一下）
>         system("clear"); // 清屏
>         for(int y=0; y<HEIGHT; ++y) {
>             for(int x=0; x<WIDTH; ++x) std::cout << buffer[y][x];
>             std::cout << "\n";
>         }
>         std::this_thread::sleep_for(std::chrono::milliseconds(33)); // ~30 FPS
>     }
> }
> 
> int main(){
>     std::srand(std::time(nullptr));
>     for(int i = 0; i < MAX_PARTICLES; ++i) {
>         g_buffer[0][i].x = std::rand() % WIDTH;
>         g_buffer[0][i].y = std::rand() % HEIGHT;
>         g_buffer[0][i].speed = 1 + (std::rand() % 2); // 速度 1 或 2
>         g_buffer[0][i].active = true; // 关键！让它活过来
>         g_buffer[1][i] = g_buffer[0][i];
>     }
>     
>     std::thread update(UpdateThread);
>     std::thread render(RenderThread);
> 
>     update.join();
>     render.join();
> 
>     return 0;
> }
> ```


---
### 本章练习过程踩的坑

#### Q1: 既然 `std::mutex` (锁) 写起来简单又不容易出错，为什么还要用难写的 `std::atomic`？

- **你的疑惑**：能不能只用锁？
    
- **真相**：
    
    1. **太慢**：锁会导致线程**睡眠和唤醒**（Context Switch），这在每帧几千次调用的图形渲染中是不可接受的。
        
    2. **伤缓存**：线程睡着后，CPU 缓存里的数据（纹理/顶点）会被清空，醒来后要重新加载，这对图形性能是毁灭性的。
        
    
    - **结论**：逻辑复杂用锁，高性能热点数据（如状态标记、计数器）用 Atomic。
        

#### Q2: 面试必问题：`std::shared_ptr` 到底是线程安全的吗？

- **你的疑惑**：它不是叫智能指针吗？应该很聪明吧？
    
- **真相**：它有“两张面孔”。
    
    1. **引用计数**是安全的（原子操作）。
        
    2. **指针本身**是不安全的（因为它是两个变量：`ptr` + `control_block`）。
        
    
    - **结论**：多线程读写**同一个** `shared_ptr` 变量时，必须加锁。
        

#### Q3: 为什么 `load(memory_order_release)` 是非法的？

- **你的疑惑**：我不可以“发布”一个“读取”动作吗？
    
- **真相**：语义矛盾。
    
    - **Release (发布)** = 我写完了，给你们看（必须配合 **Store/写**）。
        
    - **Acquire (获取)** = 我读到了，我要同步数据（必须配合 **Load/读**）。
        
    - **结论**：你不能通过“签收快递”（读）来完成“发货”（写）。
        

#### Q4: `while (g_dataReady.load())` 到底是在等 true 还是 false？

- **你的疑惑**：这里的逻辑容易绕晕。
    
- **真相**：这是“反向等待”。
    
    - `while(true)` = 只要它是真的，我就**卡住/死等**。
        
    - **生产者**：若盘子满 (`true`)，我就死等，直到变空。
        
    - **消费者**：若盘子空 (`false`)，我就死等，直到变满。
        

#### Q5: 为什么我的粒子系统一运行就崩了？(Vector 陷阱)

- **你的错误**：一边 `push_back` (写)，一边遍历读取 (读)。
    
- **真相**：**`std::atomic` 只能保护那个“数字”本身，它保护不了它“指向的内存”。** 虽然用原子操作完美地同步了“数量”，但没有同步“容器的结构变化”。多线程环境下`std::vector` 不是线程安全的。`std::vector` 扩容（Reallocation）时会**搬家**并**销毁旧内存**。读取线程拿着旧地址去访问，直接导致段错误。
    
- **修正**：**预分配内存 (`reserve` / 数组)**，或者使用双缓冲。

#### Q8: 为什么动画卡顿，完全没有并行的感觉？(锁的范围)

- **你的错误**：把 `std::this_thread::sleep_for` 包在了锁里面。
    
- **真相**：你占着茅坑（锁）睡觉。逻辑线程想工作却进不来，大家只能串行排队。
    
- **修正**：**拿完数据立刻释放锁**，在锁外面睡觉/处理耗时任务。
    

#### Q9: 为什么双缓冲只有前几个粒子在动，后面都是黑的？

- **你的错误**：`memcpy` 的大小填了 `MAX_PARTICLES` (100)。
    
- **真相**：`memcpy` 也是按**字节 (Byte)** 算的。你只拷了 100 个字节（大约 8 个粒子），剩下的内存没拷过去。
    
- **修正**：`MAX_PARTICLES * sizeof(Particle)`。

#### Q10: 为什么粒子飞得像瞬移一样快？

- **你的错误**：逻辑线程没有任何 `sleep`。
    
- **真相**：逻辑线程跑得比渲染线程快几百倍（比如 2000 FPS vs 30 FPS）。渲染线程每一帧看到粒子都已经绕地球好几圈了。
    
- **修正**：逻辑线程也要模拟帧率 (`sleep(16ms)`)。

#### Q11: 为什么不建议硬编码 `backBuffer = &g_bufferB`？

- **你的疑惑**：反正现在是对的，为什么要改？
    
- **真相**：**单一真理来源 (Single Source of Truth)**。如果你手动指定，一旦全局初始状态改变，或者扩展成三缓冲，你的手动指定就会失效（导致读写冲突）。
    
- **修正**：让算法动态决定 (`1 - current_index`)。