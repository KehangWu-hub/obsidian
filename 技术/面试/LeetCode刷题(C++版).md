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

### `find()`：判断键是否存在

```cpp
unordered_map<int, int> mp;
mp[3] = 7;

if (mp.find(3) != mp.end()) {
    // 键 3 存在
}
```

`find(key)` 返回迭代器；找不到时返回 `end()`。这是兼容性最好的存在性判断方式。

### `operator[]`：读取或写入值

```cpp
unordered_map<string, int> count;

count["apple"] = 2;
count["apple"]++;
```

`mp[key]` 在键不存在时会先插入一个默认值：

- 数值默认是 `0`。
- `string` 默认是空字符串。
- `vector<T>` 默认是空数组。

因此，`mp[key]++` 很适合计数，`groups[key].push_back(value)` 很适合分组。

```cpp
unordered_map<char, vector<string>> groups;
groups['a'].push_back("apple");
```

但如果只是想检查键是否存在，不要直接写 `mp[key]`，否则查询动作会修改哈希表：

```cpp
unordered_map<string, int> mp;
int value = mp["missing"]; // 插入 {"missing", 0}
```

纯查询应使用 `find()`；确定键存在时，也可以使用 `at()` 读取。`at()` 不会自动插入，但键不存在时会抛出异常。

### `find()` 返回的迭代器

```cpp
auto it = mp.find(key);

if (it != mp.end()) {
    auto storedKey = it->first;
    auto storedValue = it->second;
}
```

- `it->first` 是键。
- `it->second` 是值。

已经拿到迭代器时，直接使用 `it->second`，不必再用 `mp[key]` 查一次。

### 计数与分组

```cpp
unordered_map<int, int> count;

for (int num : nums) {
    count[num]++;
}
```

```cpp
unordered_map<string, vector<string>> groups;

for (const string& word : words) {
    string key = word;
    sort(key.begin(), key.end());
    groups[key].push_back(word);
}
```

`operator[]` 会自动创建默认值，因此 C++ 不需要 Python 的 `defaultdict`。

### 遍历 `unordered_map`

```cpp
for (const auto& [key, value] : mp) {
    // C++17 结构化绑定
}
```

如果只需要值：

```cpp
vector<vector<string>> answer;

for (auto& [key, group] : groups) {
    answer.push_back(move(group));
}
```

`move(group)` 可以把分组内容移动到答案中，避免复制。移动后不要再依赖 `group` 原来的内容。如果暂时不熟悉移动语义，写 `answer.push_back(group)` 也完全正确。

### `unordered_set`：存在性判断与去重

```cpp
unordered_set<int> seen(nums.begin(), nums.end());

seen.insert(4);
seen.erase(2);

if (seen.find(3) != seen.end()) {
    // 3 存在
}
```

集合只保存元素，不保存额外信息。如果只需要去重或判断某个值是否存在，优先使用 `unordered_set`。

### `sort()`：生成统一的字符串键

```cpp
string key = "tea";
sort(key.begin(), key.end());
// key == "aet"
```

`sort()` 会直接修改原对象。因此分组异位词时，应先复制一份字符串作为键，保留原单词用于答案。

### `reserve()`：提前预留空间

已知大致元素数量时，可以提前预留桶空间，减少扩容和重新哈希：

```cpp
unordered_map<int, int> mp;
mp.reserve(nums.size());
```

这通常只是性能优化，不改变算法复杂度，也不是每道题都必须写。

### `max()`：维护最优答案

```cpp
longest = max(longest, currentLength);
```

`max(a, b)` 返回两者中的较大值，两个参数通常应具有相同类型。

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
