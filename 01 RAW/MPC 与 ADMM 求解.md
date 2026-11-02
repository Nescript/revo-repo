# MPC 与 ADMM 求解

> 学习材料草稿，供审阅。主题：模型预测控制（MPC）如何用 ADMM 求解，以及 tinyMPC 的实现思路。

## 1. MPC 到底是什么问题

模型预测控制（MPC）指在每个控制周期，基于当前状态 $x_0$，解一个有限时域最优控制问题，然后只执行第一步控制量 $u_0^*$；下一周期用新的状态重新求解（滚动时域，receding horizon）。

对线性系统 + 二次代价 + 线性约束，MPC 的在线问题是：

$$
\begin{aligned}
\min_{x_k,\, u_k} \quad & \sum_{k=0}^{N-1} \frac{1}{2}\left( x_k^\top Q x_k + u_k^\top R u_k \right) + \frac{1}{2} x_N^\top Q_f x_N \\
\text{s.t.} \quad & x_{k+1} = A x_k + B u_k && \text{动力学（等式约束）}\\
& x_0 = x_{\text{init}} \\
& \underline{x} \le x_k \le \bar{x}, \quad \underline{u} \le u_k \le \bar{u} && \text{状态/输入约束（不等式约束）}
\end{aligned}
$$

其中 $Q \succeq 0$、$R \succ 0$、$Q_f \succeq 0$ 是权重矩阵，$N$ 是预测时域。

这是一个 **QP**（二次规划）：二次目标 + 线性约束。通用 QP 求解器（如内点法）可以解，但每步要做矩阵分解，在嵌入式平台上太贵——这正是 ADMM 类方法的切入点。

## 2. 约束的"无穷代价"写法

约束可以改写成目标函数的一部分，工具是**指示函数**。对约束集合 $\mathcal{C}$（例如盒约束 $\underline{u} \le u \le \bar{u}$）：

$$
\mathcal{I}_{\mathcal{C}}(u) = \begin{cases} 0 & u \in \mathcal{C} \\ +\infty & u \notin \mathcal{C} \end{cases}
$$

于是"约束"变成了"超出约束代价为无穷大"，问题在形式上变成无约束：

$$
\min \; f(x, u) + \mathcal{I}_{\mathcal{X}}(z) + \mathcal{I}_{\mathcal{U}}(v)
$$

代价是：指示函数不光滑，不能用普通梯度法优化。ADMM 正是为这种"两个目标项分开处理"的结构设计的。

## 3. ADMM 的一般形式

ADMM（交替方向乘子法，Alternating Direction Method of Multipliers）求解如下结构的问题：

$$
\min_{w,\, z} \; f(w) + g(z) \quad \text{s.t.} \quad w = z
$$

定义**增广拉格朗日函数**（拉格朗日项 + 二次惩罚项，惩罚参数 $\rho > 0$）：

$$
\mathcal{L}_\rho(w, z, \lambda) = f(w) + g(z) + \lambda^\top (w - z) + \frac{\rho}{2}\|w - z\|^2
$$

用缩放形式（scaled form，令 $y = \lambda / \rho$）写出三步交替迭代：

$$
\begin{aligned}
w^{k+1} &= \arg\min_w \; f(w) + \frac{\rho}{2}\|w - z^k + y^k\|^2 \\
z^{k+1} &= \arg\min_z \; g(z) + \frac{\rho}{2}\|w^{k+1} - z + y^k\|^2 \\
y^{k+1} &= y^k + (w^{k+1} - z^{k+1})
\end{aligned}
$$

直觉：$w$ 和 $z$ 是同一个决策量的两份拷贝，各自只面对一个目标项；对偶变量 $y$ 不断累积两者的分歧，二次惩罚项把两份拷贝"拉"到一起。收敛时 $w = z$，既满足 $f$ 的结构，又满足 $g$ 中的约束。

## 4. 套到 MPC 上：三步各自变成什么

这是核心。令 $w = (x, u)$ 负责**满足动力学**，$z, v$ 是负责**满足状态/输入约束**的拷贝：

$$
\min \; \underbrace{\sum_k \frac{1}{2}\left(x_k^\top Q x_k + u_k^\top R u_k\right)}_{f：只含二次代价与动力学等式约束} + \; \underbrace{\mathcal{I}_{\mathcal{X}}(z) + \mathcal{I}_{\mathcal{U}}(v)}_{g：只含不等式约束} \quad \text{s.t.} \quad x = z,\; u = v
$$

三步迭代分别变成：

**第一步（primal 更新）= 解一个 LQR 问题。** 因为 $f$ 里只有二次代价和线性动力学，加上 ADMM 的二次惩罚项后，整个子问题仍然是一个**无不等式约束的线性二次最优控制问题**——正是 LQR（只是代价里多了由 $z^k, y^k$ 产生的线性修正项）。所以可以用 Riccati 递归高效求解：backward pass 算增益，forward pass 做 rollout。这就是"第一步类似求解 LQR"的含义。

**第二步（约束更新）= 向约束集合投影。** 对指示函数做最小化，结果就是投影算子；对盒约束，投影退化成逐元素 clamp：

$$
z^{k+1} = \mathrm{clip}\left(x^{k+1} + y^k,\; \underline{x},\; \bar{x}\right), \qquad
v^{k+1} = \mathrm{clip}\left(u^{k+1} + g^k,\; \underline{u},\; \bar{u}\right)
$$

**第三步（对偶更新）= 协调。** 对偶变量累积两份拷贝的分歧：

$$
y \leftarrow y + (x - z), \qquad g \leftarrow g + (u - v)
$$

整体图景：LQR 给出一条满足动力学但可能违反约束的轨迹 → 投影把它拉回约束内（但可能破坏动力学）→ 对偶变量记录分歧，修正下一轮 LQR 的代价 → 迭代直到两者一致。

**收敛判断**用原始残差和对偶残差：

$$
r_{\text{pri}} = \|x - z\|, \qquad r_{\text{dual}} = \rho \|z^{k} - z^{k-1}\|
$$

都小于阈值（如 $10^{-3} \sim 10^{-4}$）即停。

## 5. tinyMPC 为什么快

通用求解器 OSQP 也基于同样的 ADMM 框架。tinyMPC（Nguyen et al., CMU, 2024）的贡献在工程侧：

- **预计算 Riccati 矩阵**：动力学 $A, B$ 固定时，backward pass 里的大矩阵（反馈增益 $K$、cost-to-go 矩阵 $P$ 及相关分解）不随 ADMM 迭代变化，可以离线算一次存下来；在线每轮只做矩阵-向量乘法。
- **内存极小**：能跑在 Teensy 4.0、Crazyflie 的 STM32 这类 MCU 上，求解耗时毫秒级。
- **warm start**：用上一周期的解初始化 ADMM，实际只需很少迭代。

代价是：

- 模型 $A, B$ 或权重 $Q, R$ 变了，预计算就要重做（时变系统需要扩展或在线重算）；
- ADMM 本质是一阶方法，精度"够用"但不如内点法精确——对控制来说通常可以接受；
- $\rho$ 影响收敛速度，实践里常用自适应 $\rho$ 策略。

## 6. 轮腿机器人语境下的 MPC

对轮腿机器人，常见做法是：

- **模型**：用质心动力学（单刚体模型，SRBD），围绕当前状态/标称轨迹线性化得到 $A, B$（时变或定常）；
- **决策变量**：各接触足的反作用力 $f \in \mathbb{R}^3$（以及质心状态轨迹）；
- **约束**（全部是线性的，因此仍落在 QP + 投影框架内）：
  - 摩擦锥：$|f_x| \le \mu f_z,\; |f_y| \le \mu f_z$（锥近似为金字塔，即线性不等式）；
  - 法向力非负：$f_z \ge 0$（脚不能拉地），且有上限；
  - 关节力矩上限（盒约束）；
- 接触时序、摆动足轨迹由外层（步态规划）给定，MPC 只负责力分配。

## 7. 参考

- tinyMPC 论文：Nguyen et al., *TinyMPC: Model-Predictive Control on Resource-Constrained Platforms*, 2024. https://arxiv.org/abs/2310.16985
- tinyMPC 文档：https://tinympc.org/
- OSQP（通用 ADMM-QP 求解器）：https://osqp.org/docs/examples/mpc.html
- Boyd et al., *Distributed Optimization and Statistical Learning via the ADMM*（ADMM 经典综述）

## 待深入的方向

1. 推导第一步：把 ADMM 惩罚项展开，看 LQR 的 Riccati 递归具体被改成什么样；
2. 代码走一遍：用 Python 写一个 ADMM-MPC 最小实现，对照 OSQP 验证；
3. 读 tinyMPC 源码：看预计算和投影在 C++ 里怎么落地。
