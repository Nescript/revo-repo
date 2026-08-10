在原本的控制框架里，每个周期的循环主要是：
状态量->控制算法->控制量

RL 的框架很类似：
观测（observation）-> 策略（policy） -> 动作（action）

基本可以一一对应

# 训练过程

我们将 采样+更新 的一个周期称为一轮 PPO 训练
1. 冻结当前 actor 为旧策略。
2. 在 N 个[[并行环境和一些相关名词]]中采样 T 步，收集 N×T 条状态转移。
3. 保存观测、动作、奖励、结束标志、旧对数概率与旧价值预测。
4. 计算 bootstrap、GAE 优势值与 critic 回报目标。
5. 将数据打乱并切成 mini-batch。
6. 前向计算当前动作对数概率、当前价值预测、裁剪目标、value loss 与熵项。
7. 反向传播并小幅更新 actor 与 critic；对同一 batch 重复多个 epoch。
8. 丢弃旧 rollout，用新 actor 重新采样。

# PPO

> John 对他的孩子 David 有一些期待，John 用期待对比 David 的实际行动来评估 David 的表现是好是坏，依此来指导 Lucy 调整 David 的学习方式。
> 但期待不能脱离实际，John 不能期望他四岁的 David 智商超越  Einstein，所以 John 也不断学习调整自己的期待，希望更加符合 David 的实际情况 

这里的 John 就是 critic 网络，而 David 就是 actor 网络 

rollout和策略周期：一个策略周期是一步（针对critic和actor计算的特定t），一个rollout包含一系列策略周期