---
tags:
  - ROS2
  - Ubuntu
  - Linux
date: 2025-03-04
---

这篇记录了尝试将带有闭链结构的机器人导入Gazebo仿真踩坑的过程。

# 进入正题

通过查阅更多的资料与教程，了解到要将solidworks导入ros中仿真，通常是按照以下几步：

1. 通过urdf插件，将sw文件转换成urdf
2. 将urdf在Rviz中可视化（**更新：其实rviz只是一个可视化工具，和gazebo不存在先后关系，只能说当时初学者啥也不懂，现在回头来看当时的心路历程颇有感慨**）
3. 将urdf导入Gazebo进行仿真

使用插件从solidworks导出urdf文件并导入ros中，已经非常成熟且有很多教程了。然而ros2和ros有较大的差别，urdf的使用也有不少的差异。最重要的是**urdf插件导出的urdf模型并不能直接在ros2中使用。** 国内的教程非常少，且每个实践下来都有各自的问题，我将多个教程和帖子连起来看，互相补充，得到一套方法。

将整个机器人导入有点复杂，我决定先用一个最简单的舵机模型，把整个过程实现一遍，先打通流程，然后再机器人模型导入，这样万无一失，如果在这个过程中踩坑了，由于是简单的舵机模型，处理起来要方便很多

（事后来看，这个决定**极其错误**，想法很美好，但忽略了一个问题：复杂模型和简单模型差别太大，很多坑只有复杂模型会遇到）

找了个视频教程，教程中使用的是Moviet2，过程极其顺利，舵机很快就在Moviet2中可视化出来，并可以通过拉条控制旋转

![robot-demo](../../Assets/closed-chain-gazebo/robot-demo.png)

信心大增，开始正式导入表情机器人模型

# 踩坑一

按照下面方式建的功能包理论上是很常规的

```bash
ros2 pkg create facerobot_description --build-type ament_python
```

事实上，在一开始用舵机做实验的时候，确实也没问题，但是换成表情机器人后，后续会出现无法显示模型的问题

简直是莫名其妙

上网查找原因，发现很多人也遇到这种情况，且国内没有解决方法，解决方法就是不自己建功能包，而是选择git clone一个现有的包，把其中的urdf和meshes文件替换成自己的，站在前人的肩膀上

受启发的帖子：[sw2urdf插件导出urdf模型并在ros2-rviz2显示](https://zhuanlan.zhihu.com/p/465398486)

我选择clone了一个现有的仓库

```python
git clone https://github.com/olmerg/lesson_urdf.git
```

（**更新：如果无法clone，也可以直接去github上面下载之后解压缩，把文件夹放在工作空间下面，也是一样的**）

并对其进行修改：

1. 修改setup.py文件
2. 修改package.xml文件
3. 修改meshes文件夹
4. 修改urdf文件夹（**更新：这一步注意，Ctrl+F可以搜索多条路径的前缀然后一次性修改**）
5. 修改launch文件夹

主要是修改里面的文件夹名称和路径名称（**更新：路径要用绝对路径，这时候后来发现的，用相对路径一开始是可以的，可以顺利加载到rviz里面，但是加载到gazebo里面的时候会出现问题，原因：我们urdf文件引用了stl文件，引用形式为 package://；而gazebo会将urdf转为sdf文件，引用形式自动转为model://，所以用绝对路径可以规避这一点**）

一通修改之后，相当于是建好了自己的功能包

终于，在Rviz中可视化出来了

此时发现Link旁边没有相应的拉条，没有办法拖动拉条进行相应的让机器人进行面部动作

此时意识到导出的urdf文件有问题，回头去看，发现原来是舵机旋转需要的轴，我以为选择Automatically他会自动生成，结果并没有，还是需要手动添加（那我请问这个自动生成的选项是摆设吗？）

总之，回头将轴手动一一添加上，再回到ros2中，发现一个更为致命的问题

（**更新：不仅要手动添加轴，还要手动添加每一个joint的原点和坐标系，一个都不能偷懒，自动生成的坐标系很有可能有问题**）

# 踩坑二

(刷到一个帖子介绍了一个网站，可以预览urdf文件，预览发现有问题就可以及时退回修改，没问题就可以正式导到Rviz2和Gazebo中进行下一步操作，这个网站可以在windows里打开，验证完毕再切回Ubuntu。网站链接：[URDF Viewer Example](https://gkjohnson.github.io/urdf-loaders/javascript/example/bundle/))

**urdf不能实现闭链！！！**

表情机器人里很多结构都是闭链结构，但是urdf不能实现这种结构，urdf只能a→b→c→d，但不能a→b→c→d→a，我学urdf的时候也没人告诉我，以至于当我在ros2里拖动拉条的时候才发现不对

![urdf-notes](../../Assets/closed-chain-gazebo/urdf-notes.png)

想到的解决方法：

1. 通过添加支持闭链的urdf插件，从而使得urdf具备实现闭链的能力
2. 将urdf转成sdf。**SDF格式**是URDF格式的升级版，支持描述并联结构。（**更新：当时这个想法不对，urdf转sdf其实是导入到gazebo的必经之路，urdf无法直接导入gazebo。所以正确做法是修改urdf文件。拆掉闭环中的一个关节，让机器人变成一个开环机器人。一般来说，闭环结构中肯定会有一个无动力的关节（有的时候可能不止一个），拆掉这个关节即可。在URDF中，则体现为删除一个joint描述。生成urdf文件后，然后将上一步删除的joint添加上 `<gazebo>` 标签加回URDF中。这使得URDF再转换为SDF的过程中，这一块joint会被添加回生成的SDF中，从而补全闭环。这里将就看吧，后续会出一个从头到尾的详细教程**）

尝试方法一，翻了好几个帖子，都是只介绍推出了支持闭链的urdf插件，却不提供安装包，pass

尝试方法二，SDF建模方法有三种：

1. 利用Gazebo模型编辑器建模，保存即可得到SDF文件
2. 从零开始编写一个新的SDF文件，并导入个性化的网格模型；
3. 用solidworks_urdf插件，获取网格模型和相关参数，填充到SDF框架中

毫无疑问我们选择第三种，URDF和SDF在结构上十分相似（实际上后者就是前者的加强版），因此所导出的网格文件、位置、惯量等参数，都可以很方便地移植到SDF模型中。

bug:打不开gazebo，显示找不到world文件。

解决办法：要在setup.py里添加复制world文件到install目录下的代码

bug：gazebo打开了，但是没有模型加载出来，报错：[ERROR] [spawn_entity.py-4]: process has died [pid 7487, exit code 1, cmd '/opt/ros/humble/lib/gazebo_ros/spawn_entity.py -topic /robot_description -entity jaw --ros-args'].把cat命令换成xacro，上面报错消失了，gazebo卡在加载界面很长时间，然后进去了发现还是没有模型，出现很多这种报错：[gzclient-3] [Wrn] [FuelModelDatabase.cc:313] URI not supported by Fuel [model://jaw_urdf/meshes/visual/smile4_1_L.STL][gzclient-3] [Wrn] [SystemPaths.cc:459] File or path does not exist [""] [model://jaw_urdf/meshes/visual/smile4_1_L.STL][gzclient-3] [Err] [Visual.cc:2956] No mesh specified。

原因：我们urdf文件引用了stl文件，引用形式为 package://；而gazebo会将urdf转为sdf文件，引用形式自动转为model://

解决方法：在urdf中，每个mesh标签都直接使用绝对路径

bug：机器人上各个零部件在诡异抽动，明明并没有添加任何ros_control

原因：应该是惯性矩阵的问题

（**网上有误人子弟的帖子，说是把有关转动惯量的部分全部注释掉就好了，滑天下之大稽，还不止一个两个脑残这么教，注释完了gazebo里面也就不显示这个零件了**）

惯性矩阵是不能随便修改的，很容易就不符合主惯性矩三角不等式，即ixx + iyy ≥ izz，ixx + izz ≥ iyy，iyy + izz ≥ ixx

解决办法：尝试添加控制器（尝试中）

# 写在很久之后

其实到最后我都没有写完这一篇帖子，因为在我找到最终解决办法之前，我就已经放弃了这个课题。其实人生就是这样，有很多的残缺。站在今天，codex应该可以很轻松地帮我解决这个问题了，但我当年的苦心求索并非没有意义。人是需要努力的，就算努力最终不一定有结果，但在这个过程中人能学到很多。
