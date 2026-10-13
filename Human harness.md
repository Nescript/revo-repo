Done:
改了defination部分的,现在除了Matrix提示没有直接头文件包含,其他无异常
整理老代码作为baseline
 - 继续整理直到clangd没有警告和报错
 - (基本搞定了,matrix是clangd的问题不管了)
 学习firefox写AGENTS.

Current:


Goal
- 用自己的话重述控制逻辑

Next
通过状态机打印验证跳跃逻辑
将老代码移植Mujoco仿真,验证跳跃逻辑
- Mujoco搭建完整的包含弹簧的仿真逻辑
	- 如何复用?不完全依赖AI?
- 代码移植

Queuemd
确定controller内部acc_z的符号
修复跳跃日志打印器和切换逻辑修复
- 日志打印器刷屏
- 修复仿真跳跃逻辑
支持力解算部分不整个传入state,只传入用到的变量

Future 
My own skill


BreakPoint:
		添加到codex会自动打开终端,很烦
		what is precommit
		支持力结算part done
		