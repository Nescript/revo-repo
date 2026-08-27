# ROSE² 论文阅读笔记：从二维 OccupancyGrid 提取房间

本文阅读对象是 Luperto 等人的 v1 论文《Robust Structure Identification and Room Segmentation of Cluttered Indoor Environments from Occupancy Grid Maps》。论文发表于 2022-03-07，作者为 Matteo Luperto、Tomasz Piotr Kucner、Andrea Tassi、Martin Magnusson 和 Francesco Amigoni。

论文原文：[arXiv HTML v1](https://arxiv.org/html/2203.03519v1)，[PDF v1](https://arxiv.org/pdf/2203.03519v1)。论文写作中的核心流程没有 `Algorithm` 编号，主流程也没有编号公式。本文按论文的 Section III、图 1 至图 10 和 Section IV 逐项引用。论文在 Section IV 给出了 IoU 表达式，v1 页面没有为它显示公式编号。

文中标签含义如下：

1. **论文明确**：可以在论文对应小节、图或表中直接找到。
2. **代码确认**：可以在当前仓库的源文件和符号中直接找到。
3. **推断或采用建议**：根据论文与当前 ROS 2 项目约束给出的工程判断。

## 1. 先给结论

ROSE² 的关键思路是先从占用栅格中找出建筑物级别的主方向，再把局部观测到的线段整理成跨房间共享的墙体，之后用这些墙体构造一个平面分割，把相邻的平面单元聚成房间。它把家具、玻璃反射、局部遮挡和部分探索带来的噪声放在全局方向与墙体连续性框架中处理。

论文的主数据流可以写成：

```text
M
  └─ ROSE：二维 DFT、径向频谱峰、逆 DFT、结构分数阈值
       └─ Ψ、\bar{M}
            └─ 概率 Hough 线段 S
                 └─ 角度聚类、空间聚类、投影对齐、近邻墙合并
                      └─ 墙体簇 W、代表线 L、面 F、边 E、边权 w(e)
                           └─ DBSCAN 聚类面
                                └─ 房间多边形 r_i、楼层平面 \mathcal{F}
                                     └─ 给原始地图的空闲单元赋房间标签 \check{M}
```

在两侧墙体都没有被观测到的局部区域，论文还提供一条补充路径：计算 Voronoi 拓扑图，检查一个房间对应的图是否连通，必要时沿代表线或主方向添加分隔线。[论文 Section III 总览](https://arxiv.org/html/2203.03519v1#S3)，[图 1，PDF 第 1 页](https://arxiv.org/pdf/2203.03519v1#page=1)。

## 2. 研究问题、输入、输出和假设

### 2.1 研究问题

论文把任务定义为从二维占用地图中识别墙和房间，并把地图分成一组房间。动机来自室内机器人对定位、运动规划、任务规划和环境语义的需要。二维占用地图含有障碍概率，却没有直接给出房间类型、墙体布局等高层信息。家具和可移动物体会遮挡墙，低位二维激光雷达还可能记录桌椅腿、袋子、镜面反射、法式窗和玻璃墙等非建筑成分。[论文摘要与 Section I](https://arxiv.org/html/2203.03519v1#S1)

### 2.2 输入

论文 Section III 规定输入是常规的分类二维占用栅格 $M$。单元属于三类之一：

| 类别 | 论文中的含义 | 对清扫系统的直接含义 |
| --- | --- | --- |
| occupied | 被障碍占据 | 不能作为房间地面 |
| free | 空闲单元 | 可参与房间标签候选 |
| unknown | 未知单元 | 需要保留未知状态，不能直接当作已确认地面 |

论文强调方法独立于机器人配置和构图所用 SLAM。它可以处理完整地图，也可以处理探索早期的部分地图。论文没有要求训练新模型，应用到新环境时无需预训练阶段。[论文 Section III 开头与 Section I](https://arxiv.org/html/2203.03519v1#S3)

### 2.3 环境假设

论文采用的核心先验是：人造室内环境的墙体通常沿数量有限的主方向 $\Psi$ 排列。这个先验用于频域结构提取和线段对齐。

“Manhattan world”常被用来描述墙体只沿两条互相垂直的轴排列的室内模型。论文明确写出 ROSE² 不依赖伪 Manhattan world 的形状假设，ROSE 也可以处理多于两条主方向。因此，$\Psi$ 是从地图中估计出来的方向集合，不能预先固定成水平和竖直两项。[论文 Section I；Section III-A](https://arxiv.org/html/2203.03519v1#S3.SS1)，[图 5 的非 Manhattan 地图结果，PDF 第 5 页](https://arxiv.org/pdf/2203.03519v1#page=5)

### 2.4 输出

论文 Section III-C 给出输出集合：

$$
\langle \bar{M}, L, F, \mathcal{F}, \check{M} \rangle
$$


各符号的角色如下：

| 符号 | 论文定义或作用 |
| --- | --- |
| $\bar{M}$ | 去除噪声和非建筑成分后的干净地图 |
| $\Psi$ | 主方向集合，来自频谱径向峰 |
| $S$ | 概率 Hough 变换得到的线段集合，$s\in S$ |
| $W$ | 方向相近且空间连续的局部墙体线段簇 |
| $\bar{W}$ | 将 $W$ 中线段投影到主方向之后的对齐线段簇 |
| $L$ | 代表整面墙方向和位置的代表线集合 |
| $F$ | 代表线交叉后形成的凸面集合，面之间共享边 |
| $E$ | 面之间的公共边集合 |
| $w(e)$ | 边上被观测墙线投影覆盖的比例，满覆盖约为 1，空区域约为 0 |
| $\mathcal{F}=\{r_1,\ldots,r_n\}$ | 合并后的房间多边形集合，也称几何楼层平面表示 |
| $\check{M}$ | 将原始地图中的空闲单元赋予对应房间后的分割地图 |

论文把 $\mathcal{F}$ 的边界同时用于补足尚未观测到的墙体。这个几何预测可以服务于部分地图的房间形状推断，论文在本文中主要用它做房间分割，其他用途以定性结果展示。[论文 Section III-C](https://arxiv.org/html/2203.03519v1#S3.SS3)

## 3. 方法逐步拆解

### 3.1 Stage A：地图清理与结构特征识别

#### A1. 把建筑墙和杂波分开

ROSE² 先调用作者此前的 ROSE 频率方法。目标是从 $M$ 的 occupied 单元中找出更可能属于墙体的单元，同时抑制家具、噪声、反射和其他非建筑成分，得到 $\bar M$。这一步处理的是“哪些占据单元像建筑结构”，还没有产生房间标签。[论文 Section III-A](https://arxiv.org/html/2203.03519v1#S3.SS1)

#### A2. 二维 DFT 和径向频谱峰

论文描述的频域流程如下：

1. 对占用地图 $M$ 计算二维离散傅里叶变换。
2. 在频谱图中观察穿过中心的径向脊线。墙体方向的重复排列会在这些方向上产生能量集中。
3. 沿每个候选方向 $\psi$ 累加频谱幅度。
4. 选出所有候选方向中最突出的峰，形成主方向集合 $\Psi$。
5. 保留这些方向附近的频谱分量，把其余频谱分量置零。
6. 对保留的频谱做逆 DFT。每个 occupied 单元因此获得一个“对环境结构的贡献分数”。
7. 丢弃低于阈值的单元，得到干净地图 $\bar M$。

论文把这一步描述为自动生成一个带通滤波器，目标是保留高能量结构分量。阈值通过尝试多个候选值，使清理后地图中的线段数与空闲栅格单元数之比落在实验确定的区间中，从而减少针对地图类型的手工参数调整。论文没有在本文中展开 ROSE 的 DFT 公式，细节指向参考文献 [14]。[论文 Section III-A，PDF 第 3 页](https://arxiv.org/pdf/2203.03519v1#page=3)，[ROSE 前作](https://arxiv.org/abs/2004.08794)

#### A3. 主方向的含义

主方向是频谱中建筑重复排列的方向模式。它不等同于某一条墙，也不等于房间的语义类别。一个主方向可以支持许多局部线段，一个墙体也可能因为遮挡而只留下几段观测。

频谱中的径向角还要转换成地图中的墙线角。**代码确认：** 当前 vendored ROSE 实现在 `process_map` 中先给峰值角加 $\pi/2$，再根据图像坐标方向换算，最后写入 `main_directions`。因此，读取频谱图时不能直接把峰值角当作 map-frame 中的墙体 yaw。[对应实现](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/rose_v1_repo/fft_structure_extraction.py)

一个常见误读是把 ROSE² 当作只适用于直角房间。论文在 Section III-A 明确说明 ROSE 可以处理多于两条主方向，实验中的 Figure 5 还包含非 Manhattan 平面。工程上应把 $\Psi$ 当作连续角度集合来处理，保留斜墙的可能性。[论文 Section III-A 与图 5](https://arxiv.org/html/2203.03519v1#S3.SS1)

#### A4. 当前仓库的对应代码

**代码确认：** `oomwoo_rose2` 的 [engine.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/engine.py) 中，`Rose2Segmenter.segment` 先调用 `_source_image`，再创建 `FFTStructureExtraction`，依次调用 `process_map`、`simple_filter_map`、`generate_initial_hypothesis_simple` 和 `find_walls_flood_filing`。`rose.main_directions` 是方向列表，`rose.analysed_map` 是清理结果。具体符号位置可见 `Rose2Segmenter.segment`、`Rose2Segmenter._source_image`。

**代码确认：** [upstream/rose_v1_repo/fft_structure_extraction.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/rose_v1_repo/fft_structure_extraction.py) 的 `FFTStructureExtraction.compute_fft` 使用 `fftshift(fft2(...))`，`process_map` 在极坐标频谱上累加幅度并选峰，随后通过逆 DFT 形成结构分数，`simple_filter_map` 按归一化分数阈值保留单元。

**代码确认：** 当前 `oomwoo_segmentation` 目录也有一套原生 Python 3 实现。其 [engine/fft.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation/oomwoo_segmentation/engine/fft.py) 中的 `FFTStructureExtraction` 具有相同的 `compute_fft`、`process_map` 和 `simple_filter_map` 阶段。两套实现的入口和参数集合并不完全相同，详见本文第 8 节的仓库状态说明。

#### A5. Flood fill 应放在哪里理解

论文 Section III-A 的正文只说明 ROSE 产生结构分数和清理地图，没有把 flood fill 列为房间分割步骤。

**代码确认：** 当前 vendored ROSE 文件的 `FFTStructureExtraction.find_walls_flood_filing` 使用 `skimage.segmentation.flood_fill`，对沿主方向生成的 slices 做连通标记、凸包和最小包围矩形计算。这是 ROSE 内部的中间处理步骤，服务于后续线段候选生成。

**代码确认：** 当前 `oomwoo_rose2.engine.Rose2Segmenter._geodesic_coverage` 还实现了一个 8 邻域多源 BFS，把房间多边形栅格化后未分配的 cleanable 单元传播到已有标签。这个覆盖补全层属于 OOMWOO 端口的后处理，论文正文没有这一步。

因此，阅读论文时可以记住：ROSE² 的房间判定依赖墙线、面、边权和面聚类。代码里出现的 flood fill 或 BFS 有各自的中间用途，不能把它们概括成论文的“房间 flood fill 算法”。

### 3.2 Stage B：墙体检测

#### B1. 从干净地图提取线段

对 $\bar M$ 运行概率 Hough line transform，得到线段集合 $S$。论文指出同一面墙经常表现为若干条轻微错位的小线段，因此单次 Hough 结果还不能直接当作墙体。[论文 Section III-B 与图 2](https://arxiv.org/html/2203.03519v1#S3.SS2)，[图 2，PDF 第 3 页](https://arxiv.org/pdf/2203.03519v1#page=3)

#### B2. 角度聚类与空间聚类

论文分两层聚类：

1. 先按角度系数相近聚类。
2. 再在方向相近的线段中，用 DBSCAN 将空间上连续、彼此接近的线段组成墙体簇 $W=\{s_1,\ldots,s_n\}$。

这里的空间连续性是关键。两段线即使各自很长，若它们在地图中相距较远，也没有足够理由视为同一面局部墙。[论文 Section III-B](https://arxiv.org/html/2203.03519v1#S3.SS2)

#### B3. 投影到主方向

对墙体簇中的每一条 $s$，寻找最近的主方向 $\psi\in\Psi$。以线段中点 $p$ 为通过点，把 $s$ 投影到方向为 $\psi$ 的直线上，得到对齐线段 $\bar s$。所有对齐后的线段组成 $\bar W$。

这个步骤把局部观测中的小角度误差压到建筑级方向上。它没有把所有环境强行旋转到水平竖直坐标，而是让每条线贴近该地图自己的方向集合 $\Psi$。[论文 Section III-B](https://arxiv.org/html/2203.03519v1#S3.SS2)

#### B4. 合并同一面墙的多个簇

同一面共享墙可能从不同房间被观察到，因而形成多个 $\bar W$。论文为每个簇取一个中心点 $P$：在垂直于主方向的方向上，使用该簇所有对齐线段中点的中位位置。两个墙簇对应的平行线 $l$ 和 $l'$ 若距离小于阈值，就合并为一个簇。论文用“接近门宽”帮助理解该阈值的几何意义，并没有在本文给出统一的米制数值。[论文 Section III-B 与图 2](https://arxiv.org/html/2203.03519v1#S3.SS2)

#### B5. 代表线、面、边

对每个合并后的墙簇 $W_k$，建立一条代表线 $l_k$。它的方向等于某个主方向 $\psi$，并通过该墙簇的中心点。代表线被延伸到整张地图，用于构造建筑级的几何平面表示。

代表线相交形成的凸区域叫作 face，记作 $f\in F$。两张相邻 face 共享的代表线片段叫作 edge，记作 $e\in E$。这一步将“很多局部观测线”变成一组可用于房间聚类的平面单元。[论文 Section III-B](https://arxiv.org/html/2203.03519v1#S3.SS2)

#### B6. 边权和代表线过滤

对每条边 $e$，把对应墙簇的线段投影到 $e$ 上，用投影覆盖比例定义 $w(e)$：

* 观测线覆盖边的大部分时，$w(e)\approx1$。
* 边位于没有观测墙线的区域时，$w(e)\approx0$。

论文进一步过滤代表线：若一条代表线全部边的累计覆盖低于经验阈值 0.1，就删除这条代表线，以抑制只由局部物体引起的线。被删除代表线中，仍保留覆盖比例很高的边，以避免丢掉有价值的局部结构。[论文 Section III-B](https://arxiv.org/html/2203.03519v1#S3.SS2)

#### B7. 当前仓库的对应代码

**代码确认：** [upstream/rose_v2_repo/minibatch.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/rose_v2_repo/minibatch.py) 的 `Minibatch.start_main` 依次调用：

1. `util.layout.start_canny_and_hough` 获取墙线。
2. `util.layout.cluster_ang`、`assign_orebro_direction` 做角度聚类和主方向对齐。
3. `get_wall_clusters`、`get_representatives`、`new_spatial_cluster` 做墙簇和代表线整理。
4. `extend_line` 以及 `object.ExtendedSegment.create_extended_segments` 延伸代表线。
5. `ExtendedSegment.merge_together` 合并近邻延伸线。
6. `Segment.set_weights` 计算边或延伸线的观测覆盖比例。
7. `Segment.remove_less_representatives` 按支持度阈值筛选。

关键函数位于 [upstream/util/layout.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/util/layout.py)、[upstream/object/Segment.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/object/Segment.py) 和 [upstream/object/ExtendedSegment.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/object/ExtendedSegment.py)。

**代码确认：** 当前端口的默认 `Rose2Config` 在 [engine.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/engine.py) 中把 `lines_threshold` 设为 0.22，把 `lines_distance_px` 设为 20，把 `edges_threshold` 设为 0。它还设置 `hard_wall_threshold=0.40`。这些数值来自当前端口的兼容配置，论文只明确给出代表线累计权重 0.1 的经验阈值。两者不能混写成论文参数。

### 3.3 Stage C：房间检测

#### C1. 先有平面单元，再有房间

代表线将空间切成一批 face。论文按两条规则聚类 face：

1. 两个相邻 face 的公共边对应墙体时，属于不同房间。
2. 两个相邻 face 的公共边没有墙体证据时，属于同一房间。

论文使用 DBSCAN，并把公共边 $e_{f,f'}$ 的权重 $w(e_{f,f'})$ 作为聚类距离的依据。高墙体支持度倾向于分开相邻 face，低墙体支持度倾向于合并相邻 face。聚类后的每个 $F_i$ 中的 face 被合并为一个房间多边形 $r_i$。房间多边形边界按论文假设对应外部墙体。[论文 Section III-C](https://arxiv.org/html/2203.03519v1#S3.SS3)

#### C2. 代码中的亲和矩阵

论文正文把矩阵构造细节指向此前工作 [16]，本文没有给出对应公式。当前仓库的移植代码可以直接确认下列实现细节。对两个 face $i,j$，代码在 [upstream/util/matrice.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/util/matrice.py) 的 `create_matrix_l` 中构造矩阵 $L$：

\[
L_{ij}=\begin{cases}
1, & i=j,\\
\exp\left(-w(e_{ij})/\sigma\right), & i,j\text{ 相邻且未触发硬墙阈值},\\
0, & i,j\text{ 不相邻，或 }w(e_{ij})\text{ 达到硬墙阈值}.
\end{cases}
\]

随后代码计算：

\[
D=\operatorname{diag}\left(\sum_jL_{ij}\right),\qquad
M_{\mathrm{aff}}=D^{-1}L,
\]

对 $M_{\mathrm{aff}}$ 做对称化，最终把

\[
X=1-M_{\mathrm{aff}}
\]

作为 `DBSCAN(metric="precomputed")` 的输入。这里的 $M_{\mathrm{aff}}$ 是代码中的亲和矩阵，和论文输入地图 $M$ 的含义不同。上述三组公式属于**代码确认**，不能称作本文论文的编号公式。

原生 Python 3 版本在 [engine/clustering.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation/oomwoo_segmentation/engine/clustering.py) 的 `create_matrices`、`dbscan_cluster_cells` 和 `merge_cells` 中实现相同的数据流。

#### C3. 楼层平面和部分观测

合并得到的房间多边形集合 $\mathcal F$ 一方面表达已观测墙体，另一方面用房间边界推断尚未观测到的墙体。论文把这个能力用于部分房间形状预测和地图补充的定性展示。最终将原始地图 $M$ 中的空闲单元赋予 $\mathcal F$ 中对应房间，得到 $\check M$。[论文 Section III-C 与图 1(e)](https://arxiv.org/html/2203.03519v1#S3.SS3)

### 3.4 Stage D：补足缺失结构

当一面分隔两个房间的墙在地图中两侧都没有被直接观测到时，代表线本身无法提供该分隔。论文采用 Voronoi 拓扑图 $G=(N,T)$：

1. $N$ 是图节点，$T$ 是图边。
2. 对一个候选房间 $r$，取属于它的节点集合 $N_r$，检查子图 $G_r$ 是否连通。
3. 如果 $G_r$ 不连通，就按分离后的连通分量拆成 $r'$ 和 $r''$。
4. 如果已有代表线 $l$ 可以把两组节点分开，就使用这条线。
5. 如果没有合适的代表线，就使用方向属于 $\Psi$ 的新线，把两组节点分开。

论文 Figure 3 依次展示代表线、Voronoi 图和加入分隔后的楼层平面。这里的 Voronoi 图承担拓扑连通性检查和缺失墙分隔任务。Section II 所说的 Voronoi-based room segmentation 是另一类直接用障碍物距离划分空间的基线，两者用途不同。[论文 Section II、Section III-D 与图 3](https://arxiv.org/html/2203.03519v1#S3.SS4)，[图 3，PDF 第 4 页](https://arxiv.org/pdf/2203.03519v1#page=4)

**代码确认：** `oomwoo_rose2` 的 [upstream/rose_v2_repo/minibatch.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/upstream/rose_v2_repo/minibatch.py) 在 `rooms_voronoi=True` 时导入 `util.voronoi`，调用 `compute_voronoi_graph` 和 `voronoi_segmentation`。当前 [config/rose2.yaml](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/config/rose2.yaml) 将 `rooms_voronoi` 设为 `false`，并提供 `voronoi_closeness`、`voronoi_blur`、`voronoi_iterations` 参数。因此，当前默认运行路径主要是代表线、face、边权和 DBSCAN，Voronoi 细化是可选路径。

## 4. 一个最小栅格例子

下面的地图只用于帮助建立数据流。它把墙画成 `#`，空闲地面画成 `.`，家具画成 `F`，未知画成 `?`。图中的数值和线段位置是示意，不能当作论文实验结果。

### 4.1 输入 $M$

```text
###############
#.....#.......#
#..F..#.......#
#.....#.......#
#.............#
#.....#.......#
#.....#.......#
#.....#.......#
###############
```

中间竖墙的第 5 行有一个门洞。家具 `F` 是占据单元，可能遮挡局部地面或墙线。真实 OccupancyGrid 还可以有 `unknown` 单元，示意图没有展开所有情况。

### 4.2 Stage A 的结果

频域分析从地图中观察到近似水平和竖直的重复方向，于是示意地得到：

```text
Ψ = {0°, 90°}
```

逆 DFT 得到每个 occupied 单元的结构分数。低分家具单元被清掉后，得到 `\bar M`。注意，这个步骤保留的是更像墙的占据证据，仍然不会直接产生“左房间”和“右房间”的标签。

### 4.3 Stage B 的结果

Hough 从 `\bar M` 提取若干段水平和竖直线：

```text
S = {s1, s2, s3, ...}
```

相同方向且空间相近的线段组成墙簇 $W$。每条线被投影到对应的 $\psi$，然后合并为贯穿地图的代表线。代表线交叉后得到左侧 face、右侧 face，以及它们共享的竖直边。

中间代表线对应的公共边大部分受到墙体观测支持，门洞只减少其中一小段覆盖，因此整条边的示意权重仍然较高：

```text
中间公共边：w(e) 较高
房间内部的伪分割边：w(e) 接近 0
```

### 4.4 Stage C 的结果

DBSCAN 读取相邻 face 之间的边权。中间公共边具有较高的总体墙体支持度，因此两侧 face 保持分离。房间内部若被低支持度代表线切出多个小 face，这些小 face 会重新合并。示例最终可以得到：

```text
左侧 face 集合  -> r1 -> label 1
右侧 face 集合  -> r2 -> label 2
```

然后把空闲单元映射到房间多边形，形成 `\check M`。如果两个小 face 之间没有墙体证据，它们会合并成同一个 $r_i$。门洞只会降低公共边的覆盖率，论文没有把门检测列为独立输出。如果一条未观测到的墙使某个候选房间对应的 Voronoi 子图断开，Stage D 会按连通分量拆分。

这个例子体现了三个容易混淆的层次：

1. `\bar M` 是清理后的墙体证据图。
2. $F$ 是由代表线切出的几何面。
3. $\mathcal F$ 和 `\check M` 才是房间多边形及房间标签结果。

## 5. 实验和指标

### 5.1 指标定义

论文把预测房间记作 $\hat r$，人工标注或楼层平面得到的对应房间记作 $r$。对每个预测房间，寻找与它重叠面积最大的真值房间，然后计算 IoU。论文的表达式为：

\[
\operatorname{IoU}(\hat r,r)=\frac{\hat r\cap r}{\hat r\cup r}.
\]

按论文文字，这里的交、并按区域面积理解，并把 IoU 缩放到 0 至 100。论文还定义：

\[
\operatorname{precision}=\frac{\text{预测房间与最佳真值房间的最大重叠面积}}{\text{预测房间面积}},
\]

\[
\operatorname{recall}=\frac{\text{真值房间与最佳预测房间的最大重叠面积}}{\text{真值房间面积}}.
\]

论文提醒 precision 和 recall 对欠分割、过分割有互补偏差。它把 IoU 作为更重要的指标，因为 IoU 对这类偏差的敏感方式更平衡。[论文 Section IV](https://arxiv.org/html/2203.03519v1#S4)

### 5.2 10 张杂乱完整地图

Table I 汇总 10 张完全探索的真实地图，所有方法不因地图类型改变参数。括号内是标准差。

| 方法 | precision | recall | IoU |
| --- | ---: | ---: | ---: |
| ROSE² | 89.02 (8.39) | 93.93 (4.21) | 73.30 (17.83) |
| Morph | 84.98 (6.12) | 82.30 (11.16) | 51.24 (12.25) |
| Dist | 88.34 (7.85) | 79.79 (12.15) | 54.65 (14.73) |
| Voronoi | 85.86 (9.84) | 71.92 (13.25) | 28.65 (10.55) |

论文把这组结果用于支持 ROSE² 在杂乱真实地图上取得更高的平均分割质量。[Table I，论文 Section IV-A](https://arxiv.org/html/2203.03519v1#S4.SS1)，[PDF 第 5 页](https://arxiv.org/pdf/2203.03519v1#page=5)

### 5.3 图 4、图 5、图 6 的含义

| 图 | 场景 | ROSE² IoU | 对比方法 IoU | 阅读重点 |
| --- | --- | ---: | --- | --- |
| 图 4 | 论文图 1 的杂乱地图 | 95.07 | Voronoi 27.34，Morph 67.32，Dist 63.39 | 清理杂波后，分割边界更贴近房间布局 |
| 图 5 | 非 Manhattan 平面 | 80.44 | Voronoi 44.22，Morph 55.65，Dist 52.39 | 论文用它说明方法可以处理斜向或多方向布局 |
| 图 6 | 存在玻璃墙等感知伪影 | 38.15 | Voronoi 10.23，Morph 33.03，Dist 39.31 | ROSE² 在困难地图上也可能只有中低 IoU，方法没有消除所有地图伪影 |

来源：[图 4](https://arxiv.org/pdf/2203.03519v1#page=5)、[图 5 和图 6](https://arxiv.org/pdf/2203.03519v1#page=5)。

### 5.4 Survey 基准地图

论文还测试了房间分割综述使用的 20 张干净地图和 20 张加入人工几何噪声的地图。作者称这些地图比真实办公地图简单。

1. 有家具或人工噪声的地图：precision 为 93.54 (5.46)，recall 为 91.03 (3.46)。
2. 无家具地图：precision 为 93.26 (6.40)，recall 为 97.58 (2.20)。
3. 两类地图均未改变参数。

论文在此处没有在正文列出对应 IoU 汇总，完整结果指向代码仓库。[论文 Section IV-A](https://arxiv.org/html/2203.03519v1#S4.SS1)

### 5.5 部分地图和探索覆盖率

部分地图来自 Freiburg Building 79 和 University of Bremen Cartesium 的 GMapping 运行。作者用最终地图估计从探索开始到完整地图的覆盖比例，观察不同覆盖率下的 IoU。

1. Figure 7 的 FR79 示例：ROSE² IoU 80.98，Voronoi 33.10，Morph 66.47，Dist 74.62。
2. Figure 8 展示 FR79 和 Cartesium 的 IoU 随覆盖率变化，论文描述 ROSE² 在不同覆盖率下大约保持在 80 附近。
3. Figure 9 的 INTEL 部分地图：IoU 75.98。
4. Figure 10 的两个部分地图：FR79 IoU 90.42，Cartesium IoU 74.63。

这些结果支持论文关于“只观测到少量墙体时仍可生成有意义楼层平面”的判断。它们没有给出所有环境、传感器和机器人平台上的保证。[论文 Section IV-B、IV-C](https://arxiv.org/html/2203.03519v1#S4.SS2)，[图 7、图 8，PDF 第 6 页](https://arxiv.org/pdf/2203.03519v1#page=6)

### 5.6 论文证明了什么

根据论文自己的数据，可以得到以下范围内的结论：

1. 在作者选取的 10 张杂乱完整地图上，ROSE² 的平均 IoU 高于三种对比方法。
2. 在给出的非 Manhattan 示例上，ROSE² 获得更高 IoU，说明运行时不要求全部墙体只沿两条互相垂直的方向排列。
3. 在部分探索地图上，结构线和楼层平面能帮助保持较稳定的分割。
4. 结构清理、墙体对齐、边权和面聚类共同带来可解释的几何中间结果。

### 5.7 论文没有证明什么

以下事项不能从论文结果直接推出：

1. 没有针对 ROS 2 `nav_msgs/msg/OccupancyGrid` 消息、OOMWOO 地图、Nav2 或具体 2D LiDAR 型号的端到端验证。
2. 没有给出机器人目标硬件上的延迟、峰值内存或实时性曲线。
3. 没有给出清扫机器人 footprint、门宽安全余量、未知单元策略或跨墙标签安全检查。
4. 没有给出房间名称、厨房卧室等语义类别。
5. 没有证明动态家具长期变化下的稳定更新。论文结尾把变化环境中的长期建图列为后续工作。[论文 Section V](https://arxiv.org/html/2203.03519v1#S5)
6. 图 6 的 IoU 为 38.15，表明感知伪影严重时仍会出现明显误差。
7. 论文比较的是其选定数据集和三种基线，不能把 Table I 直接当作所有方法和所有地图上的统一排行榜。

论文在 v1 中把复现实验代码链接到 [goldleaf3i/declutter-reconstruct](https://github.com/goldleaf3i/declutter-reconstruct)。该仓库 README 还提示旧代码可能需要代码级调整。作者后来在 [aislabunimi/ROSE2](https://github.com/aislabunimi/ROSE2) 提供了 ROS 集成版本，并说明机器人使用可参考新版本，论文结果复现应参考旧仓库。两者的用途与代码状态需要分开记录。

## 6. 当前 oomwoo 源码映射

下面只记录当前检出文件中可以直接确认的符号。映射描述的是数据流对应关系，不代表仓库已经完成论文级逐阶段数值复现。

| 论文阶段 | 当前 `oomwoo_rose2` 代码 | 当前 `oomwoo_segmentation` 代码 | 代码确认内容 |
| --- | --- | --- | --- |
| OccupancyGrid 输入与 cleanable mask | [engine.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/engine.py)：`Rose2Segmenter.segment`、`_source_image`；[node.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/node.py)：`Rose2SegmentationNode._execute` | [engine.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation/oomwoo_segmentation/engine/engine.py)：`SegmentationEngine.segment`、`_source_image`；[node.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation/oomwoo_segmentation/node.py)：`SegmentationNode._execute` | 两个入口都把 OccupancyGrid 转成栅格数组，并把不可清扫单元排除。两套节点使用的 ROS action/message 包存在差异，见第 8 节。 |
| ROSE 频域清理 | vendored `upstream/rose_v1_repo/fft_structure_extraction.py`：`FFTStructureExtraction.process_map`、`simple_filter_map` | [engine/fft.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation/oomwoo_segmentation/engine/fft.py)：`FFTStructureExtraction.process_map`、`simple_filter_map` | 都直接实现 DFT、主方向提取、结构分数和阈值清理。 |
| ROSE slices 的 flood fill | vendored `upstream/rose_v1_repo/fft_structure_extraction.py`：`find_walls_flood_filing` | `engine/fft.py`：同名函数当前为空实现 | flood fill 出现在 vendored ROSE 中。原生版本保留调用阶段，但函数体没有对应的标记逻辑。 |
| Hough 线段 | `upstream/rose_v2_repo/minibatch.py`：`Minibatch.start_main`；`upstream/util/layout.py`：`start_canny_and_hough` | [engine/geometry.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation/oomwoo_segmentation/engine/geometry.py)：`start_canny_and_hough` | 从清理后的地图提取概率 Hough 线段。 |
| 角度和空间聚类 | `upstream/util/layout.py`：`cluster_ang`、`get_wall_clusters`、`get_representatives`、`new_spatial_cluster` | `engine/geometry.py`：`cluster_ang`、`get_wall_clusters`、`get_representatives`、`new_spatial_cluster` | 把方向相近、空间连续的线段整理为墙簇。 |
| 对齐和延伸 | `upstream/util/layout.py`：`assign_orebro_direction`、`extend_line`；`upstream/object/Line.py`：`create_extended_lines`；`upstream/object/ExtendedSegment.py`：`create_extended_segments` | `engine/geometry.py`：`assign_orebro_direction`、`create_extended_lines`、`create_extended_segments` | 产生沿主方向的延伸线段。 |
| 合并与边权 | `upstream/object/ExtendedSegment.py`：`merge_together`；`upstream/object/Segment.py`：`set_weights`、`remove_less_representatives` | `engine/geometry.py`：`merge_together`、`set_weights`、`remove_less_representatives` | 合并接近的墙线，计算投影覆盖支持度并筛选。 |
| 面和边 | `upstream/object/Segment.py`：`create_edges`；`upstream/object/Surface.py`：`create_cells`；`upstream/util/layout.py`：`classification_surface` | `engine/geometry.py`：`create_edges`、`create_cells`；`engine/clustering.py`：`classification_surface` | 交点切分边，构造闭合平面单元，并按外部轮廓分类。 |
| 亲和矩阵和 DBSCAN | `upstream/util/matrice.py`：`create_matrix_l`、`create_matrix_d`；`upstream/util/layout.py`：`create_matrices`、`DB_scan`、`create_space` | `engine/clustering.py`：`create_matrices`、`dbscan_cluster_cells`、`merge_cells` | 使用边权形成预计算距离矩阵，聚类 face 并合并为 Shapely 房间多边形。 |
| 房间栅格化 | `engine.py`：`Rose2Segmenter._rasterize_rooms` | `engine/postprocessing.py`：`rasterize_rooms` | Shapely 房间多边形转换为 int32 标签栅格。 |
| OOMWOO 覆盖补全 | `engine.py`：`_geodesic_coverage` | `engine/postprocessing.py`：`geodesic_coverage` | 使用连接组件和多源 BFS 给 cleanable 单元补标签。这是端口后处理，论文没有描述。 |
| 墙体输出 | `engine.py`：`_extract_walls` | `engine/postprocessing.py`：`extract_walls` | 将保留的延伸线裁剪到地图边界并转成 map-frame `WallSegment`。 |
| 可选 Voronoi | `upstream/rose_v2_repo/minibatch.py`：`rooms_voronoi` 分支；`upstream/util/voronoi.py`：`compute_voronoi_graph`、`voronoi_segmentation` | 当前 `SegmentationEngine` 没有 Voronoi 分支 | 当前 ROSE2 provider 有可选路径，默认配置关闭。 |

## 7. 参数、敏感性和容易误读之处

### 7.1 论文给出的参数信息

论文明确给出的代表线累计边权过滤阈值是 0.1。墙簇合并阈值只给出“接近门宽”的几何解释，ROSE 清理阈值通过线段数与空闲单元比例自动调节。论文强调跨地图类型运行时无需为完整或部分地图、仿真或真实地图分别手工调参。[论文 Section III-A、III-B](https://arxiv.org/html/2203.03519v1#S3)

### 7.2 当前代码参数

当前 `oomwoo_rose2.Rose2Config` 和 [config/rose2.yaml](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/config/rose2.yaml) 直接可见：

| 参数 | 当前默认值 | 作用和敏感点 |
| --- | ---: | --- |
| `filter_level` | 0.18 | 结构分数阈值。过高会漏掉弱墙，过低会保留杂波。 |
| `fft_peak_height` | 0.20 | 频谱峰显著性控制。峰太少会导致方向不足，峰太多会引入杂方向。 |
| `fft_band_width` | 50 | 频域方向带宽。它影响方向证据覆盖范围。 |
| `spatial_clustering_line_segments_threshold` | 5 px | 空间上相近线段的聚类距离。 |
| `lines_threshold` | 0.22 | 当前端口的延伸墙线支持度门槛，属于兼容配置。 |
| `lines_distance_px` | 20 px | 延伸墙线合并距离。应根据地图分辨率换算门宽和墙厚。 |
| `edges_threshold` | 0.0 | 当前端口保留边的最低边权门槛。 |
| `hard_wall_threshold` | 0.40 | 当前端口在面亲和矩阵中阻止跨高支持度墙合并。 |
| `min_component_size` | 10 px | 当前端口 BFS 后处理中的小连通分量过滤。 |
| `enable_geodesic_coverage` | true | 当前端口是否执行 cleanable 覆盖补全。 |
| `rooms_voronoi` | false | 是否启用可选 Voronoi 细化。 |

这些数值来自当前端口代码，论文实验表格不能直接证明它们在 OOMWOO 地图上最优。参数以像素计量时，地图分辨率改变会改变几何意义。部署前应把墙厚、门宽、机器人 footprint 与 resolution 的关系纳入测试。

### 7.3 容易误读的地方

1. **主方向不等于 Manhattan 假设。** 论文允许多于两条方向，也报告非 Manhattan 地图。
2. **主方向不等于墙线。** 它是频谱中方向模式，具体墙体要经过 Hough、聚类、投影、合并和边权计算。
3. **$\bar M$ 不等于房间标签。** 它只是更偏向建筑墙的清理图。
4. **face 不等于 room。** face 是代表线切出的原子平面单元，多个 face 可合并成一个 room。
5. **代表线不等于完整观测墙。** 它会跨越地图延伸，用于预测和分割，观测支持由 $w(e)$ 表示。
6. **Voronoi 在论文中承担缺失结构的拓扑检查。** Section III-C 的主路线是 face、边权和 DBSCAN。Section II 还把 Voronoi-based 方法列作对比基线。
7. **论文没有把 flood fill 定义成房间分割主算法。** 当前 vendored ROSE 的 flood fill 用于 slices 连通标记，当前 OOMWOO BFS 用于栅格覆盖补全。
8. **房间多边形可能延伸到未观测区域。** 论文把这看作楼层平面和形状预测能力。清扫系统需要额外检查未知单元、障碍、机器人 footprint 和可达性。
9. **论文的“empty cells”不能自动解释成所有未知单元。** OOMWOO 应把 free、occupied、unknown 保持为三种状态，只有已确认 free 且满足清扫约束的单元进入候选区域。
10. **当前端口加入了论文未描述的兼容层。** 包括硬墙阈值、结构栅格支持修复、frame fringe 过滤、BFS 覆盖补全和 canonical label 验证。它们会影响论文代码和当前结果之间的逐像素一致性。

## 8. 对 2D LiDAR ROS2 OccupancyGrid 的采用建议

以下内容属于**推断或采用建议**，论文没有替 OOMWOO 定义 ROS 2 接口。

### 8.1 先固定输入语义

1. 输入使用 `nav_msgs/msg/OccupancyGrid`，保留 resolution、width、height、origin、frame 和 row-major cell data。
2. 把 occupied、free、unknown 分开保存。unknown 的默认策略应是“保留并排除清扫”，直到人工确认。
3. 将机器人 footprint、最小门宽、膨胀距离转换到像素前，记录地图分辨率和参数版本。
4. 对图像行列顺序做显式测试。论文没有规定 ROS 图像翻转规则，当前仓库的 `SourceMap` 与地图 I/O 负责这一层。

### 8.2 把几何结果和任务语义分开

建议保留四种独立结果：

1. source-grid 的房间 label grid，作为每个单元的权威分区结果。
2. 房间多边形，用于显示和导出。
3. Detected Wall 列表，用于人工审核和后续生成 Virtual Wall 候选。
4. Nav2 keepout mask，单独表达导航禁区，不能把房间 ID 写进 keepout mask。

当前共享 action 的 [SegmentRooms.action](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation_interfaces/action/SegmentRooms.action) 定义了 OccupancyGrid 输入、可选 cleanable mask、标签、房间元数据、检测墙和诊断图。当前 `oomwoo_rose2` 节点在 [node.py](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2/oomwoo_rose2/node.py) 的 `Rose2SegmentationNode._execute` 中提供 ROS 2 action 适配。

### 8.3 把自动分割当作候选生成

论文展示了较好的平均 IoU，但 Figure 6 和部分地图结果表明困难地图仍会有较大误差。建议流程为：

```text
OccupancyGrid
  -> ROSE² 候选分割
  -> 显示清理图、代表墙、房间多边形、标签图和未分配单元
  -> 检查跨墙标签、unknown 标签、footprint 可达性、房间连通性
  -> 人工编辑与确认
  -> Published Region Set
  -> 清扫任务
```

当前仓库的 Phase 1 开发上下文要求 `label 0` 表示未分配，正整数表示候选房间，并把未分配 free 单元呈现给用户。可参考 [docs/DEVELOPMENT.md](../../ros_ws/src/oomwoo-cleaning-jobs/docs/DEVELOPMENT.md) 的自动分割、验证和清扫边界说明。

### 8.4 用项目地图重新评估

建议至少记录：

1. 房间 IoU、precision、recall，与论文定义保持一致。
2. wall overlap，标签跨越高支持墙的单元数。
3. free 单元未分配比例。
4. unknown 被赋予清扫标签的数量，目标为 0。
5. 每个房间 footprint 膨胀或腐蚀后的可达连通分量。
6. 地图分辨率、origin yaw、地图哈希、参数和实现版本。
7. 单次处理时间和峰值内存。

这些附加指标是面向清扫安全和 ROS2 工程的建议，论文只直接报告了房间分割指标和定性几何结果。

### 8.5 Voronoi 与 flood fill 的采用边界

当前 `oomwoo_rose2` 默认关闭 `rooms_voronoi`。若要启用，应单独测量可选依赖、处理时间和与默认 DBSCAN 结果的差异。当前 BFS 覆盖补全也应单独验证它是否跨越了真实墙、虚拟墙和未知单元。验证结果需要写入固定地图回归，而不能只看房间数量。

## 9. 当前仓库状态和需要主线程决定的事项

当前检出内容同时存在两套可见的算法入口：

1. `src/oomwoo_rose2` 包含 GPLv3 的 vendored ROSE + ROSE2 端口，使用 `oomwoo_segmentation_interfaces/action/SegmentRooms`。
2. `src/oomwoo_segmentation/oomwoo_segmentation/engine` 也包含原生 Python 3 的 FFT、Hough、几何、DBSCAN 和后处理实现，`oomwoo_segmentation/node.py` 还提供自己的 action server。
3. [src/oomwoo-cleaning-jobs/docs/DEVELOPMENT.md](../../ros_ws/src/oomwoo-cleaning-jobs/docs/DEVELOPMENT.md) 将 `oomwoo_segmentation` 描述为算法中立工具，并把 `oomwoo_rose2` 写成当前生产 provider。
4. 两个 action server 使用的消息包和输入字段存在差异：ROSE2 路径使用 `oomwoo_segmentation_interfaces` 的 `MaskGrid`，原生路径使用 `oomwoo_segmentation_msgs` 的 `Image` mask 适配。

这是当前仓库事实之间的待决差异。本文把两套代码中可以直接确认的共同阶段都列入映射，没有把原生路径写成当前唯一公共接口。主线程需要决定保留哪一条 action/provider 路径，并同步更新开发上下文与测试入口。

## 10. 一页式复习

1. 输入是有 free、occupied、unknown 三类单元的二维占用栅格 $M$。
2. ROSE 用二维 DFT 观察频谱径向峰，得到主方向 $\Psi$ 和结构分数。
3. 按结构分数生成清理地图 $\bar M$，降低家具和局部伪影对墙检测的影响。
4. 概率 Hough 得到线段集合 $S$。
5. 角度聚类、空间聚类和主方向投影把线段整理成墙簇 $W$、对齐簇 $\bar W$。
6. 近邻共线墙簇合并为代表线 $L$，代表线相交形成 face $F$ 和边 $E$。
7. 边权 $w(e)$ 表示观测墙线对公共边的支持度，支持度高的边分隔房间。
8. DBSCAN 按边权聚类 face，Shapely 合并得到房间多边形 $\mathcal F$。
9. 将原始 free 单元映射到房间，得到标签图 $\check M$。
10. 缺失墙导致拓扑不连通时，Voronoi 图帮助拆分候选房间，并沿已有或主方向线补足分隔。
11. 论文证明了选定地图集上的分割效果与部分地图稳定性，尚未证明 OOMWOO 清扫安全、ROS2 实时性和动态环境长期稳定性。
12. 在 OOMWOO 中应把 ROSE² 输出当作候选 Region Set，保留 unknown、未分配单元、墙体支持度和版本信息，并经过几何与人工审核。

## 11. 一手来源

1. [论文 HTML v1](https://arxiv.org/html/2203.03519v1)
2. [论文 PDF v1](https://arxiv.org/pdf/2203.03519v1)
3. [ROSE 频域结构提取前作](https://arxiv.org/abs/2004.08794)
4. [论文复现实验代码：goldleaf3i/declutter-reconstruct](https://github.com/goldleaf3i/declutter-reconstruct)
5. [作者 ROS 集成代码：aislabunimi/ROSE2](https://github.com/aislabunimi/ROSE2)
6. [当前仓库 ROSE2 端口](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_rose2)
7. [当前仓库共享 SegmentRooms action](../../ros_ws/src/oomwoo-cleaning-jobs/src/oomwoo_segmentation_interfaces/action/SegmentRooms.action)
