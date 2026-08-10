一轮 PPO 训练指一次采样和更新的完整周期。

1. 冻结当前 actor 为旧策略。
2. 在 $N$ 个[[并行仿真]]环境中采样 $T$ 步，收集 $N \times T$ 条状态转移。
3. 保存观测、动作、奖励、结束标志、旧对数概率与旧价值预测。
4. 计算 bootstrap、GAE 优势值与 critic 回报目标。
5. 将数据打乱并切成 minibatch。
6. 前向计算当前动作对数概率、当前价值预测、裁剪目标、value loss 与熵项。
7. 反向传播并小幅更新 actor 与 critic；对同一 batch 重复多个 epoch。
8. 丢弃旧 rollout，用新 actor 重新采样。
