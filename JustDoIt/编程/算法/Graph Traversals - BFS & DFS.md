# 1.  Breadth First Search and Depth First Search

BFS：用 queue 来管理待访问的顶点。

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752755956.png)


DFS：用 stack 来管理被挂起的顶点。
核心思想是当你从当前顶点发现一个未被访问过的新顶点时，算法会立即**暂停对当前顶点的探索**，将当前顶点挂起，优先去访问新顶点，并递归地继续这个过程。Once you have visited a new vertex, suspend the exploration of current vertex and start exploring new vertex.

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752756356.png)


![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752756448.png)

# 2. Articulation Point and Biconnected Components

![](https://raw.githubusercontent.com/BlairRenaissance/ImageHost/main/Clipboard_Screenshot_1752759699.png)

图中讲解的是图论中 **关节点（Articulation Point）** 和 **双连通分量（Biconnected Components）** 的概念，以及如何通过DFS来识别它们。
### 核心概念

1. **关节点（Articulation Point）**：
    - 关节点是图中的 “脆弱点”，移除它会导致图分裂成多个子图。
    - 一个顶点是关节点，当且仅当移除该顶点及其相关边后，图的连通分量数量增加。比如上图中3是关节点，连接4 5移除关节点后联通分量数量增加。
2. **双连通分量（Biconnected Components）**：
    - 没有关节点的极大子图称为双连通分量。
    - 双连通分量中的任意两点之间至少存在两条不相交的路径。
3. **DFS 遍历**：
    - 通过 DFS 遍历图，记录每个顶点的发现时间（d）和能到达的最早顶点的发现时间（L）。
    - 发现时间（d）：顶点被首次访问的顺序，即`dfs`序号。
    - 最早可达时间（L）：顶点能通过树边和**至多一条回边**到达的**最早顶点**的`dfs`序号。（因为DFS是不访问首次发现的顶点的）。

### 图中关键信息

1. **DFS 树**（左侧）：
    - 展示了 DFS 遍历的顺序和树结构。
    - 顶点旁的 `d` 表示发现时间。
    - 虚线表示回边（非树边）。
2. **表格**（中间）：
    - 第一行（1-6）是顶点编号。
    - 第二行（d）是每个顶点的发现时间。
    - 第三行（L）是每个顶点的最早可达时间。
3. **条件判断**（中间）：
    - `L[v] ≥ d[u]`：判断顶点 `u` 是否为关节点的条件。
    - **若`u`是深度优先搜索树的根节点**
	    - 关节点的定义是删除该节点后，图会变成两个或多个连通分量。对于根节点`u`，若它有两棵或以上的子树，那么删除`u`后，这些子树会相互分离，`u`就是关节点。
	- **若`u`不是深度优先搜索树的根节点**
	    - 根据关节点的定义，若删除`u`后图会分裂成两个或多个连通分量，则`u`是关节点，这意味着存在`u`的一个子节点`v`，使得`v`及其子孙节点无法通过回边到达`u`的祖先节点（不包括`u`本身）。
	    - 因为`d[u]`是`u`的`dfs`序号，代表`u`及其祖先节点的访问顺序，`L[v]`是`v`能到达的最早顶点的`dfs`序号。若`L[v] ≥ d[u]`，说明`v`及其子孙节点能到达的最早顶点最晚就是`u`，无法到达`u`的祖先节点，所以删除`u`后，`v`所在的子树会与`u`的祖先节点所在部分分离，图会变成非连通图，因此`u`是关节点。反之，若不存在这样的子节点`v`使得`L[v] ≥ d[u]`，则说明每个子节点`v`及其子孙都能通过回边到达`u`的祖先节点，删除`u`后图仍然连通，`u`不是关节点。

### 示例解析

- **顶点 3**：
    - 发现时间 `d[3] = 3`。
    - 最早可达时间 `L[3] = 3`。
    - 子节点 5 的 `L[5] = 3`，满足 `L[5] ≥ d[3]`，因此顶点 3 是关节点。

### 结论

通过 DFS 遍历和 `L[v] ≥ d[u]` 的条件，可以有效识别图中的关节点和双连通分量。关节点的识别对于网络设计、图的可靠性分析等领域具有重要意义。