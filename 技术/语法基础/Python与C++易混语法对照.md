---
tags:
  - Python
  - CPP
  - 编程
  - 语法对照
date: 2026-08-23
---

> 这不是两门语言的完整教程，而是一份“从 Python 切到 C++，或从 C++ 切回 Python”时的防混淆清单。基础内容可配合 [[技术/语法基础/Python入门]]、[[Python 工程实践]] 和 [[C++语法]] 阅读。

# 1. 一眼速查

| 目的    | Python                    | C++                                            |
| ----- | ------------------------- | ---------------------------------------------- |
| 输出    | `print(x)`                | `std::cout << x << '\n';`                      |
| 输入整数  | `x = int(input())`        | `int x; std::cin >> x;`                        |
| 布尔值   | `True`、`False`            | `true`、`false`                                 |
| 空值    | `None`                    | `nullptr`（指针）                                  |
| 逻辑运算  | `and`、`or`、`not`          | `&&`、`\|\|`、`!`                                |
| 条件分支  | `if / elif / else`        | `if / else if / else`                          |
| 定义函数  | `def add(a, b):`          | `int add(int a, int b) {}`                     |
| 动态数组  | `list`                    | `std::vector`                                  |
| 哈希映射  | `dict`                    | `std::unordered_map`                           |
| 长度    | `len(a)`                  | `a.size()`                                     |
| 追加元素  | `a.append(x)`             | `a.push_back(x)`                               |
| 删除末尾  | `a.pop()`                 | `a.pop_back()`（无返回值）                           |
| 遍历元素  | `for x in a:`             | `for (const auto& x : a)`                      |
| 遍历下标  | `for i in range(len(a)):` | `for (std::size_t i = 0; i < a.size(); ++i)`   |
| 成员判断  | `x in a`                  | `a.contains(x)`（部分 C++20 容器）或 `std::find(...)` |
| 导入/引入 | `import math`             | `#include <cmath>`                             |
| 当前对象  | `self`（显式形参）              | `this`（隐式指针）                                   |
| 继承    | `class Dog(Animal):`      | `class Dog : public Animal {}`                 |

# 2. 代码结构：缩进、花括号和分号

## Python

Python 用**缩进**划分代码块，通常使用 4 个空格；语句末尾一般不写分号。

```python
if score >= 60:
    print("pass")
else:
    print("fail")
```

## C++

C++ 用 `{}` 划分代码块，多数语句以 `;` 结尾。缩进主要用于可读性。

```cpp
if (score >= 60) {
    std::cout << "pass\n";
} else {
    std::cout << "fail\n";
}
```

> **最容易顺手写错**
>
> - Python 的条件后有冒号 `:`，C++ 没有。
> - C++ 的语句常以 `;` 结尾，Python 通常不写。
> - 不要在 C++ 的 `if (...)` 后误加分号：`if (x > 0);` 会产生一个空语句。

# 3. 变量与类型

## Python：动态类型，名字绑定对象

```python
x = 10
x = "hello"  # 同一个名字可以重新绑定到另一种类型的对象
```

## C++：静态类型，变量类型通常在编译期确定

```cpp
int x = 10;
// x = "hello";  // 类型不匹配

auto y = 3.14;   // 推导为 double，之后 y 的类型仍然固定
```

> **`auto` 不等于 Python 的动态类型**
>
> C++ 的 `auto` 只是让编译器在**编译期推导一次类型**，不是让变量以后随意改变类型。

## 未初始化变量

```python
# print(x)  # 名字不存在，NameError
```

```cpp
int x;              // 某些局部基本类型不会被自动初始化
// std::cout << x;  // 读取未初始化值会导致未定义行为
```

因此 C++ 中应主动初始化：

```cpp
int x{};                 // 0
std::vector<int> nums{}; // 空 vector
```

# 4. 布尔、逻辑与真假判断

```python
if x > 0 and x < 10:
    print(not finished)
```

```cpp
if (x > 0 && x < 10) {
    std::cout << !finished;
}
```

易混点：

- Python 是 `True` / `False`，首字母大写；C++ 是 `true` / `false`，全小写。
- Python 常写 `if items:` 判断容器非空。
- C++ 容器通常不能直接当布尔值，应写 `if (!items.empty())`。
- 两者的 `and`/`&&`、`or`/`||` 都会短路求值。

# 5. 条件分支与比较

## `elif` 与 `else if`

```python
if x < 0:
    result = "negative"
elif x == 0:
    result = "zero"
else:
    result = "positive"
```

```cpp
if (x < 0) {
    result = "negative";
} else if (x == 0) {
    result = "zero";
} else {
    result = "positive";
}
```

## 连续比较

```python
if 0 < x < 10:  # 正确，符合数学直觉
    ...
```

```cpp
if (0 < x && x < 10) {  // 正确写法
}

// 0 < x < 10 在 C++ 中不会按数学含义判断，应避免！
```

## 赋值和相等

- 两门语言都用 `==` 比较相等。
- C++ 可在条件中写赋值表达式，例如 `if (x = 1)`，这通常是误写，应留意编译器警告。
- Python 普通赋值 `=` 不能直接放进条件；赋值表达式使用单独的 `:=`（海象运算符）。

# 6. 数值运算：除法是重灾区

| 表达式 | Python | C++ |
| --- | --- | --- |
| `5 / 2` | `2.5` | 若两边是 `int`，结果为 `2` |
| `5 // 2` | `2` | 没有 `//` 整除运算符，`//` 是注释开头 |
| `-5 // 2` | `-3`（向负无穷取整） | 不适用 |
| `-5 / 2`（整数） | `-2.5` | `-2`（向 0 截断） |
| `2 ** 3` | `8` | 不支持，应使用 `std::pow(2, 3)` |
| `2 ^ 3` | 按位异或，结果 `1` | 按位异或，结果 `1` |

C++ 想保留小数，至少一个操作数必须是浮点数：

```cpp
double result = 5.0 / 2;                 // 2.5
double result2 = static_cast<double>(5) / 2;
```

> **`^` 不是乘方**
>
> Python 和 C++ 中的 `^` 都是**按位异或**。Python 乘方用 `**`；C++ 通常用 `<cmath>` 中的 `std::pow`。

## 自增与复合赋值

```python
x += 1       # Python 没有 x++ 和 ++x
```

```cpp
++x;         // 常用
x++;
x += 1;
```

# 7. 字符串与字符

## Python

Python 没有独立的 `char` 类型，单个字符仍然是长度为 1 的字符串。

```python
ch = 'A'          # str
text = "hello"   # str
print(f"x = {x}")
```

## C++

单引号通常表示 `char`，双引号表示字符串字面量。

```cpp
char ch = 'A';
std::string text = "hello";
std::cout << "x = " << x << '\n';
```

其他差异：

- Python 字符串不可变；C++ 的 `std::string` 通常可以原地修改。
- Python 支持负下标：`s[-1]`；C++ 不支持，越界访问可能导致未定义行为。
- Python 原生支持切片：`s[1:4]`；C++ 可用 `s.substr(1, 3)`，第二个参数是**长度**。
- Python 的 `str(123)` 对应 C++ 的 `std::to_string(123)`。
- C++ 字符串转整数常用 `std::stoi(s)`；Python 用 `int(s)`。

# 8. 输入与输出

## Python 输入永远先得到字符串

```python
age = int(input("age: "))
a, b = map(int, input().split())
print(a, b, sep=", ", end="\n")
```

## C++ 流输入会按变量类型解析

```cpp
int age;
std::cin >> age;

int a, b;
std::cin >> a >> b;
std::cout << a << ", " << b << '\n';
```

读取整行：

```python
line = input()
```

```cpp
std::string line;
std::getline(std::cin, line);
```

> **C++ 的 `>>` 与 `getline` 混用**
>
> `std::cin >> x` 可能把换行符留在输入缓冲区，紧接着的 `getline` 会读到空行。常见处理方式是先调用 `std::cin.ignore(...)`。

# 9. 常用容器对照

| Python | C++ | 说明 |
| --- | --- | --- |
| `list` | `std::vector<T>` | 动态顺序容器 |
| `tuple` | `std::tuple<...>` / `std::pair<...>` | 固定组合 |
| `dict` | `std::unordered_map<K,V>` / `std::map<K,V>` | 哈希映射 / 有序映射 |
| `set` | `std::unordered_set<T>` / `std::set<T>` | 哈希集合 / 有序集合 |
| `collections.deque` | `std::deque<T>` | 双端队列 |

## 列表与 `vector`

```python
nums = [1, 2, 3]
nums.append(4)
last = nums.pop()       # 删除并返回 4
```

```cpp
std::vector<int> nums{1, 2, 3};
nums.push_back(4);
int last = nums.back();
nums.pop_back();        // 只删除，不返回元素
```

## 字典与映射

```python
scores = {"Alice": 90}
scores["Bob"] = 85
if "Alice" in scores:
    print(scores["Alice"])
```

```cpp
std::unordered_map<std::string, int> scores{{"Alice", 90}};
scores["Bob"] = 85;
if (scores.contains("Alice")) {          // C++20
    std::cout << scores.at("Alice");
}
```

> **C++ `map[key]` 的副作用**
>
> 当 `key` 不存在时，`map[key]` / `unordered_map[key]` 会插入一个默认值。只想读取或检查时，优先使用 `at`、`find` 或 C++20 的 `contains`。

# 10. 循环：`range` 不等于 C++ 范围 `for`

## 按元素遍历

```python
for x in nums:
    print(x)
```

```cpp
for (const auto& x : nums) {
    std::cout << x << '\n';
}
```

若要修改 C++ 容器中的原元素，使用引用：

```cpp
for (auto& x : nums) {
    x *= 2;
}
```

## 按下标遍历

```python
for i in range(len(nums)):  # 0 到 len(nums)-1
    print(i, nums[i])

for i, x in enumerate(nums):
    print(i, x)
```

```cpp
for (std::size_t i = 0; i < nums.size(); ++i) {
    std::cout << i << ' ' << nums[i] << '\n';
}
```

## `range` 的右端点不包含

```python
range(1, 5)  # 1, 2, 3, 4
```

对应 C++：

```cpp
for (int i = 1; i < 5; ++i) {
}
```

## `break`、`continue` 与循环的 `else`

两门语言都有 `break` 和 `continue`，但 Python 还支持循环 `else`：循环未被 `break` 打断时执行 `else`。C++ 没有对应语法。

# 11. 函数、参数与返回值

## 基本写法

```python
def add(a: int, b: int = 1) -> int:
    return a + b
```

```cpp
int add(int a, int b = 1) {
    return a + b;
}
```

Python 类型注解默认主要供阅读、IDE 和类型检查器使用，通常不会自动进行运行时强制检查；C++ 参数与返回类型会参与编译期类型检查。

## 传值、引用与对象修改

C++ 明确区分传值、引用和常量引用：

```cpp
void f(int x);                    // 复制值，修改不影响调用者
void g(int& x);                   // 引用，可修改调用者
void h(const std::string& text);  // 避免复制，且不允许修改
```

Python 传递的是**对象引用的值**：

```python
def change(items):
    items.append(1)  # 修改同一个可变对象，调用者可见

def rebind(items):
    items = [1]      # 只让局部名字绑定新对象，不替换调用者的变量
```

## 多返回值

```python
def point():
    return 3, 4

x, y = point()
```

```cpp
std::pair<int, int> point() {
    return {3, 4};
}

auto [x, y] = point();  // C++17 结构化绑定
```

## 默认参数的坑

Python 不要把可变对象直接作为默认值：

```python
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

C++ 默认参数通常放在函数声明中，并且默认参数之后的参数也必须有默认值。

# 12. 赋值、复制与引用语义

这是两门语言思维差异最大的地方之一。

## Python：赋值通常只是让名字绑定同一个对象

```python
a = [1, 2]
b = a
b.append(3)
print(a)  # [1, 2, 3]

c = a.copy()  # 浅拷贝
```

## C++：普通赋值通常复制值

```cpp
std::vector<int> a{1, 2};
std::vector<int> b = a;  // 独立副本
b.push_back(3);          // a 不变

auto& c = a;             // c 是 a 的引用，修改 c 会影响 a
```

记忆方式：

- Python：`b = a` 多数时候像“再贴一个标签”。
- C++：`b = a` 多数时候像“复制一份内容”；写 `&` 才明确表示引用。

# 13. `==`、`is`、地址与空值

## Python

```python
a == b      # 比较值是否相等
a is b      # 比较是否为同一个对象
x is None   # 判断 None 的推荐写法
```

## C++

```cpp
a == b          // 基本类型比较值；类可重载 operator==
ptr == nullptr  // 判断空指针
&a == &b        // 比较两个对象的地址
```

> **不要把 `is` 当成更快的 `==`**
>
> Python 中判断数字、字符串等值是否相等应使用 `==`；`is` 主要用于 `None`、单例或明确需要判断对象身份的场景。

# 14. 作用域

Python 的 `if`、`for`、`while` 通常**不会创建新的局部作用域**；函数、类和模块会形成重要作用域。

```python
if True:
    x = 10
print(x)  # 10
```

C++ 的花括号块会创建块级作用域：

```cpp
if (true) {
    int x = 10;
}
// std::cout << x;  // x 已离开作用域
```

# 15. 类：`self` 与 `this`

## Python

```python
class Person:
    def __init__(self, name):
        self.name = name

    def greet(self):
        return f"Hi, {self.name}"
```

## C++

```cpp
class Person {
public:
    explicit Person(std::string name) : name_(std::move(name)) {}

    std::string greet() const {
        return "Hi, " + this->name_;
    }

private:
    std::string name_;
};
```

核心区别：

- Python 的 `self` 不是关键字，但按惯例必须显式写在实例方法的第一个形参位置。
- C++ 的 `this` 是指向当前对象的隐式指针，成员函数形参中不用写它。
- Python 常靠命名约定表达“内部成员”；C++ 有 `public`、`protected`、`private` 访问控制。
- Python 构造初始化常写在 `__init__`；C++ 构造函数推荐使用成员初始化列表。

# 16. 异常处理

```python
try:
    value = int(text)
except ValueError as e:
    print(e)
finally:
    cleanup()
```

```cpp
try {
    int value = std::stoi(text);
} catch (const std::invalid_argument& e) {
    std::cerr << e.what();
}
// C++ 没有 finally，资源清理通常依靠 RAII
```

Python 用 `raise` 抛出异常；C++ 用 `throw`。

# 17. `import` 与 `#include`

```python
import math
print(math.sqrt(4))
```

```cpp
#include <cmath>
std::cout << std::sqrt(4.0);
```

- Python `import` 在运行时查找、加载模块，首次导入通常会执行模块顶层代码。
- C++ `#include` 是预处理指令，主要把头文件内容提供给当前编译单元。
- Python 用 `module.name` 管理名称；C++ 标准库名称通常位于 `std::` 命名空间。
- `using namespace std;` 与 Python 的 `import` 不是一回事，不建议在大型项目或头文件中全局使用。

# 18. 排序与常用方法名

## 排序

```python
b = sorted(a)  # 返回新列表，a 不变
a.sort()       # 原地排序，返回 None
```

```cpp
std::vector<int> b = a;
std::sort(b.begin(), b.end());  // 原地排序，需 <algorithm>
```

降序：

```python
a.sort(reverse=True)
```

```cpp
std::sort(a.begin(), a.end(), std::greater<>());
```

## 高频方法名错位

| 操作 | Python | C++ |
| --- | --- | --- |
| 长度 | `len(a)` | `a.size()` |
| 是否为空 | `not a` | `a.empty()` |
| 列表末尾添加 | `a.append(x)` | `a.push_back(x)` |
| 字符串查找 | `s.find(x)`，未找到为 `-1` | `s.find(x)`，未找到为 `std::string::npos` |
| 清空 | `a.clear()` | `a.clear()` |
| 反转 | `a.reverse()` | `std::reverse(a.begin(), a.end())` |

# 19. Python 独有或写法明显不同的功能

## 切片

```python
a[start:stop:step]
a[::-1]  # 反转副本
```

C++ 标准容器没有统一的内置切片语法，通常使用迭代器、范围库或自行复制。

## 列表推导式

```python
squares = [x * x for x in nums if x > 0]
```

C++ 通常使用循环，或 `<algorithm>` / ranges：

```cpp
std::vector<int> squares;
for (int x : nums) {
    if (x > 0) squares.push_back(x * x);
}
```

## `match` 与 `switch`

Python 3.10+ 的 `match` 是结构化模式匹配，不只是 C++ `switch` 的同义替换。C++ `switch` 主要针对整数、枚举等离散值。

# 20. 入口与主程序

Python 常见入口保护：

```python
def main():
    print("start")

if __name__ == "__main__":
    main()
```

C++ 程序从 `main` 函数开始：

```cpp
int main() {
    std::cout << "start\n";
    return 0;
}
```

# 21. 高频“肌肉记忆”纠错表

| 错误写法或错误想法 | 应改为 |
| --- | --- |
| Python 写 `true` / `false` | `True` / `False` |
| C++ 写 `True` / `False` | `true` / `false` |
| Python 写 `&&`、`\|\|`、`!` | `and`、`or`、`not` |
| C++ 写 `and`、`or` 时感到陌生 | 虽然 C++ 有替代记号，但工程中更常见 `&&`、`\|\|` |
| Python 写 `else if` | `elif` |
| C++ 写 `elif` | `else if` |
| Python 写 `x++` | `x += 1` |
| 把 `^` 当乘方 | Python 用 `**`；C++ 用 `std::pow` |
| 认为 C++ `5 / 2 == 2.5` | 两个整数相除得到 `2`，先转浮点数 |
| 认为 Python `//` 永远等于向 0 截断 | 对负数是向负无穷取整 |
| C++ 写 `0 < x < 10` | `0 < x && x < 10` |
| Python 用 `is` 比较字符串内容 | 用 `==` |
| Python 用 `if list.empty()` | `if not list:` |
| C++ 用 `if (v)` 判断 `vector` 非空 | `if (!v.empty())` |
| 认为 C++ `pop_back()` 会返回删除值 | 先 `back()` 取值，再 `pop_back()` |
| 认为 `b = a` 在两门语言里复制效果相同 | Python 多为共享对象；C++ 普通值对象多为复制 |
| C++ 使用负下标取最后元素 | 使用 `a.back()`，不要写 `a[-1]` |
| 把 Python 类型注解当成强制类型声明 | 注解默认不自动强制运行时类型 |
| 把 C++ `auto` 当成动态类型 | `auto` 只在编译期推导，类型随后固定 |

# 22. 最小对照示例：统计偶数之和

## Python

```python
def sum_even(nums: list[int]) -> int:
    total = 0
    for x in nums:
        if x % 2 == 0:
            total += x
    return total

nums = [1, 2, 3, 4]
print(sum_even(nums))  # 6
```

## C++

```cpp
#include <iostream>
#include <vector>

int sum_even(const std::vector<int>& nums) {
    int total = 0;
    for (int x : nums) {
        if (x % 2 == 0) {
            total += x;
        }
    }
    return total;
}

int main() {
    std::vector<int> nums{1, 2, 3, 4};
    std::cout << sum_even(nums) << '\n';  // 6
    return 0;
}
```

# 23. 核心记忆框架

1. **代码块**：Python 看缩进，C++ 看花括号。
2. **类型**：Python 名字动态绑定对象；C++ 变量类型编译期确定。
3. **复制**：Python 赋值常共享对象；C++ 值赋值通常复制对象。
4. **容器**：Python 偏内置统一语法；C++ 偏模板类型、迭代器和成员函数。
5. **除法**：Python `/` 总是普通除法；C++ 是否保留小数取决于操作数类型。
6. **对象**：Python 显式写 `self`；C++ 隐式拥有 `this`。
7. **工程过程**：Python 通常解释运行；C++ 通常先编译、链接，再运行。
