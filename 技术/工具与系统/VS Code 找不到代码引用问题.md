---
tags:
  - VSCode
date: 2026-09-08
---

## 问题现象

编写 Python 时，在 VS Code 中选中一个函数（比如 `step`），但在其他类中调用它的地方（比如 `self.bandit.step()`）却没有高亮显示。右键点击“查找所有引用”也毫无反应，让人误以为这个函数没被用过。

## 根本原因：Python 的“动态类型”特性

Python 是一门非常自由的语言，声明变量时不需要指定类型。

当你写下 `def __init__(self, bandit):` 时，VS Code 的代码分析器（Pylance/IntelliSense）并不知道传入的 `bandit` 到底是个什么对象（是数字？是字符串？还是老虎机？）。因为不知道身份，VS Code 为了避免报错，干脆就不进行跨文件/跨类的高亮关联。

## 终极解决办法：类型提示 (Type Hint)

在定义参数时，顺手给它“贴个标签”，明确告诉 VS Code 它的真实身份。

- **修改前（VS Code 无法识别）：**

    ```python
    def __init__(self, bandit):
    ```

- **修改后（VS Code 瞬间变聪明）：**

    ```python
    def __init__(self, bandit: BernoulliBandit):
    ```

**(加上 `: BernoulliBandit` 后，VS Code 瞬间就能把两个类关联起来，代码高亮、`Ctrl + 点击` 跳转、自动补全全部复活)**

## 备用方案：暴力搜索法

如果是在阅读别人写的老代码（没有类型提示），别依赖高亮来判断函数有没有被调用。直接使用：

1.  **单文件搜索**：`Ctrl + F`
2.  **全局搜索（最管用）**：`Ctrl + Shift + F`，在整个工程文件夹里直接搜索函数名。

