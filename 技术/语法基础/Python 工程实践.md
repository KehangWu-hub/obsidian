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

# `@dataclass`、`@abstractmethod` 与 `@staticmethod`

三者都是装饰器，但作用对象和目的不同：`@dataclass` 处理类，`@abstractmethod` 和 `@staticmethod` 处理方法。

## `@dataclass`

`@dataclass` 来自 `dataclasses` 模块，用于简化以保存数据为主的类。它会根据类型注解自动生成 `__init__`、`__repr__` 和 `__eq__` 等方法。

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

point = Point(1, 2)
print(point)  # Point(x=1, y=2)
```

不使用 `@dataclass` 时，需要手动编写初始化方法：

```python
class Point:
    def __init__(self, x: int, y: int):
        self.x = x
        self.y = y
```

`@dataclass` 适合配置、坐标、记录等数据类，但不会阻止你继续添加普通方法。

## `@abstractmethod`

`@abstractmethod` 来自 `abc` 模块，用于声明子类必须实现的方法。包含抽象方法的类还需要继承 `ABC`，否则抽象约束不会完整生效。

```python
from abc import ABC, abstractmethod

class VecEnv(ABC):
    @abstractmethod
    def reset(self):
        pass

class RobotEnv(VecEnv):
    pass

env = RobotEnv()  # TypeError：RobotEnv 没有实现 reset()
```

子类实现全部抽象方法后才能实例化：

```python
class RobotEnv(VecEnv):
    def reset(self):
        print("环境已重置")

env = RobotEnv()
```

`@abstractmethod` 并非“加不加都一样”。没有它时，遗漏方法通常要等到运行并调用该方法后才会暴露；有了它，Python 会在实例化子类时立即报错。它检查的是接口是否实现，不负责提供具体逻辑。

抽象方法也可以包含默认实现。子类仍需覆盖该方法，并可通过 `super()` 复用其中的代码。

## `@staticmethod`

`@staticmethod` 把函数放在类的命名空间中，但调用时不会自动传入实例 `self` 或类 `cls`。

```python
class Vector:
    @staticmethod
    def is_valid(x: float, y: float) -> bool:
        return x != 0 or y != 0

print(Vector.is_valid(1, 0))  # True
```

静态方法既可以通过类调用，也可以通过实例调用，但通常推荐通过类调用，以明确它不依赖实例状态。

```python
vector = Vector()
vector.is_valid(0, 0)  # 可以调用，但 Vector.is_valid(...) 更清晰
```

适合使用 `@staticmethod` 的情况：

- 方法在概念上属于这个类。
- 方法不读取或修改实例属性，因此不需要 `self`。
- 方法不读取或修改类属性，因此不需要 `cls`。

如果函数与类没有明显关系，直接定义为模块级函数通常更合适。

## 三者的区别

| 装饰器 | 放置位置 | 自动传入参数 | 主要作用 |
| --- | --- | --- | --- |
| `@dataclass` | 类定义上方 | 不适用 | 自动生成数据类的常用方法 |
| `@abstractmethod` | 方法定义上方 | 取决于方法类型，普通实例方法传入 `self` | 强制子类实现指定接口 |
| `@staticmethod` | 方法定义上方 | 无 | 定义不依赖实例和类状态的方法 |

`@staticmethod` 还可以与 `@abstractmethod` 组合，要求子类实现一个静态方法。装饰器顺序不能颠倒：

```python
class Parser(ABC):
    @staticmethod
    @abstractmethod
    def parse(text: str):
        pass
```

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

# Python 中 `class` 套 `class` 的实例化

## 嵌套类不会自动实例化

在一个类中定义另一个类，只是在外层类的命名空间中创建了一个类对象。创建外层类的实例时，Python 不会自动创建内层类的实例。

```python
class RobotCfg:
    class Control:
        def __init__(self):
            print("创建 Control 实例")

cfg = RobotCfg()  # 不会输出任何内容
```

此时，`RobotCfg.Control` 表示内层类本身，而不是内层类的实例：

```python
print(RobotCfg.Control)       # <class '__main__.RobotCfg.Control'>

control = RobotCfg.Control()  # 此时才会实例化，并调用 Control.__init__()
```

## 外层实例访问到的仍可能是类

内层类是外层类的类属性，因此也可以通过外层实例找到它，但访问结果仍是类对象。

```python
cfg = RobotCfg()

print(cfg.Control is RobotCfg.Control)  # True
control = cfg.Control()                 # 手动实例化内层类
```

`cfg.Control` 能被访问，不代表 `Control` 已经随 `cfg` 一起实例化。判断时要看后面是否调用了括号 `()`：

- `cfg.Control`：取得内层类。
- `cfg.Control()`：创建内层类的实例。

## 需要自动创建时应显式实例化

如果希望每个外层实例都拥有独立的内层实例，应在外层类的 `__init__()` 中主动创建。

```python
class RobotCfg:
    class Control:
        def __init__(self):
            self.stiffness = 20.0

    def __init__(self):
        self.control = self.Control()

cfg_a = RobotCfg()
cfg_b = RobotCfg()

print(cfg_a.control.stiffness)          # 20.0
print(cfg_a.control is cfg_b.control)   # False
```

这里需要区分大小写不同的两个名称：

- `Control` 是内层类。
- `control` 是保存在外层实例中的内层类实例。

## 配置类中的常见情况

一些框架使用嵌套类组织配置，但内层通常只保存类属性，因此不一定需要实例化。

```python
class RobotCfg:
    class Control:
        stiffness = 20.0
        damping = 0.5

print(RobotCfg.Control.stiffness)  # 20.0
```

这种写法把 `Control` 当作配置的命名空间。是否需要实例化取决于框架如何读取配置，不能仅凭 `class` 套 `class` 判断。

## 内层类不会自动持有外层实例

实例化内层类时，Python 不会自动传入外层实例。内层方法中的 `self` 指向内层类的实例。

```python
class Robot:
    def __init__(self, name: str):
        self.name = name

    class Controller:
        def show_name(self):
            return self.name  # self 是 Controller 实例，不是 Robot 实例
```

如果内层实例需要访问外层实例，应显式传入并保存：

```python
class Robot:
    def __init__(self, name: str):
        self.name = name
        self.controller = self.Controller(self)

    class Controller:
        def __init__(self, robot):
            self.robot = robot

        def show_name(self):
            return self.robot.name

robot = Robot("A1")
print(robot.controller.show_name())  # A1
```

# `dim=-1`：沿最后一个维度操作

`dim` 用来指定对张量的哪一个维度进行操作。`dim=-1` 表示最后一个维度。

> `dim=-1` 指的是最后一个维度，不是“最后一行”。

先看一个二维张量：

```python
import torch

x = torch.tensor([
    [1.0, 2.0, 3.0],
    [4.0, 5.0, 6.0],
])  # shape: (2, 3)
```

`shape=(2, 3)` 的含义是：

- `dim=0` 的长度为 2，表示有两行。
- `dim=1` 的长度为 3，表示每一行有三个数。
- 这是二维张量，所以最后一个维度是 `dim=1`。因此，`dim=-1` 和 `dim=1` 完全相同。

## `torch.sum(x, dim=-1)`

`sum` 表示求和。因为最后一个维度对应“每一行中的三个数”，所以 `dim=-1` 会分别计算：

```text
[1, 2, 3] → 1 + 2 + 3 = 6
[4, 5, 6] → 4 + 5 + 6 = 15
```

最终得到：

```python
torch.sum(x, dim=-1)
# tensor([6., 15.])
```

如果改成 `dim=0`，则会把两行中相同位置的数相加：

```text
1 + 4 = 5
2 + 5 = 7
3 + 6 = 9
```

```python
torch.sum(x, dim=0)
# tensor([5., 7., 9.])
```

对比结果：

| 代码 | 实际计算 | 结果 |
| --- | --- | --- |
| `torch.sum(x, dim=0)` | 两行之间对应位置相加 | `[5, 7, 9]` |
| `torch.sum(x, dim=1)` | 每一行内部求和 | `[6, 15]` |
| `torch.sum(x, dim=-1)` | 与 `dim=1` 相同 | `[6, 15]` |

## `torch.softmax(x, dim=-1)`

`softmax` 不会求和，而是把一组数转换成总和为 1 的概率。`dim=-1` 表示分别转换每一行：

```python
torch.softmax(x, dim=-1)

# tensor([
#     [0.0900, 0.2447, 0.6652],  # 第一行转换后的和为 1
#     [0.0900, 0.2447, 0.6652],  # 第二行转换后的和为 1
# ])
```

因此，下面两行代码虽然都写了 `dim=-1`，执行的操作并不相同：

```python
torch.sum(x, dim=-1)      # 每一行内部求和
torch.softmax(x, dim=-1)  # 每一行分别转换为概率
```

`sum` 或 `softmax` 决定“执行什么操作”，`dim=-1` 只决定“对哪一组数执行”。

## 更高维张量中的 `dim=-1`

负数维度从后向前计数：

| 张量形状 | 最后一个维度 | `dim=-1` 操作的分组 |
| --- | --- | --- |
| `(2, 3)` | 长度为 3 的维度 | 每组包含 3 个数 |
| `(2, 3, 4)` | 长度为 4 的维度 | 每组包含 4 个数 |
| `(8, 10, 64)` | 长度为 64 的维度 | 每组包含 64 个数 |

例如，张量形状是 `(2, 3, 4)` 时：

```python
y = torch.ones(2, 3, 4)
torch.sum(y, dim=-1).shape
# torch.Size([2, 3])
```

最后一个维度中的 4 个数被求和，所以形状从 `(2, 3, 4)` 变成 `(2, 3)`。

NumPy 中对应的参数通常叫 `axis`，例如 `np.sum(x, axis=-1)`。
