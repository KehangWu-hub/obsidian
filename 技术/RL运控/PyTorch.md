---
tags:
  - Python
  - PyTorch
  - 深度学习
date: 2026-09-16
---

# PyTorch 概览

PyTorch 主要包含以下几部分：

- `torch`：张量、数学运算、设备管理和自动微分。
- `torch.nn`：神经网络层、激活函数和损失函数。
- `torch.optim`：SGD、Adam 等优化器。
- `torch.distributions`：概率分布，用于随机采样、计算概率密度等。
- `torch.utils.data`：数据集和批量加载工具。

```python
import torch
from torch import nn
from torch.distributions import Normal
```

`torch` 是顶层包，`nn` 只是 `torch.nn` 的常用别名。因此，`nn.Linear` 的完整名称是 `torch.nn.Linear`。

# Tensor 与张量操作

Tensor 是 PyTorch 中的基本数据结构，可以理解为支持 GPU 计算和自动求导的多维数组。

```python
x = torch.tensor(
    [[1.0, 2.0, 3.0],
     [4.0, 5.0, 6.0]],
    dtype=torch.float32,
)

print(x.shape)   # torch.Size([2, 3])
print(x.dtype)   # torch.float32
print(x.device)  # cpu
```

## 形状

神经网络代码中最常见的错误是张量形状不匹配。常见约定如下：

| 数据   | 常见形状                               |
| ---- | ---------------------------------- |
| 批量向量 | `[batch, features]`                |
| 序列   | `[batch, time, features]`          |
| 图像   | `[batch, channels, height, width]` |

```python
x = torch.randn(32, 10)  # 32 个样本，每个样本 10 个特征

x = x.unsqueeze(1)       # [32, 10] -> [32, 1, 10]
x = x.squeeze(1)         # [32, 1, 10] -> [32, 10]
x = x.reshape(32, 2, 5)  # [32, 10] -> [32, 2, 5]
```

`torch.cat` 在现有维度上拼接，`torch.stack` 则会新建一个维度：

```python
a = torch.randn(4, 3)
b = torch.randn(4, 3)

torch.cat((a, b), dim=1).shape    # [4, 6]
torch.stack((a, b), dim=1).shape  # [4, 2, 3]
```

## 设备

模型和输入必须位于同一设备。

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

model = model.to(device)
x = x.to(device)
```

`.to(device)` 会返回转换后的 Tensor，因此移动输入时需要接收返回值。`model.to(device)` 会将已注册的参数和缓冲区一起移动。

# torch.nn 神经网络模块

`torch.nn` 提供构建神经网络所需的基类、网络层、激活函数和损失函数。

## nn.Module

`nn.Module` 是所有 PyTorch 网络层和模型的基类。自定义模型通常需要：

1. 继承 `nn.Module`。
2. 在 `__init__()` 中定义网络层。
3. 在 `forward()` 中定义前向计算。

```python
class MLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.activation = nn.ReLU()
        self.fc2 = nn.Linear(hidden_dim, output_dim)

    def forward(self, x):
        x = self.fc1(x)
        x = self.activation(x)
        return self.fc2(x)


model = MLP(input_dim=10, hidden_dim=64, output_dim=4)
output = model(torch.randn(32, 10))
```

应使用 `model(x)` 调用模型，不要直接调用 `model.forward(x)`。`nn.Module.__call__()` 会在执行 `forward()` 前后处理钩子等内部逻辑。

将网络层保存为 `self.fc1` 等子模块后，PyTorch 会自动注册它们的参数。可以通过以下方法查看：

```python
for name, parameter in model.named_parameters():
    print(name, parameter.shape)
```

## nn.Parameter

`nn.Parameter` 用于定义需要随模型一起训练的 Tensor。将它赋值给 `nn.Module` 的属性后，PyTorch 会自动将其注册为模型参数：

```python
class Policy(nn.Module):
    def __init__(self, num_actions, init_noise_std=1.0):
        super().__init__()
        self.std = nn.Parameter(
            init_noise_std * torch.ones(num_actions)
        )
        self.distribution = None
```

`self.std` 会出现在 `model.parameters()` 和 `model.state_dict()` 中。执行 `loss.backward()` 后，它的梯度保存在 `self.std.grad`，优化器会像更新网络权重一样更新它。

普通 Tensor 不会因为被保存为模型属性就自动成为参数：

```python
self.std = torch.ones(num_actions)                 # 普通 Tensor
self.std = nn.Parameter(torch.ones(num_actions))   # 可训练参数
```

在连续动作策略中，`std` 可以表示高斯动作分布的标准差，每个动作维度对应一个可学习的噪声强度。`self.distribution = None` 只是初始化占位，不是可训练参数。

## nn.Linear

`nn.Linear(in_features, out_features)` 实现全连接线性变换：

$$
y=xW^\mathsf{T}+b
$$

```python
layer = nn.Linear(in_features=10, out_features=64)

x = torch.randn(32, 10)
y = layer(x)

print(y.shape)  # [32, 64]
```

`in_features` 必须等于输入最后一维的大小。`nn.Linear` 会保留前面的所有维度，只变换最后一维：

```python
layer = nn.Linear(10, 64)
x = torch.randn(8, 20, 10)
y = layer(x)

print(y.shape)  # [8, 20, 64]
```

它内部包含可训练的 `weight` 和 `bias`：

```python
print(layer.weight.shape)  # [64, 10]
print(layer.bias.shape)    # [64]
```

## nn.ReLU

`nn.ReLU()` 创建一个 ReLU 激活层：

$$
\operatorname{ReLU}(x)=\max(0,x)
$$

```python
activation = nn.ReLU()

x = torch.tensor([-2.0, 0.0, 3.0])
y = activation(x)

print(y)  # tensor([0., 0., 3.])
```

`nn.ReLU` 是类，`nn.ReLU()` 是由该类创建的模块对象。在模型中需要保存或传递激活层时，通常使用 `nn.ReLU()`。

PyTorch 也提供了函数形式：

```python
y1 = nn.ReLU()(x)
y2 = torch.relu(x)
```

两者在这个例子中结果相同。模块形式更容易放入 `nn.Sequential`，函数形式则适合直接写在 `forward()` 的计算逻辑中。

其他常用激活层见 [[神经网络笔记#激活函数]]。

## nn.Sequential

`nn.Sequential` 按声明顺序保存多个模块，并将前一个模块的输出传给后一个模块：

```python
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.ReLU(),
    nn.Linear(64, 4),
)

x = torch.randn(32, 10)
y = model(x)

print(y.shape)  # [32, 4]
```

数据依次经过：

```text
[32, 10] -> Linear -> [32, 64] -> ReLU -> [32, 64]
         -> Linear -> [32, 4]
```

ReLU 不改变张量形状，只将负数置为 $0$。第二个 `Linear` 的 `in_features=64` 必须与前一层的输出维度一致。

也可以为各层命名：

```python
from collections import OrderedDict

model = nn.Sequential(OrderedDict([
    ("hidden", nn.Linear(10, 64)),
    ("activation", nn.ReLU()),
    ("output", nn.Linear(64, 4)),
]))
```

`nn.Sequential` 适合单输入、单输出的线性数据流。当模型包含多个输入、条件分支、跳连接或需要同时返回多个值时，自定义 `nn.Module` 通常更清楚。

## 损失函数

损失函数衡量模型预测值和目标值之间的差距。训练时先得到 `loss`，再调用 `loss.backward()` 计算梯度，由优化器更新参数。

### MSELoss

`nn.MSELoss` 计算均方误差，常用于回归任务：

$$
L=\frac{1}{N}\sum_{i=1}^{N}(\hat y_i-y_i)^2
$$

```python
prediction = torch.tensor([[1.2], [2.7], [4.1]])
target = torch.tensor([[1.0], [3.0], [4.0]])

criterion = nn.MSELoss()
loss = criterion(prediction, target)
```

误差经过平方后，较大的误差会受到更强的惩罚。`prediction` 和 `target` 应具有相同形状，避免出现非预期的广播。

### CrossEntropyLoss

`nn.CrossEntropyLoss` 常用于单标签多分类。模型为每个类别输出一个 logit，目标保存正确类别的索引。

```python
# 4 个样本，每个样本有 3 个类别分数
logits = torch.randn(4, 3)
target = torch.tensor([0, 1, 2, 1], dtype=torch.long)

criterion = nn.CrossEntropyLoss()
loss = criterion(logits, target)
```

对应的形状为：

```text
logits: [batch, classes]
target: [batch]
```

logit 是未经归一化的类别分数。`CrossEntropyLoss` 内部已经包含 Softmax 相关计算，因此训练时应直接传入 logits，不要提前手动执行 Softmax。

# torch.distributions 概率分布

## Normal：正态分布（高斯分布）

```python
from torch.distributions import Normal
```

这句代码从 `torch.distributions` 模块中导入 **`Normal` 类**。给它均值和标准差，就能创建一个正态分布对象，再从中随机抽取数值。

```python
mean = torch.tensor([0.2])
std = torch.tensor([0.1])

distribution = Normal(mean, std)  # 创建分布对象
action = distribution.sample()  # 从分布中抽取一个动作，返回 Tensor
```

均值 `0.2` 决定分布的中心，标准差 `0.1` 决定采样的分散程度。抽出的数值可能是 `0.15`，也可能是 `0.27`；标准差越大，探索范围通常越宽。传入的标准差需要大于零。

这里可以分清三个东西：**`Normal` 是类，`distribution` 是分布对象，`action` 是采样得到的张量。**

### 在机器人策略中怎样使用？

在 [[技术/RL运控/rsl_rl库|rsl_rl]] 的 Actor 中，网络先给出动作均值，再用 `Normal` 建立分布并采样动作：

```python
mean = self.actor(observations)              # [环境数, 动作维度]
distribution = Normal(mean, self.std)        # std 为 [动作维度]，自动广播
actions = distribution.sample()             # [环境数, 动作维度]
```

例如 4096 个机器人、每个机器人 12 维动作，`actions` 的形状就是 `[4096, 12]`。每只机器人围绕自己的动作均值进行随机探索。

### log_prob() 和 entropy()

```python
log_prob = distribution.log_prob(actions)   # 各动作维度的对数概率密度
log_prob = log_prob.sum(dim=-1)              # 每个机器人的整组动作得到一个值

entropy = distribution.entropy().sum(dim=-1) # 每个机器人动作分布的熵
```

- **`log_prob()`**：衡量这些动作在当前分布下的对数概率密度。PPO 用新旧策略的 `log_prob` 计算概率比，判断策略改变了多少。
- **`entropy()`**：衡量分布的随机程度，用于鼓励探索。

各动作维度按独立分布处理，整组动作的概率密度是各维的乘积，取对数后就变成相加，所以代码使用 `.sum(dim=-1)`。

# 模型训练与使用

模型训练会重复执行四个步骤：前向计算、计算损失、反向传播、更新参数。推理时只需要前向计算，不再更新参数。

## 自动微分与优化器

### 自动微分

PyTorch 会在前向计算时记录 Tensor 之间的运算关系。调用 `loss.backward()` 后，PyTorch 从损失开始反向计算梯度，并将每个参数的结果保存在 `.grad` 中。

梯度表示参数发生小变化时，损失会如何变化。反向传播只负责计算梯度，不会直接修改参数。

### 优化器

优化器读取参数的 `.grad`，并按照指定的更新规则修改参数，使损失逐渐减小。

```python
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
```

- `model.parameters()`：将需要更新的模型参数交给优化器。
- `lr`：学习率，决定每次参数更新的幅度。过大可能使训练不稳定，过小则会使收敛过慢。

| 优化器 | 特点 |
| --- | --- |
| `SGD` | 更新规则简单，通常需要更谨慎地调整学习率 |
| `Adam` | 会根据梯度统计调整各参数的更新幅度，常用作初始选择 |

### 一次参数更新

```python
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.ReLU(),
    nn.Linear(64, 1),
)

optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.MSELoss()

x = torch.randn(32, 10)
target = torch.randn(32, 1)

optimizer.zero_grad()
prediction = model(x)
loss = criterion(prediction, target)

loss.backward()
optimizer.step()
```

一次标准更新包含：

1. `optimizer.zero_grad()`：清除上一次更新留下的梯度。
2. `prediction = model(x)`：执行前向计算。
3. `loss = criterion(...)`：计算预测误差。
4. `loss.backward()`：反向传播并计算梯度。
5. `optimizer.step()`：使用梯度更新参数。

> PyTorch 默认会累积梯度。如果忘记清零，新梯度会与上一次的梯度相加。

## 训练与推理模式

```python
# 训练
model.train()
prediction = model(x)

# 推理
model.eval()
with torch.inference_mode():
    prediction = model(x)
```

`model.train()` 和 `model.eval()` 会切换 Dropout、BatchNorm 等模块的行为，但不会自动启用或禁用梯度。`torch.inference_mode()` 才是在推理时关闭自动求导的方式之一。

- `model.train()`：Dropout 保持随机丢弃，BatchNorm 更新运行统计量。
- `model.eval()`：Dropout 停止随机丢弃，BatchNorm 使用已保存的统计量。

## 模型保存与加载

`state_dict` 是一个保存参数名称和 Tensor 的字典。通常只保存它，而不直接序列化整个模型：

```python
torch.save(model.state_dict(), "model.pt")

model = MLP(input_dim=10, hidden_dim=64, output_dim=4)
state_dict = torch.load("model.pt", map_location="cpu")
model.load_state_dict(state_dict)
model.eval()
```

加载 `state_dict` 之前需要先创建与保存时结构一致的模型。如果还需要恢复训练，应另外保存优化器状态、训练轮数和其他必要配置。
