---
tags:
  - Linux
  - Ubuntu
date: 2026-08-13
---

## 打开终端

在 Ubuntu 中按 `Ctrl+Alt+T`，可以快速打开终端。

如果想直接在某个文件夹中打开终端，可以在文件管理器中进入该文件夹，右键单击空白处，然后选择“在终端中打开”。

## 查看剩余存储空间

查看各文件系统的总容量、已用空间和剩余空间：

```bash
df -h
```

其中 `-h` 会用 GB、MB 等易读单位显示容量。通常重点查看挂载点为 `/` 的一行。

## 查看与切换文件夹

显示当前所在路径：

```bash
pwd
```

列出当前文件夹中的内容：

```bash
ls
```

进入指定文件夹：

```bash
cd folder_name
```

常用的路径切换方式：

```bash
cd ..       # 返回上一级文件夹
cd ~        # 进入当前用户的主目录
cd /        # 进入根目录
cd -        # 返回上一次所在的文件夹
cd "folder name"  # 进入名称中含空格的文件夹
```

路径以 `/` 开头时是绝对路径；不以 `/` 开头时，通常是相对于当前文件夹的相对路径。输入路径时可以按 `Tab` 键自动补全。

## 创建文件和文件夹

创建文件夹：

```bash
mkdir folder_name
mkdir -p parent/child
```

`-p` 会在需要时一并创建不存在的父文件夹。

创建空文件：

```bash
touch file_name.txt
```

## 复制、移动与删除

复制文件或文件夹：

```bash
cp source.txt destination.txt
cp -r source_folder destination_folder
```

移动文件，或为文件重命名：

```bash
mv old_name.txt new_name.txt
mv file.txt destination_folder/
```

删除文件或文件夹：

```bash
rm file_name.txt
rm -r folder_name
```

> `rm` 删除的内容通常不会进入回收站，使用 `rm -r` 前应仔细确认路径。不要在不理解含义时使用 `rm -rf`。

## 其他终端细节

清空当前终端画面：

```bash
clear
```

终端中的矩形光标表示当前字符位置。可以把==矩形左边缘==理解为常见竖线光标所在的位置。

按方向键 `↑` 可以调出上一条命令，适合快速修改并重新执行。

`Ctrl + C` 用于终止当前命令，不是复制。 Linux 终端中的复制和粘贴通常是 `Ctrl + Shift + C` 与 `Ctrl + Shift + V`。

输入命令或文件名的一部分后按 `Tab`，可以自动补全。

## 安装 Deb 软件包

1. 从软件官网下载安装包，通常选择适合当前架构的 `.deb` 文件，例如 `amd64`。
2. 在下载目录打开终端。
3. 使用 `apt` 安装本地软件包，它会同时处理依赖：

```bash
cd ~/下载
sudo apt install ./package_name_amd64.deb
```

也可以使用 `dpkg`：

```bash
sudo dpkg -i package_name_amd64.deb
sudo apt --fix-broken install
```

## 卸载软件

保留配置文件：

```bash
sudo apt remove package_name
```

同时删除系统级配置文件：

```bash
sudo apt purge package_name
```

如果不确定软件包的准确名称，可以先搜索：

```bash
apt list --installed 2>/dev/null | grep -i keyword
```

## Shell、Bash 与终端的区别

这三个词经常一起出现，但含义不同：

- **终端（Terminal）**：显示文字、接收键盘输入的窗口，例如 Ubuntu 的“终端”应用。
- **Shell**：运行在终端里的命令解释器。它读取 `cd`、`ls` 等命令，再让系统执行。
- **Bash**：一种具体的 Shell，也是 Linux 中最常见的 Shell 之一。

可以把它们理解为：

> **终端是窗口，Shell 是窗口里负责理解命令的程序，Bash 是这个程序的一种具体实现。**

“Shell”是一个类别，不是某一个固定程序。除了 Bash，还有 `sh`、`zsh`、`fish` 等 Shell。它们能执行许多相同的基础命令，但脚本语法和功能不完全相同。

### Shell 脚本和 Bash 脚本

“Shell 脚本”是一个统称，指写给某种 Shell 执行的脚本；“Bash 脚本”则明确表示脚本使用 Bash 语法。

例如，脚本第一行是：

```bash
#!/usr/bin/env bash
```

这表示脚本应交给 Bash 解释。因此更准确地说，它是一个 Bash 脚本。

下面两个命令不一定等价：

```bash
bash hello.sh
sh hello.sh
```

前者明确使用 Bash，后者使用系统提供的 `sh`。在 Ubuntu 中，`sh` 通常不是 Bash；如果脚本使用了 Bash 特有语法，用 `sh` 执行可能报错。因此，带有 Bash shebang 的脚本通常使用 `bash hello.sh` 或 `./hello.sh` 执行。

## Shell 脚本（`.sh` 文件）

`.sh` 文件通常是 **Shell 脚本**：把原本要在终端中逐条输入的命令写进一个文本文件，再一次性执行。它不仅能依次执行命令，还能使用变量、条件判断、循环和函数。

例如，创建 `hello.sh`：

```bash
#!/usr/bin/env bash

echo "开始执行"
mkdir -p demo
touch demo/example.txt
ls -l demo
echo "执行完成"
```

第一行称为 **shebang**，表示直接运行此文件时使用 Bash 解释它。空行不会执行，以 `#` 开头的其他行是注释。

### 执行脚本

最简单的方式是让 Bash 读取并执行脚本，不要求文件有执行权限：

```bash
bash hello.sh
```

脚本中的命令默认按照书写顺序执行，但并非简单地“无条件执行每一行”：条件判断、循环等会控制执行流程。此外，某条命令失败后，Bash 默认通常仍会继续执行后面的命令。

### 脚本在哪个文件夹中执行

脚本里的相对路径以**运行命令时终端所在的文件夹**为基准，不一定以 `.sh` 文件所在的文件夹为基准。例如：

```bash
cd ~/Downloads
bash ~/scripts/hello.sh
```

此时脚本里的 `demo/example.txt` 会指向 `~/Downloads/demo/example.txt`。可以在执行前用 `pwd` 确认当前文件夹。

### 给脚本传入参数

脚本可以接收命令行参数：

```bash
#!/usr/bin/env bash

name="$1"
echo "你好，$name"
```

执行：

```bash
bash hello.sh "小明"
```

其中 `$1` 是第一个参数，`$2` 是第二个参数，`$@` 表示全部参数。引用变量时通常应写成 `"$name"`，避免内容中的空格被错误拆分。

### 执行前的安全检查

脚本可以执行当前用户有权限执行的所有命令；使用 `sudo` 后还可能修改系统文件。因此，不要直接运行来源不明的脚本。先阅读内容：

```bash
cat hello.sh
less hello.sh
```

### 补充：为什么有时会看到 `source`

对于刚开始使用 Shell 脚本的人，通常只需要使用：

```bash
bash hello.sh
```

有时教程里还会出现：

```bash
source settings.sh
```

二者最容易理解的区别是：

- `bash settings.sh`：开一个新的 Shell 去执行这个 `.sh`脚本。
- `source settings.sh`：在当前终端里执行，脚本对当前终端所做的改变会保留下来。

例如 `change-folder.sh` 的内容是：

```bash
cd ~/Downloads
```

如果运行：

```bash
bash change-folder.sh
```

脚本执行完以后，当前终端仍然停留在原来的文件夹。因为 Bash 相当于临时开启了一个“小终端”来执行脚本，执行完便关闭了。

如果运行：

```bash
source change-folder.sh
```

当前终端就会切换到 `~/Downloads`，因为这条 `cd` 命令直接作用在当前终端中。

因此可以先这样记： **运行一般脚本用 `bash`；只有教程明确要求加载环境配置时，才使用 `source`。**

`source settings.sh` 还有一种等价的简写：`. settings.sh`。
