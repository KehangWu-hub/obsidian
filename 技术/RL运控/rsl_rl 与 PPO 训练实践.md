---
tags:
  - 强化学习
  - PPO
  - rsl_rl
date: 2026-08-17
---

# rsl_rl

## 1. 简介

`rsl_rl` 是一个基于 PyTorch 实现的强化学习库，由 ETH Zurich 的 RSL 实验室开发。它专门为同步的大规模并行采样而设计，通常作为 **Isaac Gym** 或 **Isaac Lab** 的后端算法库。

## 2. 核心特点

-   **极致的训练速度**：通过 GPU 端的向量化环境采样，可以在几十分钟内完成传统 RL 需要几天才能完成的训练量。
-   **PPO 算法优化**：内置了高度优化的 **PPO (Proximal Policy Optimization)** 算法，非常适合处理高维连续动作空间（如足式机器人的关节电机控制）。
-   **轻量化架构**：相比于 Stable Baselines3 (SB3) 或 Ray Rllib，它的代码结构非常精简，开发者可以轻松修改网络结构或 Loss 函数。
-   **足式机器人适配**：内置了处理刚体动力学任务中常见的观察值、特权信息（Privileged Information）和周期性奖励函数的逻辑。

## 3. 算法原理：PPO

`rsl_rl` 主要实现的是 Actor-Critic 架构的 PPO 算法。其目标函数公式如下：

$$J^{CLIP}(\theta) = \hat{\mathbb{E}}_t \left[ \min(r_t(\theta) \hat{A}_t, \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon) \hat{A}_t) \right]$$

其中：

- $r_t(\theta)$ 是新旧策略的概率比。
- $\hat{A}_t$ 是优势函数（Advantage Function）。
- $\epsilon$ 是裁剪超参数，防止策略更新步长过大。

## 4. 关键文件结构

在使用 `rsl_rl` 时，你通常会接触到以下核心类：

-   `OnPolicyRunner`: 负责管理整个训练循环（存储、评估、存盘）。
-   `PPO`: 算法核心，处理梯度更新和 Loss 计算。
-   `ActorCritic`: 定义神经网络架构（通常包括 Actor 网络和 Critic 网络）。
-   `RolloutStorage`: 在 GPU 上直接存储经验轨迹的数据结构。

## 5. 典型工作流

1.  **环境定义**：在 Isaac Gym/Lab 中定义机器人的 URDF、传感器和奖励函数。
2.  **配置参数**：通过 Python 的 `config` 类定义超参数（如 `learning_rate`, `num_steps_per_env`, `entropy_coef`）。
3.  **启动训练**：

    ```python
    from rsl_rl.runners import OnPolicyRunner
    # 初始化环境
    env = VecEnv(...)
    # 初始化训练器
    runner = OnPolicyRunner(env, train_cfg, log_dir)
    # 开始学习
    runner.learn(num_learning_iterations=1000, init_with_graceful_stop=True)
    ```

# 训练、监控与可视化流程

不要在带界面的情况下跑训练（极慢），标准操作分为三步：

## Step 1: 纯后台训练 (Train)

通过 --headless 剥离渲染，将 100% 的算力交给张量计算和物理引擎。

```Bash
# 激活环境并挂载 C++ 底层库隔离区或者其他配置
conda activate LeggedGym
export LD_LIBRARY_PATH=/home/wkh/lessons/legged_gym/isaac_libs:$LD_LIBRARY_PATH

# 无头模式启动训练
python legged_gym/scripts/train.py --task=a1 --num_envs=64 --headless
```

## Step 2: 数据监控 (Monitor)

代码会自动在 logs 目录下按时间戳存档。使用 TensorBoard 查看“大脑”发育情况。

```Bash
# 建议使用绝对路径启动，防止找不到数据
tensorboard --logdir=/home/wkh/lessons/legged_gym/logs
```

核心指标： Train/mean_reward（平均奖励，需稳步上升）和 Episode/length（存活时间，需逐渐变长）。注意： 数据非实时写入，需让程序运行几分钟后刷新网页。

## Step 3: 前台可视化验收 (Play)

加载最新训练好的模型权重（只做前向推理，算力压力极小），弹窗查看实际表现。

```Bash
# 运行前确保处于 Xorg 桌面环境
python legged_gym/scripts/play.py --task=a1
```

# Isaac Gym 与 PyTorch 的导入顺序

## import的顺序问题

物理引擎（Isaac Gym）和神经网络（PyTorch）都需要高强度使用 GPU。

如果先导入 torch： PyTorch 机制极其霸道，它会瞬间接管并锁死显卡的 CUDA 内存池。随后当 Isaac Gym 试图建立物理引擎时，会发现底层通道被强占，导致内存指针错乱，直接引发“段错误”或“CUBLAS 未初始化”报错。

如果先导入 isaacgym： Isaac Gym 会在显卡中安顿好渲染和计算通道，并将其设置为“允许共享”的开放状态。随后导入 PyTorch 时，它会检测到该状态并选择和平接入，实现两者完美共存。
