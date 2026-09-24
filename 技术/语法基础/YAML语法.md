---
tags:
  - yaml
  - ROS2
  - 编程
date: 2025-03-19
---

![cover](../../Assets/yaml-syntax/cover.jpg)

# 什么是YAML

我一向不喜欢故弄玄虚的定义，什么“数据序列化格式”，听着就头疼。我对yaml的理解就是一种配置语言，即**一种用来编写配置文件的格式**。类似于xml和json。

在ROS2 Control框架中，YAML文件作为核心配置工具，贯穿了控制器管理、硬件接口定义、参数传递和仿真集成等多个关键环节。

# 基本语法

以 k: v 的形式来表示键值对的关系，**冒号后面必须有一个空格**

\#表示注释

# 格式

## 对象和键值对

通过 k: v 的方式表示对象或者键值对，冒号后必须要加一个空格

```YAML
Name: ke
Sex: male
School: NPU
```

通过缩进来表示对象的多个属性

```YAML
People:
   Name: ke
   Sex: male
   School: NPU
```

也可以写成

```YAML
people: {name: ke, sex: male,School: NPU}
```

较为复杂的对象格式，可以使用问号加一个空格代表一个复杂的 key，配合一个冒号加一个空格代表一个 value

```YAML
?
    - complexkey1
    - complexkey2
:
    - complexvalue1
    - complexvalue2
```

意思即对象的属性是一个数组 [complexkey1,complexkey2]，对应的值也是一个数组 [complexvalue1,complexvalue2]

## 数组

数组（或者列表）中的元素采用 - 表示

```YAML
people:
    - A
    - B
    - C
```

或

```YAML
people: [A, B, C]
```

例子：

```YAML
companies:
    -
        id: 1
        name: company1
        price: 200W
    -
        id: 2
        name: company2
        price: 500W
```

或

```YAML
companies: [{id: 1,name: company1,price: 200W},{id: 2,name: company2,price: 500W}]
```

意思是 companies 属性是一个数组，每一个数组元素又是由 id、name、price 三个属性构成。

## 纯量

纯量是最基本的，不可再分的值，包括：

* 字符串
* 布尔值
* 整数
* 浮点数
* Null
* 时间
* 日期

```YAML
boolean:
    - TRUE  #true,True都可以
    - FALSE  #false，False都可以
float:
    - 3.14
    - 6.8523015e+5  #可以使用科学计数法
int:
    - 123
    - 0b1010_0111_0100_1010_1110    #二进制表示
null:
    nodeName: 'node'
    parent: ~  #使用~表示null
string:
    - 哈哈
    - 'Hello world'  #可以使用双引号或者单引号包裹特殊字符
    - newline
      newline2    #字符串可以拆成多行，每一行会被转化成一个空格
date:
    - 2018-02-17    #日期必须使用ISO 8601格式，即yyyy-MM-dd
datetime:
    -  2018-02-17T15:02:31+08:00    #时间使用ISO 8601格式，时间和日期之间使用T连接，最后使用+代表时区
```

# 在 Python 中读取 YAML

YAML 文件写好后，可以用第三方库 **PyYAML** 的 `yaml.safe_load()` 把它读成 Python 数据。首次使用可安装：`pip install pyyaml`。

假设 `config.yaml` 的内容是：

```yaml
robot:
  name: go2
  joints: [hip, thigh, calf]
```

用 Python 读取：

```python
import yaml

with open("config.yaml", "r", encoding="utf-8") as f:
    config = yaml.safe_load(f)

print(config["robot"]["name"])    # go2
print(config["robot"]["joints"])  # ['hip', 'thigh', 'calf']
```

`open()` 打开文件，`yaml.safe_load(f)` 解析文件内容；上例中，YAML 的键值对变成 Python 字典，列表变成 Python 列表。`with` 会在读完后关闭文件。

这里的 `safe_load()` 是 **PyYAML 的 Python 函数**，不是 YAML 文件中的语法。它只构造受支持的安全类型，不会按 YAML 内容任意创建 Python 对象；读取配置时优先使用它。空文件会返回 `None`。

# 结合ROS2 Control

## 控制器配置

控制器定义：明确控制器名称、类型（如位置控制、轨迹控制）及关联的关节列表。例如：

```YAML
controller_manager:
  ros__parameters:
    my_position_controller:
      type: position_controllers/JointPositionController
      joints: [joint1, joint2]
```

此配置定义了名为my_position_controller的位置控制器，作用于joint1和joint2。

PID参数配置：在控制算法中设置比例、积分、微分增益。例如：

```YAML
joint1_position_controller:
  type: effort_controllers/JointPositionController
  pid: {p: 2000.0, i: 100, d: 500.0}
```

此类配置常见于需要精确控制的场景（如机械臂关节控制）。

## 硬件接口抽象与通信

硬件接口类型定义：指定每个关节的指令接口（如位置、速度、力矩）和状态接口。例如：

```YAML
hardware_interface:
  joints:
    - name: joint1
      command_interface: position
      state_interface: position
```

## 例子

以一个两轮机器人的ros2_control配置文件为例

```YAML
controller_manager:
  ros__parameters:
    update_rate: 100  # 控制器更新频率100Hz（10ms周期）
    use_sim_time: true  # 使用仿真时钟而非真实时间
    #控制器注册声明
    fishbot_joint_state_broadcaster: # 关节状态控制器
      type: joint_state_broadcaster/JointStateBroadcaster
      use_sim_time: true
    fishbot_effort_controller: # 力控制器声明
      type: effort_controllers/JointGroupEffortController
    fishbot_diff_drive_controller: # 两轮差速控制器声明
      type: diff_drive_controller/DiffDriveController

# 力控制器详细配置
fishbot_effort_controller:
  ros__parameters:
    joints: # 控制的关节列表
      - left_wheel_joint
      - right_wheel_joint
    command_interfaces:  # 指令接口类型：力控制
      - effort
    state_interfaces: # 状态反馈接口
      - position # 位置反馈
      - velocity # 速度反馈
      - effort # 实际力矩反馈

# 差速驱动控制器核心参数
fishbot_diff_drive_controller:
  ros__parameters:
    # 运动学参数
    left_wheel_names: ["left_wheel_joint"]
    right_wheel_names: ["right_wheel_joint"]

    wheel_separation: 0.17 # 轮间距0.17米（左右轮中心距）
    #wheels_per_side: 1  # actually 2, but both are controlled by 1 signal
    wheel_radius: 0.032 # 轮半径0.032米（影响速度计算）
    # 参数容差补偿
    wheel_separation_multiplier: 1.0 # 轮间距修正系数
    left_wheel_radius_multiplier: 1.0 # 左轮半径修正
    right_wheel_radius_multiplier: 1.0 # 右轮半径修正
    # 里程计配置
    publish_rate: 50.0  # 里程计发布频率50Hz
    odom_frame_id: odom # 里程计坐标系
    base_frame_id: base_footprint # 机器人基坐标系
    pose_covariance_diagonal : [0.001, 0.001, 0.0, 0.0, 0.0, 0.01] # 位姿协方差（XYθ
    twist_covariance_diagonal: [0.001, 0.0, 0.0, 0.0, 0.0, 0.01] # 速度协方差
    # 控制模式
    open_loop: true # 开环控制（不依赖轮速反馈）
    enable_odom_tf: true # 发布odom到base_footprint的TF变换

    cmd_vel_timeout: 0.5 # 速度指令超时时间0.5秒（超时停止）
    #publish_limited_velocity: true
    use_stamped_vel: false # 是否使用带时间戳的速度指令
    #velocity_rolling_window_size: 10
```

关键参数关系示意图:

[cmd_vel] → DiffDriveController

           ├─ 运动学计算 → 力矩指令 → EffortController → 轮毂电机
           └─ 里程计推算 ← 轮速反馈
