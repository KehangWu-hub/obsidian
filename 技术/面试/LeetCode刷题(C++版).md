---
tags:
  - 刷题
  - 算法
date: 2026-09-15
---

==此篇为 Hot100 刷题笔记（C++ 版）==

# 底层认知

## 1. 算法评判基础：复杂度分析

- **时间复杂度（Time Complexity）**：衡量算法运行时间随数据规模 $n$ 增大的增长趋势。
- **空间复杂度（Space Complexity）**：衡量算法所需额外内存随数据规模 $n$ 增大的增长趋势。
- **大 O 符号（Big O Notation）**：忽略常数项和低阶项，只保留最高阶项。
  - 例如：操作次数为 $\frac{n(n-1)}{2}=\frac{1}{2}n^2-\frac{1}{2}n$，时间复杂度记为 $O(n^2)$。
- **常见复杂度从快到慢**：$O(1)$、$O(\log n)$、$O(n)$、$O(n\log n)$、$O(n^2)$。

刷题时常通过增加少量额外空间来降低时间复杂度。例如，用哈希表保存已经遍历过的数据，可以把查找从 $O(n)$ 降为平均 $O(1)$。

复杂度分析关注的是整体增长趋势，不是某一行代码看起来执行了几层。即使代码中有嵌套循环，只要内层循环在整个算法中总共只处理 $O(n)$ 个元素，总时间复杂度仍可能是 $O(n)$。

## 2. LeetCode 平台机制（OJ 原理）

LeetCode 会自动创建 `Solution` 对象，并使用多组测试数据调用题目指定的成员函数。因此，通常只需要提交 `class Solution`，不需要编写 `main()`、输入输出或测试代码。

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        // 只实现题目要求的函数
    }
};
```

需要注意以下几点：

- 函数名、参数类型和返回类型必须与题目给出的模板一致。
- 成员函数通常放在 `public:` 下，否则评测程序无法调用。
- LeetCode 通常已经提供常用头文件和 `using namespace std;`；在本地编译时仍要自己包含 `<vector>`、`<string>`、`<unordered_map>` 等头文件。
- 不要把调试输出当作答案的一部分。最终结果应通过 `return` 返回。

## 3. C++ 刷题中常见的传参方式

```cpp
void f1(vector<int> nums);        // 值传递：复制一份数组
void f2(vector<int>& nums);       // 引用传递：不复制，可以修改原数组
void f3(const vector<int>& nums); // 常量引用：不复制，也不允许修改
```

数组或字符串可能很大，不需要复制时优先使用引用。只读取参数时，优先使用 `const T&`；题目已经给定的函数签名不要自行修改。

范围 `for` 循环也有相同区别：

```cpp
for (int x : nums) { }         // x 是元素副本
for (int& x : nums) { }        // x 是原元素的引用，可以修改
for (const int& x : nums) { }  // 不复制，不修改
```

对于 `int`、`char` 等小类型，直接复制通常更简单；对于 `string`、`vector` 等对象，常用 `const auto&` 避免复制。

# 哈希表（Hash Table）

## 1. 概念

C++ 刷题中常用两种哈希容器：

| 容器                          | 保存内容   | 常见用途            |
| :-------------------------- | :----- | :-------------- |
| `unordered_map<Key, Value>` | 键和值的映射 | 记录索引、次数、分组等附加信息 |
| `unordered_set<Key>`        | 不重复的键  | 去重、快速判断元素是否存在   |

哈希容器通过哈希函数把键映射到桶中，查找、插入和删除的**平均**时间复杂度为 $O(1)$。不同的键可能落入同一个桶，这称为哈希冲突；冲突严重时，单次操作最坏可能退化为 $O(n)$。

`unordered_map` 不保证遍历顺序。若题目需要键有序，或需要快速寻找前驱、后继，应考虑基于平衡树的 `map` 和 `set`，其操作复杂度通常为 $O(\log n)$。

### 哪些类型可以作为键

键类型必须能够比较是否相等，并且存在对应的哈希函数。标准库已经支持常见基础类型：

- `int`、`long long`、`char`、`string` 等可以直接作为键。
- `vector<int>` 不能直接作为 `unordered_map` 的键。
- `pair<int, int>`、`array<int, 26>` 等类型在常见刷题环境中通常也不能直接作为键，需要自定义哈希函数。

如果只是要表示一个单词的字符组成，常见做法是先排序，得到 `string` 键；这样无需编写自定义哈希。

## 2. 常用方法

先区分两种容器：需要保存“键和值”时使用 `unordered_map`，只需要去重或判断元素是否存在时使用 `unordered_set`。普通的 `map` 和 `set` 也能完成这些操作，但它们的查找、插入和删除复杂度为 $O(\log n)$；`unordered_map` 和 `unordered_set` 的平均复杂度为 $O(1)$，通常更快。因此，不要求元素有序时，刷题中一般优先使用 `unordered_map` 和 `unordered_set`。

### 创建哈希表

`unordered_map<键类型, 值类型>` 用来建立键和值之间的映射：

```cpp
unordered_map<int, int> index;             // 数字 -> 下标
unordered_map<string, int> count;          // 字符串 -> 出现次数
unordered_map<string, vector<string>> groups; // 特征 -> 一组字符串
```

`unordered_set<元素类型>` 只保存不重复的元素：

```cpp
unordered_set<int> seen;

// 使用 nums 中的所有元素初始化，并自动去重
unordered_set<int> numbers(nums.begin(), nums.end());
```

### 插入或修改元素

#### `unordered_map` 的 `[]`

```cpp
unordered_map<string, int> count;

count["apple"] = 2; // 插入或修改
count["apple"]++;   // 将值从 2 改为 3
```

如果键不存在，`mp[key]` 会先创建这个键，并给它一个默认值：

| 值类型 | 默认值 |
| :--- | :--- |
| `int` | `0` |
| `string` | 空字符串 |
| `vector<T>` | 空数组 |

因此，可以直接用 `[]` 完成计数和分组：

```cpp
count[word]++;
groups[key].push_back(word);
```

C++ 不需要使用 Python 的 `defaultdict`，因为 `[]` 已经能在键不存在时创建默认值。

但要注意，`[]` 不只是查询，它可能会修改哈希表：

```cpp
unordered_map<string, int> mp;
int value = mp["missing"];

// 此时 mp 中已经多了 {"missing", 0}
```

所以，**插入、修改、计数或分组时使用 `[]`；只想判断键是否存在时使用 `find()`**。

#### `unordered_set` 的 `insert()`

```cpp
unordered_set<int> seen;

seen.insert(3);
seen.insert(5);
seen.insert(3); // 3 已经存在，不会重复保存
```

### 使用 `find()` 查找元素

`find(key)` 用于查找键，并返回一个迭代器。迭代器可以理解为指向容器中某个元素的位置。

```cpp
unordered_map<int, int> mp;
mp[3] = 7;

auto it = mp.find(3);
```

这里的 `auto` 表示让编译器根据右侧结果自动推断类型。上面的 `it` 实际类型是：

```cpp
unordered_map<int, int>::iterator
```

由于迭代器类型较长，通常直接使用 `auto`。

`find()` 有两种结果：

- 找到键：返回指向该元素的迭代器。
- 没找到键：返回 `mp.end()`。

因此，完整的判断方式是：

```cpp
auto it = mp.find(3);

if (it != mp.end()) {
    // 找到了
} else {
    // 没找到
}
```

对于 `unordered_map`，迭代器指向的是一组键值对：

```cpp
if (it != mp.end()) {
    int key = it->first;    // 键：3
    int value = it->second; // 值：7
}
```

已经通过 `find()` 得到迭代器后，直接使用 `it->second` 读取值，不需要再写一次 `mp[key]`。

`unordered_set` 的查找方式相同：

```cpp
unordered_set<int> seen = {1, 3, 5};

if (seen.find(3) != seen.end()) {
    // 3 存在
}
```

> `end()` 不指向实际元素，它只表示容器末尾之后的位置。在这里主要作为“没有找到”的标记。

### 使用 `at()` 读取值

确定键存在时，可以使用 `at()` 读取值：

```cpp
int value = mp.at(3);
```

`at()` 与 `[]` 的区别是：

- `mp[key]`：键不存在时自动插入默认值。
- `mp.at(key)`：键不存在时抛出异常，不会插入新元素。

刷题时通常使用 `find()` 完成“查找并读取”，使用 `[]` 完成“插入或修改”，因此 `at()` 用得相对较少。

### 删除元素

`erase(key)` 用于根据键删除元素：

```cpp
mp.erase(3);
seen.erase(5);
```

如果元素不存在，`erase(key)` 不会报错。

`clear()` 用于删除容器中的全部元素：

```cpp
mp.clear();
seen.clear();
```

### 遍历哈希表

C++17 可以使用结构化绑定同时取出键和值：

```cpp
for (const auto& [key, value] : mp) {
    // key 是键，value 是值
}
```

其中：

- `auto`：让编译器自动推断类型。
- `&`：使用引用，避免复制键值对。
- `const`：只读取，不允许在循环中修改。

如果需要修改值，去掉 `const`：

```cpp
for (auto& [key, value] : mp) {
    value++;
}
```

`unordered_set` 中没有键值对，直接遍历元素：

```cpp
for (int num : seen) {
    // num 是集合中的元素
}
```

哈希容器不保证遍历顺序，不能依赖输出元素的先后顺序。

### 常见使用方式

#### 统计出现次数

```cpp
unordered_map<int, int> count;

for (int num : nums) {
    count[num]++;
}
```

#### 按特征分组

```cpp
unordered_map<string, vector<string>> groups;

for (const string& word : words) {
    string key = word;
    sort(key.begin(), key.end());
    groups[key].push_back(word);
}
```

`sort()` 会直接修改字符串，所以先把 `word` 复制给 `key`，再对 `key` 排序。字母异位词排序后得到相同的键，因此会进入同一组。

#### 去重并快速判断是否存在

```cpp
unordered_set<int> numbers(nums.begin(), nums.end());

if (numbers.find(target) != numbers.end()) {
    // target 存在
}
```

### `reserve()`：可选的性能优化

如果大致知道要存多少个元素，可以提前预留空间，减少扩容次数：

```cpp
unordered_map<int, int> mp;
mp.reserve(nums.size());
```

`reserve()` 不改变算法逻辑，也不是必须使用。刚开始刷题时，可以先不写。

## 3. 题目

### 题目 1：两数之和（LeetCode 1，简单）

==原题==

给定一个整数数组 `nums` 和一个整数目标值 `target`，找出和为 `target` 的两个整数，并返回它们的数组下标。

可以假设每种输入只对应一个答案，并且同一个元素不能重复使用。答案可以按任意顺序返回。

==我的答案==

```cpp
#include <unordered_map>

class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> hashtable;
        int key = 0;

        for (vector<int>::iterator it = nums.begin(); it != nums.end(); it++) {
            hashtable[key] = *it;
            key++;
        }

        vector<int> answer;
        answer.resize(2);

        for (unordered_map<int, int>::iterator it = hashtable.begin();
             it != hashtable.end(); it++) {
            unordered_map<int, int>::iterator a =
                hashtable.find(target - it->second);

            if (a != hashtable.end()) {
                answer.push_back(it->first);
                answer.push_back(a->first);
                continue;
            }
        }

        return answer;
    }
};
```

==错误分析==

- `hashtable` 保存的是“下标 → 数值”，但 `find(target - it->second)` 会按键查找。这里实际查找的是“下标是否等于补数”，而不是“补数是否存在”。哈希表应保存“数值 → 下标”。
- `answer.resize(2)` 已经创建了两个值为 `0` 的元素，后面再调用 `push_back()` 会继续追加，返回结果的长度会超过 `2`。如果是预留空间，这里可以用reserve，或者干脆不画蛇添足。
- 找到答案后使用 `continue` 会继续遍历，可能重复加入结果。题目只有一个答案，找到后应直接 `return`。
- 当前写法没有保证两个下标不同，可能把同一个元素使用两次。

==答案（哈希表）==

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> hashtable;
        for (int i = 0; i < nums.size(); ++i) {    //注意这里是nums.size()不是sizeof(nums)
            auto it = hashtable.find(target - nums[i]);
            if (it != hashtable.end()) {
                return {it->second, i};
            }
            hashtable[nums[i]] = i;
        }
        return {};
    }
};
```

>区分nums.size()和sizeof(nums)，sizeof(nums)得到的是**vector这个对象本身占多少字节**，不是里面有多少个元素

==解析==

- **核心逻辑**：遍历当前数字时，先查找它需要的另一个数，再把当前数字和下标存入哈希表。
- **为什么先查后存**：这样不会让同一个元素与自己匹配；对于 `[3, 3]`、`target = 6`，第二个 `3` 仍能找到第一个 `3`。
- **为什么保存下标**：题目要求返回数组下标，所以使用 `unordered_map<int, int>` 保存“数值到下标”的映射。
- **时间复杂度**：平均 $O(n)$。
- **空间复杂度**：$O(n)$。

---

### 题目 2：字母异位词分组（LeetCode 49，中等）

==原题==

给定一个字符串数组，将字母异位词组合在一起，可以按任意顺序返回结果。

字母异位词由重新排列源单词中的全部字母得到。

- **示例**：输入 `strs = ["eat", "tea", "tan", "ate", "nat", "bat"]`，输出可以是 `[["bat"], ["nat", "tan"], ["ate", "eat", "tea"]]`。

==答案（排序 + 哈希分组）==

```cpp
class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string>& strs) {
        unordered_map<string, vector<string>> groups;
        groups.reserve(strs.size());

        for (const string& word : strs) {
            string key = word;
            sort(key.begin(), key.end());
            groups[key].push_back(word);
        }

        vector<vector<string>> answer;
        answer.reserve(groups.size());

        for (auto& [key, group] : groups) {
            answer.push_back(move(group));
        }

        return answer;
    }
};
```

==解析==

- **统一特征**：字母异位词排序后会得到相同字符串，例如 `"eat"` 和 `"tea"` 都得到 `"aet"`。
- **为什么复制 `word`**：`sort()` 会原地修改字符串。复制为 `key` 后排序，才能保留原单词放入答案。
- **为什么可以直接 `groups[key].push_back(word)`**：键不存在时，`groups[key]` 会自动创建一个空的 `vector<string>`。
- 设字符串数量为 $n$，单个字符串的最大长度为 $k$，时间复杂度为 $O(nk\log k)$，额外空间复杂度为 $O(nk)$。

> 如果使用 26 个字母的频次数组作为键，可以把生成特征的时间降为 $O(k)$，但需要把频次编码成字符串或为 `array<int, 26>` 自定义哈希。当前写法更直接，也足以通过本题。

---

### 题目 3：最长连续序列（LeetCode 128，中等）

==原题==

给定一个未排序的整数数组 `nums`，找出数字连续的最长序列长度。序列元素不要求在原数组中连续，算法时间复杂度应为 $O(n)$。

- **示例**：输入 `nums = [100, 4, 200, 1, 3, 2]`，输出 `4`，因为最长连续序列是 `[1, 2, 3, 4]`。

==答案（哈希集合 + 起点剪枝）==

```cpp
class Solution {
public:
    int longestConsecutive(vector<int>& nums) {
        unordered_set<int> numbers(nums.begin(), nums.end());
        int longest = 0;

        for (int num : numbers) {
            // num - 1 不存在，说明 num 是一段连续序列的起点
            if (num == INT_MIN || numbers.find(num - 1) == numbers.end()) {
                int current = num;
                int length = 1;

                while (current != INT_MAX &&
                       numbers.find(current + 1) != numbers.end()) {
                    ++current;
                    ++length;
                }

                longest = max(longest, length);
            }
        }

        return longest;
    }
};
```

==解析==

- **为什么使用 `unordered_set`**：只需要判断数字是否存在，不需要保存下标或出现次数；集合还能自动去重。
- **核心剪枝**：只有当 `num - 1` 不存在时，`num` 才是连续序列的起点，才从它开始向后查找。
- **为什么不是 $O(n^2)$**：外层循环检查每个不同数字是否为起点；每段连续序列只会从起点完整扫描一次。所有 `while` 循环累计访问的数字数量不超过集合大小，因此平均时间复杂度是 $O(n)$。
- **边界判断**：先处理 `INT_MIN` 和 `INT_MAX`，避免执行 `num - 1` 或 `current + 1` 时发生有符号整数溢出。
- **空间复杂度**：$O(n)$。
