---
tags:
  - Python
  - 编程
date: 2026-08-17
---

# np.array_equal

比较numpy数组尽量不用，要用np.array_equal。

两个 numpy 数组用时，不会像普通变量 / 列表那样返回「单个 True/False」，而是对两个数组中对应位置的元素逐一比较 ，返回一个和原数组形状完全相同的布尔数组。

```python
import numpy as np
# 任务1的位置：[5,6]，任务2的位置：[5,7]，任务3的位置：[5,6]
loc1 = np.array([5,6])
loc2 = np.array([5,7])
loc3 = np.array([5,6])

# 数组用==，逐元素比较，返回布尔数组
print(loc1 == loc2)  # 输出 [ True False] → x相等，y不相等
print(loc1 == loc3)  # 输出 [ True  True] → x、y都相等
```

np.array_equal(a, b)是专门判断两个数组是否完全相等的函数，核心特性：

1. 先判断两个数组的形状是否一致（比如都是 2 维、都是 N×2），形状不同直接返回 False；
2. 形状一致则逐元素比较，所有元素都相等才返回单个 True，否则返回单个 False；
3. 返回值是单个布尔值，可以直接用在if判断里，完美解决数组比较的问题。

```python
np.array_equal(loc1, loc2)  # False（元素不全等）
np.array_equal(loc1, loc3)  # True（所有元素相等，形状也一致）
```

# @dataclass与@abstractmethod

## @dataclass 装饰器

**来源**：`from dataclasses import dataclass` (Python 3.7+)

**定义**：一个用于简化类定义的装饰器，专门用于创建主要存储数据的类（Data Class）。

**核心功能**：自动生成 `__init__`、`__repr__`、`__eq__` 等样板代码，让代码极度简洁。

==写法对比

- **传统写法（手动挡）**：

    ```python
    class Point:
        def __init__(self, x, y):
            self.x = x
            self.y = y
        def __repr__(self):
            return f"Point(x={self.x}, y={self.y})"
    ```

- **Dataclass 写法（自动挡）**：

    ```python
    @dataclass
    class Point:
        x: int
        y: int
    ```

## 两个高频装饰器：`@abstractmethod` vs `@dataclass`

它们虽然都带 `@` 符号，但分工完全相反，一个是配置函数，一个配置类：

- **`@abstractmethod`**：
  * **用法**：放在“空壳”类里面的函数头上。
  * **作用**：只提要求，不干活。它强制规定：“谁继承我所在的类，谁就必须自己把这个函数的具体代码写出来，否则直接报错不准运行！”

- **`@dataclass`**：
  * **用法**：放在专门用来装载配置参数的类头上（如 `xxxCfg`）。
  * **作用**：帮你干活。只要加了它，你只需要像写变量一样把参数列出来，Python 就会在后台自动帮你写好繁琐的 `__init__` 初始化函数，极大地保持了代码的整洁。

注：我个人的理解@abstractmethod这个装饰器感觉完全没必要啊，加了跟没加一样。

# field(default_factory=...)

在 Python 类中，**绝对不能**直接使用可变对象（如 `list`, `dict`, `set`）作为类变量的默认值。

错误写法：`items: list = []`

后果：所有该类的实例会**共享同一个列表内存地址**。修改 A 对象的列表，B 对象的列表也会随之改变。

为了解决共享问题，我们需要告诉 Python：不要使用现成的对象，而是每次实例化时，调用一个“工厂函数”现场创建一个新对象。

示例：

```python
taskTypes: List[str] = field(default_factory=lambda: ["search", "fire", "facility"])
```

# 变量占位符与拆包赋值

## 语法现象：`_, var = function()`

在 Python 中，当一个函数返回多个值（以元组形式）时，如果我们只需要其中的某一个或某几个，可以使用 `_` 作为占位符。

## 示例代码

```python
def get_user_info():
    # 返回 (姓名, 年龄, 职业)
    return "wkh", 23, "算法工程师"

# 只需要姓名，忽略其他信息
name, _, _ = get_user_info()

# 或者使用 * 忽略剩余所有
name, *_ = get_user_info()
```

# “空壳”类与 `pass`：架构的“接口契约”

在底层源码（如 `vec_env.py`）中，经常看到一个继承ABC的空壳类，没法被实例化，类里面全是没有具体代码、只有 `pass` 的函数。空壳类（抽象基类）本身不能被“实例化”。也就是说，你不能直接用它在内存里造出一个对象。但是，会有其他的类去继承这个空壳类，并且把空壳里那些只有名字没有内容的函数（也就是那些写着 pass 的地方）真正填满具体的代码。这些继承后的子类，就可以被“实例化”了，造出来的真实对象就能被其他的代码去“调用”里面的函数。

- **核心本质**：这叫“接口契约 (Interface Contract)”，用来强制规范所有接入系统的外部环境。
- **存在的意义**：
  1. **定规矩**：它宣告了主程序（如 `rsl_rl` 大脑）只会通过固定的几个按钮（如 `step()`, `reset()`）来控制环境。
  2. **防崩溃**：强制要求任何继承它的子类（比如具体的机器狗环境）必须实现这些函数。如果不写或者名字拼错，Python 在代码启动的瞬间就会报错拦截，而不是等跑了几天才崩溃。

# `->` 与 `typing`

在强化学习中追踪几十维的巨型 Tensor 极易出错，类型提示（Type Hint）是保命的关键。

- **`->` (返回值注解)**：
  * **作用**：直接标明函数运行结束后，会吐出一个什么类型的数据（如 `def get_name() -> str:`）。

- **为什么原生自带了 `tuple/list`，还要导入 `typing.Tuple/List`？**
  * **原生类型的局限（只能看外表）**：写 `-> tuple` 只能告诉编辑器“我返回了一个元组”，但编辑器不知道里面装的是数字还是张量。
  * **`typing` 的核心价值（透视内部结构）**：在 Python 3.8 及以前，必须使用 `typing.Tuple` 才能精确定义内部结构，如 `Tuple[torch.Tensor, int]`。它能激活编辑器的“上帝视角”，一旦传入错误类型的参数，代码未运行就会亮起红线警告，极大降低 Debug 成本。
  * *(注：从 Python 3.9 开始，原生 `tuple/list` 已支持直接透视内部，如 `tuple[torch.Tensor, int]`，大型框架中为了兼容老版本往往仍保留 `typing` 的写法，但是不需要再`from typing import Tuple`了。)*

# “声明”、“传递类”和“实例化”的区别

必须严格区分“声明”、“传递类”和“实例化”这三个处于不同生命周期的操作。

### 1. 声明/类型提示 (Type Hint) —— 【贴招聘启事】

- **代码形态**：`task_class: VecEnv` （注意是冒号 `:`）
- **本质**：没有消耗内存造出任何东西。它仅仅是在“定规矩”，告诉程序员和编辑器：“未来要塞进这个变量的东西，必须是 `VecEnv` 的子类”。
- **比喻**：在房间门上贴个纸条：“本房间只准放哺乳动物”。至于这个哺乳动物是猫还是狗，不知道。

### 2. 赋值类本体 (Assigning Class) —— 【传递图纸】

- **代码形态**：`task_class = LeggedRobot` （注意后面**没有括号**），（LeggedRobot 继承了 VecEnv ）。
- **本质**：把“造狗的蓝图”存进了变量里，供后续随时使用。此时内存里依然没有活生生的狗，只有一张图纸。
- **比喻**：把“狗的基因图谱”放进了房间。

### 3. 实例化对象 (Instantiation) —— 【真正造物】

- **代码形态**：`env = task_class()` 或 `env = LeggedRobot()` （注意**有括号 `()`**）
- **本质**：真正消耗内存，根据图纸（类）在系统中生成了一个活生生、可交互的对象（Object）。
- **比喻**：真正得到了一只狗。

# eval() 函数

eval() 的全称是 evaluate（求值/评估）。它的作用是：把一段“普通文本字符串”当成“真正的 Python 代码”来执行。例如如果你写 eval("10 * 10")，Python 不会把它当成纯文字，而是会直接算出结果并返回 100。

在大型框架的配置文件里，通常只能写文本（例如 "ActorCritic"）。通过调用 eval("ActorCritic")，Python 会在代码库里找出那个真正叫 ActorCritic 的类本体。这使得我们可以只通过修改配置文件里的单词，就能动态切换不同的神经网络或算法。

`那么问题来了`：我为啥不直接输入参数，而是先字符串然后转化为参数？

我为啥不：

```python
#方案A
from rsl_rl.modules import ActorCritic  # 先导入真实的类
class A1RoughCfgPPO:
    policy_class = ActorCritic  # 直接把类对象赋给变量
```

而是：

```python
#方案B
class A1RoughCfgPPO:
    policy_class_name = "ActorCritic"  # 只写一个纯文本字符串
```

因为：

1. 在一个大型工程中，配置文件（Config）应该是最底层的“纯净水”，任何人都能随时喝一口（读取参数）。

如果你用了方案 A，配置文件就必须 import 神经网络模块。但是，神经网络在构建的时候，往往又需要反过来读取配置文件里的参数（比如要知道输入层的维度）。这样就会造成互相导入（A 导 B，B 导 A），Python 会直接报错崩溃。

用了方案 B，配置文件就彻底变成了一张“纯文本清单”。它不需要引入任何外部的复杂代码，谁想看这张清单都不会引发代码打架。

2. 在实际的科研和落地中，我们经常需要把配置文件保存成纯文本文件，比如 JSON 或 YAML 格式，甚至是作为终端命令行里的一个指令传递（比如 python train.py --policy="ActorCritic"）。

类对象是存在于内存里的活物，你绝对不可能把 ActorCritic 这个实体类保存进一个文本格式的 .json 文件里。

字符串是可以自由流通的。用字符串 "ActorCritic"，这套配置就能在文本文件、命令行、网络传输之间随意穿梭。

# Namespace 对象

在 Python 的 argparse 模块中，Namespace 对象是一个非常简单的容器。它的唯一使命就是存储从命令行解析出来的参数，并让你能以最自然的方式调用它们。

当你使用 argparse 解析命令行输入时，它不会返回一个复杂的列表或繁琐的字典，而是返回一个 Namespace 对象。这个对象本质上是一个“只包含属性的简单对象”。

如果你在终端输入 --task a1 --num_envs 4096，解析后得到的 args 对象在内存里看起来就像这样：

- args.task 的值为 "a1"

- args.num_envs 的值为 4096

# 鸭子类型 (Duck Typing)

“鸭子类型”是 Python 等动态语言的核心设计哲学。它的理念是：如果一只鸟走起来像鸭子，游泳起来像鸭子，叫起来也像鸭子，那么它就是鸭子。

对比理解：

- 传统强类型语言 (如 C++/Java) = 查户口本。你要想当一个 VecEnv，你必须在代码里显式声明继承它（写上 class BaseTask(VecEnv)）。系统只认血缘关系。

- Python (鸭子类型) = 查工作能力。系统不在乎你继承了谁。只要你的类里面包含了 step()、reset() 函数和 num_envs 变量（具备了鸭子的能力），系统就会完美地把你当成 VecEnv 来用。

# import 的执行机制与副作用

## 为啥只import但不调用

第一类：常规库（如 numpy, os）， 如果代码中没有显式调用，那么确实是没用的，删掉不影响程序运行。

第二类：底层框架（如 isaacgym）， 则绝不能删。在 Python 中，import 语句不仅仅是声明，它会立刻触发底层的初始化逻辑。import isaacgym 会在后台瞬间唤醒 C++ 编写的 PhysX 物理引擎，并向系统申请最底层的显卡权限。

## import 的执行机制

`import` 是一次“强行执行”， `import xxx`时 ，Python 会在后台做一件事：

- 找到目标文件夹的 `__init__.py`（或对应脚本），**从头到尾作为真实代码执行一遍**。
- 遇到函数/类：在内存中画好图纸（定义）。
- 遇到直接写在外面的代码（如 `registry.register(...)`）：**当场直接运行**。

所以其实import本身也可以当函数用，毕竟只要import了就自动触发init里的函数。有点类似于游戏里入场就触发一次的特性。

# `if __name__ == "__main__"` 

常见的训练脚本会在最后写：

```python
if __name__ == "__main__":
    args = get_args()
    train(args)
```

## 为什么找不到 `__name__` 的定义

`__name__` 不是作者自己定义的变量，而是 **Python 解释器自动放入每个模块的内置特殊变量**。因此，在代码中通常找不到 `__name__ = ...` 这样的赋值语句。

`__name__` 的值取决于这个 Python 文件是如何启动的：

### 情况一：直接运行该文件

```bash
python train.py
```

此时 Python 会自动设置：

```python
__name__ == "__main__"
```

所以 `if` 条件成立，缩进内的 `get_args()` 和 `train(args)` 会被执行。

### 情况二：该文件被其他文件导入

```python
import train
```

此时 `train.py` 内的 `__name__` 是模块名：

```python
__name__ == "train"
```

它不等于 `"__main__"`，因此不会自动开始训练。但 `train.py` 中定义的函数和类仍然可以被使用，例如 `train.train(args)`。

## 为什么要加这个判断

它相当于一个“程序入口保护器”：

- **直接运行文件**：开始执行主流程。
- **将文件当作模块导入**：只加载函数和类，不自动执行训练。

如果不加这个判断，而是直接在文件最外层写 `train(get_args())`，那么其他文件只要 `import train`，就可能立即触发训练。

# pip install -e . (可编辑安装)

一句话总结：在当前目录下，以“开发者可编辑模式”将项目安装到当前的 Conda 环境中。它的本质是建立一个指向源代码的快捷方式，而不是复制文件。

`-e` (全称 --editable)：代表“可编辑模式” (Editable mode)。

`.` ：代表“当前终端所在的目录”。（前提：该目录下必须存在 setup.py 或 pyproject.toml 等项目构建文件）。

- 普通安装（没有 `-e`）： 比如你 `pip install numpy`，系统会把 `numpy` 的代码复制一份，扔进你的 `Conda` 环境的一个深层文件夹（site-packages）里。

- 可编辑安装（加了 `-e`）： 系统不会复制代码，它只是在你的 `Conda` 环境里建立了一个快捷方式，指向你当前下载的这个 `rsl_rl` 文件夹。

为什么要用 `-e`：因为如果原作者代码有 Bug，或者你想修改底层算法，由于系统是指向这个文件夹的，你只要在这个文件夹里修改了代码保存，你的环境里会立刻生效，不需要重新 `pip install`。

以 `rsl_rl` 为例：

1. 你使用 `git clone` 下载了 `rsl_rl` 的源代码文件夹到你的电脑上。

2. 你在终端 `cd rsl_rl` 进入该文件夹。

3. 执行 `pip install -e .`。

结果： 此时在当前环境内， Python 已经认识了 `rsl_rl` 这个包。无论你的终端以后切换到电脑的哪个目录下运行代码，只要遇到 `import rsl_rl`，系统都会通过快捷方式，飞回你最初 `clone` 下来的那个源码文件夹去读取逻辑。

注意：绝对不能移动源代码文件夹！ 如果你把下载的 `rsl_rl` 文件夹剪切到了另一个硬盘或目录，快捷方式就会断裂。当你再次运行代码时，会直接报错 `ModuleNotFoundError: No module named 'rsl_rl'`。

# 工程路径管理与模块导入

在构建机器人算法工程时，最好别使用写死的“硬编码”路径（如 `C:/xxx`），而是用动态路径解析，以确保代码的跨平台（Windows/Ubuntu）和可移植性。

## 一、 动态路径解析 (os.path )

在项目的根目录初始化文件（通常是 `__init__.py`）中，通常使用以下固定范式来获取项目的绝对根目录：

```python
import os

# 1. 锁定根目录
ROOT_DIR = os.path.dirname(os.path.dirname(os.path.realpath(__file__)))

# 2. 安全拼接子目录
ENVS_DIR = os.path.join(ROOT_DIR, 'legged_gym', 'envs')
```

## 二、 模块导入方式

掌握了根目录坐标后，项目内的模块调用分为两种流派：

1. 绝对导入
- 语法：from legged_gym.utils import task_registry

- 逻辑：以整个项目的根目录为起点，像使用 GPS 一样层层往下寻找目标文件。

- 优点：路径极其清晰，不易产生歧义

- 缺点：如果最外层的包名（legged_gym）发生更改，所有文件内的绝对导入语句都需要跟着修改。

2. 相对导入 (Relative Import)
- 语法：from .helpers import get_args

- 逻辑：以“当前文件”所在的位置为起点，去寻找隔壁邻居。

  * . 代表当前目录。

  * .. 代表上一级目录。

- 优点：利于模块内部的代码重构。只要子文件之间的相对位置不发生改变，外层文件夹随便改名也不会影响内部的相对导入。

- 限制：只能在被当作“包 (Package)”的环境中使用，不能在直接运行的主程序入口文件（如执行 python main.py 的那个文件）中使用相对导入。

# 单例模式与全局状态共享

经常会在文件末尾看到直接将类实例化的代码，例如：

`task_registry = TaskRegistry()`

随后在其他所有文件中，导入的都是这个小写的 `task_registry` 对象，而不是大写的 `TaskRegistry` 类。这是 Python 中实现 **单例模式（Singleton Pattern）** 的经典操作。

## 一、 核心区别：导入“类” vs 导入“对象”

### 1. 导入“类” (Class)

- **写法**：`from legged_gym.utils import TaskRegistry`
- **底层逻辑**：每次想用它时，都需要手动加括号实例化：`my_registry = TaskRegistry()`。
- **致命缺点**：这相当于每次都用了一个**全新的对象**。每一个新管家的内部数据（如字典、列表）都是**彻底清空**的。无法实现跨文件的数据传递。

### 2. 导入“对象” (Instance)

- **写法**：`from legged_gym.utils import task_registry`
- **底层逻辑**：因为原文件末尾已经执行了实例化，Python 在每一次导入该模块时，使用的都是这个唯一的“对象”。
- **优势**：后续无论有多少个不同的脚本去 `import task_registry`，拿到的都是内存中**同一个对象**。

## 二、 应用场景：跨文件状态共享

以 `legged_gym` 框架的运作流程为例，这种模式是维持系统运转的生命线：

1. **写入数据阶段（入职登记）**
   当系统启动加载 `envs/__init__.py` 时，各种机器狗（A1, Cassie 等）会调用 `task_registry.register(...)`，把自己的图纸数据**写入**到管家的内部字典中。

2. **读取数据阶段（下发任务）**
   当运行 `train.py` 开始训练时，主程序会调用 `task_registry.make_env(...)`，要求管家根据名字去字典里找对应的图纸。

# 命令行参数

终端中的 `--task=go2` 是传给 `play.py` 的参数：

```bash
python play.py --task=go2
```

Python 一般使用 `argparse` 定义并读取这类参数：

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--task", type=str, default="go2")
args = parser.parse_args()

print(args.task)
```

- `add_argument()` 定义允许传入的参数。
- `type=str` 表示参数值是字符串。
- `default="go2"` 表示没有传入时使用默认值。
- `parse_args()` 解析终端输入。
- `args.task` 取出结果 `"go2"`。

`--task=go2` 和 `--task go2` 的效果相同。

## 与任务注册表的关系

命令行参数只负责传入名称，任务注册表负责找到名称对应的环境：

```python
task_registry.register("go2", Go2Env)

env_class = task_registry.get_task_class(args.task)
env = env_class()
```

完整过程是：

```text
--task=go2 → args.task → task_registry → Go2Env
```
