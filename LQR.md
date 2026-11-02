对被控系统，我们有描述其动态特性的连续时间状态空间方程：

$$
\dot{x}=Ax+Bu
$$

进一步使用前向欧拉法，将其近似离散化为计算机便于处理的形式：

$$
\dot{x}(t)\approx\frac{x(t+\Delta t)-x(t)}{\Delta t}=Ax(t)+Bu(t)
$$

令 $x_{k+1}=x(t+\Delta t)$、$x_k=x(t)$、$u_k=u(t)$，可得

$$
x_{k+1}-x_k=\Delta tAx_k+\Delta tBu_k
$$

$$
x_{k+1}=(I+\Delta tA)x_k+\Delta tBu_k
$$

令

$$
A_d=I+\Delta tA,\qquad B_d=\Delta tB
$$

即可得到离散形式，由上一步状态推出下一步状态：

$$
x_{k+1}=A_dx_k+B_du_k
$$

这里采用的是前向欧拉近似，而不是连续系统的精确离散化。

现在我们来看 LQR。

LQR 叫作线性二次型调节器，它的目标是找到一个控制律，使代价函数取得最小值，并在相应条件下使闭环系统稳定。

这里的代价函数由[[二次型]]组成，包括状态偏差的惩罚和控制量的惩罚。对于从 $t=0$ 开始的无限时域 LQR，可写为

$$
J=\frac{1}{2}\int_{0}^{\infty}\left(x^TQx+u^TRu\right)dt
$$

如果考虑有限时域，控制开始于 $0$ 时刻，结束于 $T$ 时刻，则可以写成

$$
J=\frac{1}{2}x(T)^TQ_1x(T)+\frac{1}{2}\int_{0}^{T}\left(x(t)^TQx(t)+u(t)^TRu(t)\right)dt
$$

这里分成了过程中的惩罚和不参与积分的终端惩罚。

接下来给出有限时域离散 LQR 的推导过程。在离散情况下，若控制输入为 $u_0,\ldots,u_{N-1}$，代价函数为

$$
J=\frac{1}{2}x_N^TQ_1x_N+\frac{1}{2}\sum_{i=0}^{N-1}\left(x_i^TQx_i+u_i^TRu_i\right)
$$

首先定义价值函数 $V_k(x_k)$，表示从第 $k$ 步的状态 $x_k$ 出发到第 $N$ 步的最优代价。根据动态规划原理，

$$
V_k(x_k)=\min_{u_k}\left[\frac{1}{2}x_k^TQx_k+\frac{1}{2}u_k^TRu_k+V_{k+1}(x_{k+1})\right]
$$

其中

$$
x_{k+1}=A_dx_k+B_du_k
$$

终端时刻的价值函数为

$$
V_N(x_N)=\frac{1}{2}x_N^TQ_1x_N
$$

因此可令 $P_N=Q_1$。假设下一步的价值函数可写成

$$
V_{k+1}(x_{k+1})=\frac{1}{2}x_{k+1}^TP_{k+1}x_{k+1}
$$

将离散状态转移方程代入并展开，得到关于 $u_k$ 的代价表达式：

$$
\begin{aligned}
V_k(x_k)=\min_{u_k}\Bigg[&\frac{1}{2}x_k^T\left(Q+A_d^TP_{k+1}A_d\right)x_k\\
&+\frac{1}{2}u_k^T\left(R+B_d^TP_{k+1}B_d\right)u_k\\
&+u_k^TB_d^TP_{k+1}A_dx_k\Bigg].
\end{aligned}
$$

对 $u_k$ 求导，得到

$$
\frac{\partial V_k}{\partial u_k}=\left(R+B_d^TP_{k+1}B_d\right)u_k+B_d^TP_{k+1}A_dx_k
$$

当 $R\succ0$ 且 $P_{k+1}\succeq0$ 时，代价函数关于 $u_k$ 是严格凸的，因此令导数为零即可得到唯一的最小值：

$$
\left(R+B_d^TP_{k+1}B_d\right)u_k+B_d^TP_{k+1}A_dx_k=0
$$

$$
u_k=-\left(R+B_d^TP_{k+1}B_d\right)^{-1}B_d^TP_{k+1}A_dx_k
$$

令

$$
K_k=\left(R+B_d^TP_{k+1}B_d\right)^{-1}B_d^TP_{k+1}A_d
$$

就得到了有限时域的最优控制律：

$$
u_k=-K_kx_k
$$

有限时域中的反馈增益 $K_k$ 通常随时刻 $k$ 变化。把最优控制律代回价值函数后，所有的 $u_k$ 都可以转化为 $x_k$ 的表达式，价值函数仍然是关于 $x_k$ 的二次型：

$$
V_k(x_k)=\frac{1}{2}x_k^TP_kx_k
$$

这意味着，可以从终端时刻开始向前计算，并将上述推导延伸到 $0$ 至 $N$ 的每一步。
