---
tags:
  - Python
  - CPP
  - 编程
  - 语法对照
date: 2026-08-23
---

> 用来纠正 Python 和 C++ 之间最容易串台的写法。示例默认使用 Python 3 和 C++17；标明 C++20 的写法除外。

# 基础写法

| 目的 | Python | C++ |
| --- | --- | --- |
| 输出 | `print(x)` | `std::cout << x << '\n';` |
| 输入整数 | `x = int(input())` | `int x; std::cin >> x;` |
| 布尔值 | `True`、`False` | `true`、`false` |
| 逻辑运算 | `and`、`or`、`not` | `&&`、`\|\|`、`!` |
| 条件分支 | `if / elif / else` | `if / else if / else` |
| 定义函数 | `def add(a, b):` | `int add(int a, int b) {}` |
| 动态数组 | `list` | `std::vector<T>` |
| 哈希表 | `dict` | `std::unordered_map<K, V>` |
| 集合 | `set` | `std::unordered_set<T>` |
| 长度 | `len(a)` | `a.size()` |
| 是否为空 | `not a` | `a.empty()` |
| 末尾添加 | `a.append(x)` | `a.push_back(x)` |
| 当前对象 | `self` | `this` |

# 代码块与语句

Python 用缩进表示代码块，条件后有冒号，语句末尾通常没有分号。

```python
if score >= 60:
    print("pass")
else:
    print("fail")
```

C++ 用花括号表示代码块，多数语句以分号结尾。

```cpp
if (score >= 60) {
    std::cout << "pass\n";
} else {
    std::cout << "fail\n";
}
```

不要在 C++ 的条件后顺手加分号：

```cpp
if (score >= 60); {  // if 的主体是空语句，后面的代码块总会执行
    std::cout << "pass\n";
}
```

# 类型与初始化

Python 的变量名可以先后绑定不同类型的对象。

```python
x = 10
x = "hello"
```

C++ 变量的类型在编译期确定，之后不能改变。

```cpp
int x = 10;
// x = "hello";  // 类型不匹配

auto y = 3.14;   // y 被推导为 double，此后类型仍是 double
```

`auto` 只是省略显式类型，不会让 C++ 变成动态类型。

C++ 局部基本类型应主动初始化：

```cpp
int count{};                  // 0
double total{};               // 0.0
std::vector<int> numbers{};   // 空 vector
```

读取未初始化的局部基本类型会产生未定义行为。

# 条件与比较

## 真值判断

Python 的空字符串、空容器、`0` 和 `None` 都可直接用于条件判断。

```python
if items:
    print("not empty")
```

C++ 的标准容器不能直接转换为布尔值，应明确检查是否为空。

```cpp
if (!items.empty()) {
    std::cout << "not empty\n";
}
```

Python 的 `None` 没有统一的 C++ 对应物。空指针用 `nullptr`；“一个值可能不存在”通常用 `std::optional<T>`。

## 连续比较

Python 支持连续比较：

```python
if 0 < x < 10:
    ...
```

C++ 必须拆成两个条件：

```cpp
if (0 < x && x < 10) {
    // ...
}
```

`0 < x < 10` 在 C++ 中会先计算 `0 < x`，得到 `true` 或 `false`，再拿这个布尔值和 `10` 比较。由于 `0` 和 `1` 都小于 `10`，这个条件恒为真，并不表示数学上的区间判断。

## 赋值与相等

两门语言都用 `==` 比较是否相等。

```cpp
if (x = 1) {   // 把 1 赋给 x，再把结果 1 当作 true
}

if (x == 1) {  // 比较 x 是否等于 1
}
```

C++ 的赋值表达式有结果，所以 `if (x = 1)` 可以编译，但通常是把 `==` 写成了 `=`。

Python 不允许把普通赋值语句直接放进条件。确实需要边赋值边判断时使用 `:=`：

```python
if (size := len(items)) > 0:
    print(size)
```

## `==` 与 `is`

```python
a == b       # 值是否相等
a is b       # 是否为同一个对象
x is None    # 判断 None
```

不要用 `is` 比较数字、字符串或容器的内容。

C++ 中 `==` 的具体含义由类型决定；比较对象地址时才使用指针。

```cpp
a == b             // 比较值
&a == &b           // 比较地址
ptr == nullptr     // 判断空指针
```

# 数值运算

## 除法

| 表达式 | Python | C++ |
| --- | --- | --- |
| `5 / 2` | `2.5` | 两边都是整数时结果为 `2` |
| `5 // 2` | `2` | `//` 是注释开头，不是运算符 |
| `-5 // 2` | `-3`，向负无穷取整 | 不适用 |
| `-5 / 2` | `-2.5` | 整数相除得到 `-2`，向 0 截断 |

C++ 要保留小数，至少让一个操作数成为浮点数。

```cpp
double a = 5.0 / 2;
double b = static_cast<double>(5) / 2;
```

## 乘方与异或

```python
2 ** 3  # 8，乘方
2 ^ 3   # 1，按位异或
```

```cpp
std::pow(2, 3);  // 乘方，需要 <cmath>
2 ^ 3;           // 1，按位异或
```

C++ 的 `std::pow` 返回类型取决于参数和重载；处理整数幂且要求精确整数结果时，循环乘法通常更合适。

## 自增

Python 没有 `++` 和 `--`。

```python
x += 1
```

```cpp
++x;
x++;
x += 1;
```

# 字符串与字符

Python 没有独立的字符类型，单个字符仍是 `str`。

```python
ch = "A"          # str
text = "hello"   # str
last = text[-1]   # "o"
part = text[1:4]  # "ell"
```

C++ 的单引号表示字符，双引号表示字符串字面量。

```cpp
char ch = 'A';
std::string text = "hello";
char last = text.back();
std::string part = text.substr(1, 3);  // 从下标 1 开始，取 3 个字符
```

关键区别：

- Python 字符串不可变；C++ 的 `std::string` 可以修改。
- Python 支持负下标；C++ 不支持，`text[-1]` 会发生越界访问。
- Python 切片的第二个值是结束位置；C++ `substr` 的第二个参数是长度。
- Python 用 `str(123)` 转字符串；C++ 常用 `std::to_string(123)`。
- Python 用 `int(text)` 转整数；C++ 常用 `std::stoi(text)`。

# 容器

| Python              | C++                                | 主要区别          |
| ------------------- | ---------------------------------- | ------------- |
| `list`              | `std::vector<T>`                   | C++ 元素类型固定    |
| `tuple`             | `std::tuple<...>`、`std::pair<...>` | C++ 需要写明各元素类型 |
| `dict`              | `std::unordered_map<K, V>`         | C++ 键和值的类型固定  |
| `set`               | `std::unordered_set<T>`            | C++ 元素类型固定    |
| `collections.deque` | `std::deque<T>`                    | 都支持两端操作       |

## `pop` 与 `pop_back`

```python
numbers = [1, 2, 3]
last = numbers.pop()  # 删除并返回 3
```

```cpp
std::vector<int> numbers{1, 2, 3};
int last = numbers.back();
numbers.pop_back();   // 只删除，不返回元素
```

## 查找键

```python
if key in scores:
    value = scores[key]
```

```cpp
if (auto it = scores.find(key); it != scores.end()) {
    int value = it->second;
}
```

C++20 可以用 `scores.contains(key)` 判断键是否存在。

`map[key]` 和 `unordered_map[key]` 在键不存在时会插入默认值。只想检查或读取时，使用 `find`、`at` 或 C++20 的 `contains`。

# 循环

## 遍历元素

```python
for x in numbers:
    print(x)
```

```cpp
for (const auto& x : numbers) {
    std::cout << x << '\n';
}
```

C++ 中需要修改原元素时使用非常量引用：

```cpp
for (auto& x : numbers) {
    x *= 2;
}
```

## 遍历下标

```python
for i, x in enumerate(numbers):
    print(i, x)
```

```cpp
for (std::size_t i = 0; i < numbers.size(); ++i) {
    std::cout << i << ' ' << numbers[i] << '\n';
}
```

`range` 的结束位置不包含在结果中：

```python
for i in range(1, 5):  # 1、2、3、4
    ...
```

```cpp
for (int i = 1; i < 5; ++i) {
    // ...
}
```

两门语言都有 `break` 和 `continue`。Python 还支持循环 `else`：只有循环没有被 `break` 打断时，`else` 才会执行；C++ 没有对应语法。

# 赋值、复制与引用

Python 赋值通常只是让另一个名字绑定同一个对象。

```python
a = [1, 2]
b = a
b.append(3)
print(a)  # [1, 2, 3]

c = a.copy()  # 浅拷贝
```

C++ 的普通值对象赋值通常会复制内容。

```cpp
std::vector<int> a{1, 2};
std::vector<int> b = a;
b.push_back(3);          // a 不变

auto& c = a;             // c 引用 a
```

这是两门语言最关键的思维差异之一：

- Python 的 `b = a` 不会自动复制对象。
- C++ 的 `b = a` 对普通值对象通常会创建副本。
- C++ 用 `&` 声明引用；Python 没有对应的变量声明语法。

# 函数与参数

```python
def add(a: int, b: int = 1) -> int:
    return a + b
```

```cpp
int add(int a, int b = 1) {
    return a + b;
}
```

Python 类型注解主要供阅读、IDE 和类型检查器使用，默认不会在运行时强制检查。C++ 参数和返回类型会参与编译期检查。

C++ 明确区分传值和引用：

```cpp
void read(int x);                     // 复制 x
void change(int& x);                  // 可以修改调用者的 x
void print(const std::string& text);  // 不复制，也不允许修改 text
```

Python 传递对象引用的值。修改可变对象本身会影响调用者，重新绑定局部变量不会。

```python
def change(items):
    items.append(1)  # 调用者可见

def rebind(items):
    items = [1]      # 只改变局部变量
```

Python 不要使用可变对象作为默认参数：

```python
def append_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

# 输入与输出

Python 的 `input()` 总是返回字符串。

```python
age = int(input())
a, b = map(int, input().split())
```

C++ 的流输入会按照变量类型解析。

```cpp
int age;
std::cin >> age;

int a, b;
std::cin >> a >> b;
```

读取整行：

```python
line = input()
```

```cpp
std::string line;
std::getline(std::cin, line);
```

在 C++ 中混用 `>>` 和 `getline` 时，`>>` 留下的换行符可能导致下一次 `getline` 读到空行。常见处理方式是：

```cpp
#include <limits>

std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
std::getline(std::cin, line);
```

# 类与对象

```python
class Person:
    def __init__(self, name):
        self.name = name

    def greet(self):
        return f"Hi, {self.name}"
```

```cpp
class Person {
public:
    explicit Person(const std::string& name) : name_(name) {}

    std::string greet() const {
        return "Hi, " + name_;
    }

private:
    std::string name_;
};
```

- Python 的 `self` 不是关键字，但实例方法通常必须显式写出它。
- C++ 的 `this` 是隐式提供的指针，访问成员时通常可以省略。
- Python 主要用命名约定表示内部成员；C++ 使用 `public`、`protected` 和 `private`。
- Python 在 `__init__` 中初始化实例属性；C++ 优先使用构造函数的成员初始化列表。

# 排序与常用操作

```python
b = sorted(a)            # 返回新列表，a 不变
a.sort()                 # 原地排序，返回 None
a.sort(reverse=True)     # 原地降序
```

```cpp
std::vector<int> b = a;
std::sort(b.begin(), b.end());                   // 原地升序
std::sort(a.begin(), a.end(), std::greater<>()); // 原地降序
```

| 操作 | Python | C++ |
| --- | --- | --- |
| 清空 | `a.clear()` | `a.clear()` |
| 反转 | `a.reverse()` | `std::reverse(a.begin(), a.end())` |
| 字符串查找失败 | `s.find(x) == -1` | `s.find(x) == std::string::npos` |
| 取最后一个元素 | `a[-1]` | `a.back()` |

# 模块与程序入口

Python 用 `import` 在运行时加载模块。首次导入通常会执行模块的顶层代码。

```python
import math

def main():
    print(math.sqrt(4))

if __name__ == "__main__":
    main()
```

C++ 的 `#include` 在预处理阶段引入声明等内容，程序从 `main` 函数开始。

```cpp
#include <cmath>
#include <iostream>

int main() {
    std::cout << std::sqrt(4.0) << '\n';
    return 0;
}
```

`using namespace std;` 不是 C++ 版的 `import`。它只是把命名空间中的名称引入当前作用域，头文件和大型项目中应避免全局使用。

# 高频纠错

| 容易写错 | 正确写法或说明 |
| --- | --- |
| Python 写 `true`、`false` | `True`、`False` |
| C++ 写 `True`、`False` | `true`、`false` |
| Python 写 `&&`、`\|\|`、`!` | `and`、`or`、`not` |
| Python 写 `else if` | `elif` |
| C++ 写 `elif` | `else if` |
| Python 写 `x++` | `x += 1` |
| 把 `^` 当乘方 | Python 用 `**`；C++ 用 `std::pow` 或自行计算 |
| 认为 C++ 的 `5 / 2` 是 `2.5` | 两个整数相除得到 `2` |
| C++ 写 `0 < x < 10` | 写成 `0 < x && x < 10` |
| Python 用 `is` 比较字符串内容 | 使用 `==` |
| C++ 用 `if (v)` 判断 `vector` 非空 | 使用 `if (!v.empty())` |
| 认为 C++ `pop_back()` 会返回元素 | 先调用 `back()`，再调用 `pop_back()` |
| C++ 使用 `a[-1]` 取最后一个元素 | 使用 `a.back()` |
| 把 Python 类型注解当成强制类型声明 | 注解默认不进行运行时强制检查 |
| 把 C++ `auto` 当成动态类型 | `auto` 只在编译期推导一次类型 |
| 认为两门语言中的 `b = a` 都会复制对象 | Python 通常共享对象；C++ 值对象通常复制 |
