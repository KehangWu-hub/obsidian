---
tags:
  - 强化学习
  - PPO
  - rsl_rl
date: 2026-08-17
---

本文建议对照 [rsl_rl库](https://github.com/leggedrobotics/rsl_rl)源码食用。

`rsl_rl` 是一个用 PyTorch 实现的强化学习算法库，主要提供 PPO 等 On-Policy 强化学习算法及其训练框架。

# 1. 先想清楚：我们要让机器人学会什么？

假设要训练一只四足机器人按指定速度行走。我们已经有仿真环境，里面有机器人、地面、关节和传感器。

每走一步，都要做这件事：

**读取机器人状态 → 网络给出动作 → 仿真执行动作 → 得到新状态和奖励。**

例如，机器人按要求向前走，奖励就高一些；机身晃动过大、关节用力过猛，奖励就低一些。训练的目标是调整网络参数，让它逐渐学会取得更高的累计奖励。

`rsl_rl` 负责其中的**网络和训练过程**。机器人模型、物理仿真、观测内容、奖励规则由接入它的环境提供。

## 五个文件分别管什么？

- **`env/vec_env.py` → `VecEnv`**：规定训练程序怎样向环境读取状态、发送动作。
- **`modules/actor_critic.py` → `ActorCritic`**：放两个网络。Actor 决定动作，Critic 估计当前局面的未来收益。
- **`storage/rollout_storage.py` → `RolloutStorage`**：存下机器人刚才的经历，整理成训练数据。
- **`algorithms/ppo.py` → `PPO`**：根据这些经历计算损失，更新两个网络。
- **`runners/on_policy_runner.py` → `OnPolicyRunner`**：把上述模块接起来，安排采样、训练、记录和保存。

总结就是**环境负责让机器人动起来，ActorCritic 负责给出预测，Storage 负责存数据，PPO 负责学习，Runner 负责组织整个过程。**

# 2. 环境接口 VecEnv：一次让很多机器人同时走一步

## 为什么需要很多环境？

只用一个机器人采集数据，训练会比较慢。因此训练时通常同时运行很多份仿真，每份里面各有一只机器人。

例如同时运行 **4096 个环境**：有的机器人正正常行走，有的刚摔倒，有的刚重置。它们共用同一套网络参数，但各自有自己的状态。

`VecEnv` 中的 Vec 来自 vectorized，可以理解为**把很多环境打包起来，一起处理**。

假设每只机器人的观测由 48 个数字组成，动作由 12 个数字组成，那么：

```python
obs.shape      # [4096, 48]
actions.shape  # [4096, 12]
```

`obs` 的每一行是一只机器人的观测，`actions` 的对应行就是给这只机器人的动作。网络一次处理整批观测，得到整批动作。

## “接口”是什么意思？

训练程序需要和环境配合，所以双方要约定几个函数：

```python
obs = env.get_observations()
obs, privileged_obs, rewards, dones, infos = env.step(actions)
```

第一行是“把当前观测给我”。第二行是“让各个机器人执行各自的动作，然后把结果给我”。

**`VecEnv` 就是写下这套约定的地方。** 具体机器人环境负责实现函数里面的工作：推进仿真、计算奖励、判断摔倒等。

可以把 `env.step(actions)` 展开理解为：

**接收动作 → 转成控制指令 → 推进物理仿真 → 读取新状态 → 计算奖励与结束标记 → 返回结果。**

一次 `step()` 对应一次环境动作执行过程；环境内部可以运行若干个更小的物理仿真步。

## `step()` 返回的五个东西

**`obs`：下一步给 Actor 看的观测。**

例如机身角速度、重力方向、关节位置、关节速度、目标行走速度。这些数字的具体排列由机器人环境决定。

**`privileged_obs`：给 Critic 看的观测。**

训练时，仿真还知道一些真机上难以直接获取的信息，例如地形、摩擦系数等。可以让 Critic 利用这些信息，更准确地估计未来收益。环境会把 Critic 需要的内容组织成完整输入；没有单独提供时，Critic 就和 Actor 使用相同观测。

**`rewards`：这一步每只机器人的奖励。**

形状为 `[4096]`。例如 `rewards[7]` 是第 8 个环境刚刚得到的奖励。

**`dones`：哪些机器人的当前回合结束了。**

形状为 `[4096]`。某个位置为真，表示对应机器人摔倒、完成任务或达到时间上限，需要开始新回合。这套训练流程中，具体环境负责完成相应机器人的重置，并提供下一步可用的观测。

**`infos`：补充信息。**

它是字典，常用于传递超时标记 `time_outs`、回合奖励统计 `episode` 等。

# 3. ActorCritic：一个负责动作，一个负责估值

`ActorCritic` 里面放着两个神经网络：Actor 和 Critic。这里先看普通的全连接网络版本，也就是 MLP。

## Actor：看到当前状态，决定怎么动

沿用 48 维观测、12 维动作的例子：

**48 个观测数字 → 若干隐藏层 → 12 个动作输出。**

可以配置成：

```python
actor_hidden_dims = [256, 256, 256]
```

这表示有三层隐藏层，每层 256 个神经元。Actor 的作用就是学出“什么状态下应该采取什么动作”这套映射。

在关节位置控制任务中，12 维动作可以经过环境的缩放与偏移，变成 12 个关节的目标角度，再交给 PD 控制器执行。

## 训练时为什么还要随机采样动作？

训练初期，网络还不知道怎么走。如果永远只执行同一种输出，就很难尝试到更好的动作。因此 Actor 在训练时会围绕自己的输出做一些随机探索。

这份实现让网络输出动作的**均值** `mean`，再配合可学习的**标准差** `std`，组成高斯分布：

$$
a_t \sim \mathcal{N}\bigl(\mu_\theta(o_t),\ \operatorname{diag}(\sigma^2)\bigr)
$$

- $o_t$：当前观测。
- $\mu_\theta(o_t)$：Actor 根据观测算出的动作均值。
- $\sigma$：各个动作维度的探索幅度。
- $a_t$：这次真正采样出来、准备执行的动作。

例如某个动作维度的均值是 `0.2`，标准差是 `0.1`，那么这次可能采到 `0.15`，下次可能采到 `0.27`。标准差越大，尝试的范围越宽。

核心代码很短：

```python
mean = self.actor(observations)
self.distribution = Normal(mean, mean * 0. + self.std)
actions = self.distribution.sample()
```

这里 `std` 为每个动作维度保存一个可学习参数，训练过程中和网络权重一起更新。

## Critic：估计从现在开始还能拿多少奖励

Critic 输入当前观测，输出一个数字 $V(s_t)$，表示它预测的**未来折扣累计奖励**。

“折扣”就是让更远的奖励权重稍小一点：

$$
V(s_t) \approx \mathbb{E}\left[r_t + \gamma r_{t+1} + \gamma^2 r_{t+2} + \cdots\right]
$$

例如 $\gamma=0.99$，下一步奖励乘 `0.99`，再下一步乘 `0.99²`。

可以把 Critic 理解为：**看一眼当前局面，预测接下来大概能拿多少分。** 后面 PPO 会拿实际采集的数据修正这个预测，并判断刚才的动作比预期好还是差。

## 看源码时先认这几个函数

```python
actor_critic.act(obs)                     # 采样动作，给训练中的机器人执行
actor_critic.evaluate(critic_obs)         # 调用 Critic，预测价值
actor_critic.get_actions_log_prob(actions) # 计算动作在当前分布下的 log_prob
actor_critic.act_inference(obs)           # 直接输出动作均值，供推理使用
```

`log_prob` 是动作概率密度的对数。先把它理解成**当前策略对这个动作的倾向程度**，后面用它比较新旧策略。

12 个动作维度各自有一个对数概率密度，代码把它们相加，得到整组动作的 `log_prob`。

# 4. PPO 比 ActorCritic 多了什么？

到这里，网络已经能输出动作和价值。接下来的问题是：**机器人走过之后，怎样利用结果让网络变好？**

这就是 `PPO` 的工作：

**调用网络选动作 → 收集执行结果 → 计算训练目标 → 计算损失 → 更新网络。**

`ActorCritic` 定义网络的结构和输出，`PPO` 持有这个网络对象，并组织训练它的方法：

```python
actor_critic = ActorCritic(...)
alg = PPO(actor_critic, ...)
```

PPO 初始化时还会建立优化器：

```python
self.optimizer = optim.Adam(
    self.actor_critic.parameters(),
    lr=learning_rate,
)
```

可以把优化器理解为**根据梯度调整网络参数的工具**。这里 Actor、Critic 和动作标准差的参数都交给它管理。

后面读 `ppo.py`，先抓住四个函数：

- `act()`：让网络选动作，同时暂存训练需要的信息。
- `process_env_step()`：收到环境结果，把一步经历补齐并存入缓存。
- `compute_returns()`：算出回报目标和优势。
- `update()`：用这些数据更新网络。

# 5. 一步经历怎样存下来？

## 执行动作前，先记录“当时怎么想的”

Runner 调用：

```python
actions = alg.act(obs, critic_obs)
```

`PPO.act()` 会调用 Actor 和 Critic，并记录：

- 当时看到的观测。
- 当时采样出的动作。
- Critic 当时预测的价值。
- 这个动作当时的 `log_prob`。
- 当时动作分布的均值和标准差。

这些内容先放在 `Transition` 这个临时对象里。此时机器人还没执行动作，奖励还空着。

## 执行动作后，把结果补上

```python
obs, privileged_obs, rewards, dones, infos = env.step(actions)
alg.process_env_step(rewards, dones, infos)
```

这里有两个时刻的观测：`act()` 接收的是动作执行前的 $o_t$；`env.step(actions)` 执行动作后，返回供下一次决策使用的 $o_{t+1}$，同时返回这次动作产生的奖励 $r_t$ 和结束标记 $done_t$。代码复用了 `obs` 这个变量名，所以时间上的变化不太明显。

返回的数据分成两路：

- **奖励 $r_t$ 和结束标记 $done_t$** → 由 `process_env_step()` 填入刚才暂存的记录，与动作前的 $o_t$、当时执行的 $a_t$ 放在一起，再写入 Storage，供训练使用。
- **新观测 $o_{t+1}$** → 留给下一次 `act()`，用于选择下一个动作 $a_{t+1}$。

所以“补上”指的是补齐**这次动作的经历记录**：在 $o_t$ 下做了 $a_t$，得到了 $r_t$，这一局是否结束。

`Transition` 暂存当前一个并行时间步的数据，`RolloutStorage` 保存连续多个时间步的数据。

## “分配张量”就是提前准备存数据的内存

假设 4096 个环境各采集 24 步，每条观测有 48 个数字，观测缓存可以这样创建：

```python
observations = torch.zeros(24, 4096, 48, device="cuda")
```

形状 `[24, 4096, 48]` 的意思是：

**第几步 → 第几个环境 → 这只机器人的 48 个观测数字。**

例如 `observations[3, 7]` 保存第 4 步、第 8 个环境的观测。

创建这块张量时，会为数据准备显存并填零。如果使用 `float32`，这块观测数据占：

$$
24 \times 4096 \times 48 \times 4\ \text{字节} = 18\ \text{MiB}
$$

动作、奖励、价值等也有各自的缓存。它们提前准备好，采样时直接往对应位置写：

```python
self.observations[self.step].copy_(transition.observations)
self.actions[self.step].copy_(transition.actions)
self.step += 1
```

这样每一步都能复用已有空间。训练完一批后，写入位置回到开头，下一批数据覆盖上一批。

# 6. 一轮到底采多少？episode 和 epoch 在哪里？

下面统一用这一组例子：

```python
num_envs = 4096
num_steps_per_env = 24
num_learning_epochs = 5
num_mini_batches = 4
```

## Rollout：这次收集到的一批经历

Runner 连续调用 24 次 `env.step()`，每次推进 4096 个环境，因此一共得到：

$$
24 \times 4096 = 98\,304\ \text{条单步经历}
$$

这整批数据就是本轮 **rollout**。

## Episode：一只机器人的一局

某只机器人重置后开始行走，直到摔倒、完成任务或达到时间上限，这段过程叫一个 **episode**。

采集数据时，各只机器人各走各的。有的一个 episode 能持续上千步，有的走几步就摔倒并重置。Runner 按配置采满 24 步，就开始训练。

因此，一个很长的 episode 可以经历这样的过程：

**走 24 步 → 暂停交互并训练 → 继续走 24 步 → 再训练 → …… → 最后结束这一局。**

## Mini-batch：一次拿多少数据来算梯度

这批数据有 98,304 条，一次取其中一部分训练，这部分就叫 **mini-batch**。

这里分成 4 份，每份有：

$$
98\,304 \div 4 = 24\,576\ \text{条数据}
$$

每取一份，就计算一次损失、反向传播一次，并执行一次参数更新。

## Epoch：把这批数据完整学一遍

4 个 mini-batch 都用过一遍，就完成 **1 个 epoch**。

设置 `num_learning_epochs = 5`，表示把这批 rollout 数据学 5 遍，所以一共执行：

$$
5 \times 4 = 20\ \text{次参数更新}
$$

epoch 是通用的训练用语，监督学习也常用。1 个 epoch 表示学一遍，多个 epoch 表示重复学习。

## Learning iteration：采一批，再训练这一批

一次 iteration 包含：

**4096 个环境各走 24 步 → 算训练目标 → 分成 4 个 mini-batch → 学 5 个 epoch → 开始下一次采样。**

所以看配置时可以这样读：

- `num_steps_per_env`：每个环境走多少步后开始训练。
- `num_mini_batches`：这批数据分几份训练。
- `num_learning_epochs`：这批数据学几遍。
- `num_learning_iterations`：整个“采集 + 训练”过程做多少轮。

# 7. 有了奖励，为什么还要算 returns 和 advantages？

一个动作的影响可能要过几步才看得出来。比如某一步迈得太大，当下速度很快，但几步后失去平衡。因此学习时要把后续结果也考虑进去。

采完 rollout 后，代码为每一步算出两个量：

- **`returns`：回报目标，给 Critic 学习用。** 告诉它从这个局面出发，未来收益应该估计到什么水平。
- **`advantages`：优势，给 Actor 学习用。** 告诉它当时采取的动作，比原先预期好多少。

## 优势可以怎样理解？

假设 Critic 原先预测未来收益是 `10`，根据采样结果算出的回报目标是 `13`：

$$
\hat A_t = 13 - 10 = 3
$$

优势为正，说明这次结果比预期好，训练会倾向于提高这个动作的概率。若目标只有 `7`，优势就是 `-3`，训练会倾向于降低它的概率。

## GAE：把连续几步的“超出预期”合起来

代码通过 GAE 计算优势。先看一步的预测误差：

$$
\delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)
$$

它的意思是：

**刚拿到的奖励 + 对下一状态未来收益的估计 − 原来对当前状态的估计。**

例如 $r_t=1$、$V(s_t)=10$、$V(s_{t+1})=12$、$\gamma=0.99$：

$$
\delta_t = 1 + 0.99 \times 12 - 10 = 2.88
$$

这一步的结果比原先预计更好。

再把后续几步的误差考虑进来：

$$
\hat A_t = \delta_t + \gamma\lambda\hat A_{t+1}
$$

这就是源码从后往前循环的原因：算当前优势时，需要后一步已经算好的优势。$\lambda$ 控制后续误差传回来多少。

回报目标则是：

$$
\hat R_t = \hat A_t + V(s_t)
$$

代码还会把优势做标准化，使训练时的数值尺度更稳定；Critic 使用的回报目标保留原有尺度。

## 采样结束了，后面的收益怎么办？

采满 24 步时，很多机器人还在正常行走。代码会让 Critic 再看一眼最后的状态，用它预测后续收益，再接上已经拿到的奖励。这个做法叫 **bootstrap**，可以理解为“后面还没采到的部分，先用估计值补上”。

如果机器人已经摔倒，`done` 会截断这条回合的递推，下一局的数据从新的回合开始计算。

达到时间上限的情况通过 `infos['time_outs']` 单独标记。这份实现会先在奖励上补一项价值估计，再按回合边界截断。

# 8. PPO.update()：网络究竟怎样学？

现在缓存里已经有观测、旧动作、旧动作概率、回报目标和优势。`update()` 取出 mini-batch，开始真正训练。

## 第一步：让当前网络重新看当时的观测

采样时的网络记下了：“在这个观测下，我选择这个动作的倾向程度是多少。”

经过几次参数更新后，网络已经变化。现在再次输入同样的观测，计算**当时那个动作**在当前策略下的 `log_prob`，就能比较新旧策略。

```python
self.actor_critic.act(obs_batch)  # 建立当前动作分布
new_log_prob = self.actor_critic.get_actions_log_prob(actions_batch)
new_value = self.actor_critic.evaluate(critic_obs_batch)
```

这里用于比较的是缓存中当时执行过的 `actions_batch`。采样阶段记录的 `old_log_prob` 始终作为固定参照。

## 第二步：用概率比衡量策略改了多少

$$
\rho_t = \frac{\pi_\theta(a_t\mid o_t)}{\pi_{\mathrm{old}}(a_t\mid o_t)}
$$

- $\rho_t=1$：新旧策略对这个动作的倾向相同。
- $\rho_t=1.2$：当前策略对这个动作的概率密度提高了 20%。
- $\rho_t=0.8$：降低了 20%。

源码通过对数概率计算这个比值：

```python
ratio = torch.exp(new_log_prob - old_log_prob)
```

## 第三步：策略损失让好动作更容易出现

优势为正，就希望提高动作概率；优势为负，就希望降低动作概率。但同一批数据要训练好几遍，更新过猛容易让策略失去稳定性。

PPO 用裁剪目标控制这种更新倾向：

$$
L_{\mathrm{actor}} = -\mathbb E_t\left[
\min\left(
\rho_t\hat A_t,
\operatorname{clip}(\rho_t,1-\epsilon,1+\epsilon)\hat A_t
\right)
\right]
$$

假设 `clip_param = 0.2`，裁剪区间就是 `[0.8, 1.2]`。

举例：优势为正时，提高动作概率有利于改善目标；当比值超过 `1.2` 后，裁剪分支让继续增大它不再获得额外收益。这样能缓和同一批数据上的过度更新。

源码中的 `surrogate_loss` 就是这部分策略损失。它在实际优化目标上做限制，因此最终比值仍可能超出裁剪区间。

## 第四步：价值损失让 Critic 估得更准

Critic 预测 `new_value`，采样数据给出目标 `returns`。最直观的价值损失就是平方误差：

$$
L_{\mathrm{value}} = \mathbb E_t\left[(V_\theta(s_t)-\hat R_t)^2\right]
$$

```python
value_loss = (new_value - returns_batch).pow(2).mean()
```

如果启用 `use_clipped_value_loss`，代码还会围绕采样时的旧价值构造裁剪后的预测，并用较保守的损失训练，缓和价值预测的剧烈变化。

读这里的变量名时，`target_values_batch` 存的是采样时的旧价值，`returns_batch` 才是要拟合的回报目标。

## 第五步：熵奖励保留探索

动作分布越分散，通常探索越多。**熵**就是衡量分布随机程度的一个量。

损失里加入熵奖励，可以鼓励策略保留一定探索，继续尝试可能更好的动作。

三个部分合起来：

```python
loss = (
    surrogate_loss
    + value_loss_coef * value_loss
    - entropy_coef * entropy.mean()
)
```

也就是：

**学会选择更好的动作 + 学会更准确地估值 + 保留一定探索。**

## 第六步：反向传播，修改参数

```python
optimizer.zero_grad()     # 清掉上一批梯度
loss.backward()          # 计算各参数应该往哪个方向调整
clip_grad_norm_(...)     # 限制梯度大小，缓和过大的更新
optimizer.step()         # 实际修改参数
```

这个过程对每个 mini-batch 执行一次。在前面的例子中，一轮 rollout 对应 20 次这样的更新。

## 自适应学习率：看策略变化来调步子大小

开启 `schedule="adaptive"` 时，代码还会计算新旧动作分布之间的 **KL 散度**。可以把它理解为“这两个分布差了多少”。

**变化太大 → 降低学习率；变化很小 → 适当提高学习率。**

这部分利用采样时保存的动作均值、标准差与当前分布进行比较，帮助调整更新速度。

# 9. OnPolicyRunner：把整条训练流程连起来

现在再看 Runner，就能把每一行对应到具体工作。

## 初始化：准备好网络、算法和缓存

Runner 接收环境 `env` 和三组配置：

```python
train_cfg["runner"]     # 采多少步、多久保存一次
train_cfg["policy"]     # 网络有几层、每层多宽
train_cfg["algorithm"]  # 学习率、epoch 数、PPO 裁剪等
```

初始化过程可以概括为：

**从环境读取观测和动作维度 → 创建 ActorCritic → 把网络交给 PPO → 给 Storage 分配内存 → 重置环境。**

例如环境声明 `num_obs=48`、`num_actions=12`，Runner 就据此创建输入为 48 维、输出为 12 维的 Actor。Critic 的输入维度由环境提供的 Critic 观测决定。

## `learn()`：先采数据，再更新网络

下面是帮助理解的简化流程：

```python
obs = env.get_observations()
privileged_obs = env.get_privileged_observations()
critic_obs = privileged_obs if privileged_obs is not None else obs

for iteration in range(num_learning_iterations):
    with torch.inference_mode():
        for step in range(num_steps_per_env):
            actions = alg.act(obs, critic_obs)
            obs, privileged_obs, rewards, dones, infos = env.step(actions)
            critic_obs = privileged_obs if privileged_obs is not None else obs
            alg.process_env_step(rewards, dones, infos)

        alg.compute_returns(critic_obs)

    alg.update()
    # 记录日志，按间隔保存模型
```

`torch.inference_mode()` 用在采样阶段：这时只需要网络输出动作和估值，关闭梯度记录可以节省内存与计算。

到了 `alg.update()`，网络重新计算输出，建立梯度关系，再反向传播更新参数。

看完整个库时，可以一直沿着这条调用线往下找：

**`Runner.learn()` → `PPO.act()` → `env.step()` → `PPO.process_env_step()` → `compute_returns()` → `PPO.update()`。**

# 10. 如果网络需要记忆：ActorCriticRecurrent

前面的 MLP 根据当前输入做决策。如果希望网络在内部记住前几步发生的事情，可以使用 `ActorCriticRecurrent`。

它在普通网络前加入 GRU 或 LSTM：

**当前观测 + 上一步的记忆 → RNN → 当前特征与新记忆 → MLP → 动作。**

例如机器人刚受到一次推力，即使当前观测不包含完整历史，RNN 也有机会通过内部记忆利用前几步的变化。

Actor 和 Critic 各有自己的记忆网络。代码中的 `hidden_states` 就是这些记忆。

## 采样时：每走一步更新记忆

每个环境保留自己的 hidden state。处理当前观测后，RNN 产生新状态，下一步继续使用。

某只机器人回合结束后，会清空它对应的记忆，让新回合从干净的状态开始。

## 训练时：把连续片段一起交给网络

RNN 学习需要时间顺序，因此 Storage 会把数据整理成连续轨迹片段，并保存各片段开始时的记忆。

不同片段长度可能不同，代码会用零补齐；`mask` 标记哪些位置是真数据，哪些位置是补齐的。

**按回合切片段 → 补齐长度 → 带着起始记忆顺序处理 → 取出有效位置 → 计算 PPO 损失。**

读懂普通 MLP 的训练流程后，再看这条分支会容易很多。对应文件是 `modules/actor_critic_recurrent.py`，以及 Storage 中的循环策略 mini-batch 生成函数。

# 11. 训练怎么启动，结果怎么看？

## 配置应该去哪里找？

读项目的训练配置时，先找下面这些参数：

```python
# runner：采样与保存
num_steps_per_env = 24
save_interval = 100

# policy：网络结构
actor_hidden_dims = [256, 256, 256]
critic_hidden_dims = [256, 256, 256]

# algorithm：怎么学习
num_learning_epochs = 5
num_mini_batches = 4
learning_rate = 1e-3
clip_param = 0.2
gamma = 0.99
lam = 0.95
```

这些数值用于串起本文的例子。具体任务的配置还会包含策略类名、算法类名、熵系数等。

准备好环境和完整配置后，入口是：

```python
from rsl_rl.runners import OnPolicyRunner

runner = OnPolicyRunner(
    env,
    train_cfg,
    log_dir="logs/my_run",
    device=env.device,
)
runner.learn(num_learning_iterations=1000)
```

环境、网络和缓存通常放在同一设备上，例如同一张 GPU，方便直接传递整批数据。

## 保存与继续训练

训练检查点主要保存网络参数、优化器状态和迭代编号。网络参数决定机器人当前会怎么做，优化器状态用于接着训练。

```python
runner.save("logs/my_run/checkpoint.pt")

# 先按相同网络结构和配置创建 runner，再加载
runner.load("logs/my_run/checkpoint.pt", load_optimizer=True)
runner.learn(num_learning_iterations=500)
```

这里的 `500` 表示继续进行 500 轮“采集 + 训练”。实验时也把配置一起留存，方便恢复网络结构和训练设置。

## 训练完怎样让机器人执行策略？

```python
policy = runner.get_inference_policy(device=env.device)

with torch.inference_mode():
    actions = policy(env.get_observations())
```

普通 Actor 的推理会直接使用动作均值。接下来仍由环境或真机控制程序把这些动作转换成实际控制指令。

## 日志先看什么？

- **`Train/mean_reward`**：最近结束的回合平均拿到多少奖励。结合实际运动表现判断策略是否进步。
- **`Train/mean_episode_length`**：一局平均持续多少步。对容易摔倒的任务，回合变长通常是有用信号。
- **`Loss/value_function`**：Critic 的估值与训练目标相差多少。
- **`Policy/mean_noise_std`**：动作探索幅度有多大。
- **`Perf/total_fps`**：采集与训练的总体处理速度。

策略损失会随着每轮采集的数据变化，因此看训练效果时，以奖励、回合表现和实际运行的机器人为主。

# 12. 回头看源码：按什么顺序读？

**先看 `ActorCritic`**：弄清楚观测怎样变成动作，Critic 怎样输出一个价值。

↓

**再看 `VecEnv` 和具体机器人环境的 `step()`**：弄清楚动作怎样被执行，观测、奖励和 done 怎样产生。

↓

**看 `PPO.act()` 与 `process_env_step()`**：跟着一次交互，看执行前后的信息怎样汇合。

↓

**看 `RolloutStorage`**：看数据存在哪里，24 步怎样累积成一批，怎样分成 mini-batch。

↓

**看 `compute_returns()` 与 `PPO.update()`**：把优势、回报、概率比、损失和参数更新一一对应上。

↓

**最后回看 `OnPolicyRunner.learn()`**：此时训练主循环里的每一次调用，都能对应到具体工作。

需要修改功能时，也按这个分工找：**改奖励和观测找环境，改网络找 ActorCritic，改损失找 PPO，增加数据字段找 Storage，改采样安排和日志保存找 Runner。**

整套库最终做的就是不断重复：

**机器人尝试动作 → 记录结果 → 判断比预期好多少 → 调整网络 → 用新策略继续尝试。**
