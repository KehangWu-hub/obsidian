---
tags:
  - ROS2
  - Ubuntu
  - Gazebo
date: 2025-03-10
---

![cover](../../Assets/gazebo-startup/cover.png)

# 问题描述

版本为ubuntu22.04，ros2 humble

通过`sudo apt install gazebo和sudo apt install ros-humble-gazebo-*`安装gazebo后，无法在终端中通过gazebo命令或者`ros2 launch gazebo_ros gazebo.launch.py`命令启动gazebo，只会卡在加载的那个方框进不去。（直接点击gazebo图标，是可以顺利打开的）。

网上都说，要先下载模型到~/.gazebo/models里，之后再重启就好了，我也照办了，但还是卡在那里。

通过gazebo --verbose打印启动时详细日志时出现以下问题：`[Err] [http://RTShaderSystem.cc:480] Unable to find shader lib. Shader generating will fail. Your GAZEBO_RESOURCE_PATH is probably improperly set. Have you sourced <prefix>/share/gazebo/setup.bash?`

# 解决办法

最终解决办法：

打开.bashrc，在里面添加以下两行：

```bash
source /opt/ros/humble/setup.bash
source /usr/share/gazebo/setup.bash
```

按道理说这两行不用手动添加，可能是我不知道啥时候手贱给删了，也可能是下载的时候出现了bug。

气死老子了。
