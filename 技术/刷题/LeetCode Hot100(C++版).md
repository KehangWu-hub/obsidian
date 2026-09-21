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
        unordered_map<string, vector<string>> hashtable;

        for (string& str : strs) { //这个和迭代器写法是一样的
            string key = str;
            sort(key.begin(), key.end()); //注意不要写成sort(key)，曾经出错过
            hashtable[key].push_back(str);
        }

        vector<vector<string>> answer;

        for (auto it = hashtable.begin(); it != hashtable.end(); it++) {
            answer.push_back(it->second);
        }

        return answer;
    }
};
```

==解析==

- **生成分组键**：遍历 `strs` 时，`str` 表示当前单词。将它复制到 `key` 并排序，字母异位词会得到相同的 `key`；例如 `"eat"` 和 `"tea"` 的 `key` 都是 `"aet"`。
- **为什么不直接排序 `str`**：`sort()` 会原地修改字符串。对 `key` 排序可以保留 `str` 的原始内容，便于将原单词存入结果。
- **按键分组**：`hashtable` 保存“排序后的字符串 → 原单词列表”。执行 `hashtable[key].push_back(str)` 时，如果 `key` 不存在，`hashtable[key]` 会先自动创建一个空的 `vector<string>`。
- **生成答案**：遍历 `hashtable` 时，`it->second` 就是某一组字母异位词，将它加入 `answer` 即可。`unordered_map` 不保证遍历顺序，但题目允许以任意顺序返回。
- **复杂度**：设字符串数量为 $n$，单个字符串的最大长度为 $k$，时间复杂度为 $O(nk\log k)$，额外空间复杂度为 $O(nk)$。

---

### 题目 3：最长连续序列（LeetCode 128，中等）

==原题==

给定一个未排序的整数数组 `nums`，找出数字连续的最长序列长度。序列元素不要求在原数组中连续，算法时间复杂度应为 $O(n)$。

- **示例**：输入 `nums = [100, 4, 200, 1, 3, 2]`，输出 `4`，因为最长连续序列是 `[1, 2, 3, 4]`。

==错误答案==
```cpp
class Solution {
public:
    int longestConsecutive(vector<int>& nums) {
        int answer = 1;

        sort(nums.begin(), nums.end());

        int count = 1;

        for (auto it = nums.begin(); it != nums.end(); it++) {
            if (nums.find(*it + 1) != nums.end != nums.end()) {
                count++;
            } else {
                answer = max(answer, count);
                count = 1;
            }
        }

        return answer;
    }
};
```

==错误解析==

- **`vector` 没有 `find` 成员函数**：`nums.find(...)` 无法通过编译。若要在线性容器中查找，应使用 `<algorithm>` 中的 `std::find(nums.begin(), nums.end(), *it + 1)`；并加上头文件`<algorithm>`。
- **没有考虑空数组**：`answer` 和 `count` 都初始化为 `1`，所以当 `nums` 为空时会返回 `1`，但正确结果应为 `0`。
- **没有考虑重复元素**：例如 `[1, 2, 2, 3]` 中，两个 `2` 都会让 `count` 增加，可能错误地得到长度 `4`；最长连续序列的长度实际为 `3`。
- **时间复杂度不符合要求**：排序本身需要 $O(n\log n)$；若再对每个元素调用一次 `std::find`，每次查找是 $O(n)$，整体会退化为 $O(n^2)$，不满足题目要求的 $O(n)$。


==标准答案（哈希集合 + 起点剪枝）==

```cpp
class Solution {
public:
    int longestConsecutive(vector<int>& nums) {
        unordered_set<int> num_set;

        for (int num : nums) {
            num_set.insert(num); //map和set都没有push_back
        }

        int answer = 0;

        for (int num : num_set) {
            // num 没有前驱，说明它是一段连续序列的起点
            if (!num_set.count(num - 1)) {
                int currentNum = num;
                int currentStreak = 1;

                while (num_set.count(currentNum + 1)) {
                    currentNum += 1;
                    currentStreak += 1;
                }

                answer = max(answer, currentStreak);
            }
        }

        return answer;
    }
};
```

==解析==

- **构造哈希集合**：先遍历 `nums`，将所有数字插入 `unordered_set`。哈希集合会自动去除重复元素，并且平均可以在 $O(1)$ 时间内判断一个数字是否存在。
- **寻找序列起点**：遍历 `num_set`，通过 `num_set.count(num - 1)` 判断 `num` 的前一个数字是否存在。只有 `num - 1` 不存在时，`num` 才是一段连续序列的起点。例如在 `[1, 2, 3, 4]` 中，只会从 `1` 开始向后查找，而不会再从 `2`、`3` 或 `4` 重复查找。
- **向后统计连续长度**：确定起点后，用 `currentNum` 记录当前数字，用 `currentStreak` 记录当前序列长度。只要集合中存在 `currentNum + 1`，就继续向后移动，并将长度加一。
- **更新最长长度**：一段连续序列查找结束后，使用 `max(answer, currentStreak)` 更新目前找到的最长长度。`answer` 初始值为 `0`，因此输入为空数组时会直接返回 `0`。
- **为什么整体是 $O(n)$**：虽然代码中有两层循环，但只有连续序列的起点会进入完整的 `while` 查找。每个不同的数字最多作为某段序列的一部分被访问一次，因此平均时间复杂度为 $O(n)$，而不是 $O(n^2)$。
- **空间复杂度**：哈希集合最多保存 $n$ 个不同的数字，因此额外空间复杂度为 $O(n)$。
- `set` / `unordered_set` 没有 `[]`
- 
```
for (int num : num_set)
```

**不会执行 `num++`。**

它的意思是：

**依次从 `num_set` 里面取出一个元素，把这个元素赋值给 `num`。**

例如：

```
unordered_set<int> num_set = {1, 2, 3, 4};
```

那么：

```
for (int num : num_set)
```

可以理解成：

```
num = 集合中的第一个元素;
执行循环体;

num = 集合中的第二个元素;
执行循环体;

num = 集合中的第三个元素;
执行循环体;

...
```

和 `num++` 是两回事

# 双指针

## 1. 概念

双指针是指在遍历过程中同时维护两个位置，通过指针的移动缩小搜索范围，或完成原地修改。这里的“指针”通常是数组下标或迭代器，不一定是 C++ 中的指针类型。

双指针常用于数组、字符串和链表，通常能把两层枚举的 $O(n^2)$ 时间复杂度降为 $O(n)$。能否使用双指针，关键在于移动某个指针后，能明确排除一部分不可能的答案。

## 2. 常见类型

### 左右指针

两个指针分别从序列两端开始，根据当前状态向中间移动：

```cpp
int left = 0;
int right = nums.size() - 1;

while (left < right) {
    if (/* 应移动左指针 */) {
        left++;
    } else {
        right--;
    }
}
```

这种写法常用于有序数组查找、盛最多水的容器和接雨水等问题。

### 快慢指针

两个指针从同一方向出发：快指针负责遍历，慢指针记录下一个需要写入的位置。

```cpp
int slow = 0;

for (int fast = 0; fast < nums.size(); fast++) {
    if (/* 当前元素需要保留 */) {
        nums[slow] = nums[fast];
        slow++;
    }
}
```

这种写法常用于原地删除元素、移动零和有序数组去重。

### 固定一个数，再使用左右指针

处理三数之和时，可以先排序，再枚举第一个数，并在剩余区间中使用左右指针寻找另外两个数。这样可以将时间复杂度从 $O(n^3)$ 降为 $O(n^2)$。

## 3. 题目

### 题目 1：移动零（LeetCode 283，简单）

==原题==

给定一个数组 `nums`，将所有 `0` 移动到数组末尾，同时保持非零元素的相对顺序。要求原地修改数组，不能复制整个数组。

- **示例**：输入 `nums = [0, 1, 0, 3, 12]`，修改后为 `[1, 3, 12, 0, 0]`。

==错误答案==

```cpp
class Solution {
public:
    void moveZeroes(vector<int>& nums) {
        auto slow = nums.begin();

        for (auto fast = nums.begin(); fast < nums.end(); ++fast) {
            if (*slow) {
                slow++;
            } else {
                int num = *slow;
                *slow = *fast;
                *fast = num;
            }
        }
    }
};
```

==问题原因==

整体想法没有问题：`slow` 在当前位置不是 `0` 时向后移动，在遇到 `0` 时等待 `fast` 带来后面的元素；`fast` 则始终向后遍历。

问题在于交换完成后，`slow` 所在位置已经变成了非零元素，但本轮没有立即移动 `slow`。下一轮循环需要先用一次 `if (*slow)` 才能让 `slow` 前进，而 `fast` 在这一轮仍会继续前进，因此两个指针之间会出现一轮不同步。

例如输入 `[0, 1, 2, 0]`：

- `fast` 找到 `1` 后，将它与第一个 `0` 交换，但 `slow` 仍停在原处。
- 下一轮 `fast` 已经来到 `2`，这一轮却只执行了 `slow++`，没有处理当前的 `2`。
- `fast` 继续走到数组末尾，已经没有机会再把 `2` 交换到前面。

所以，本质上就是 `slow` 在交换后慢了一步。当 `fast` 走到后面时，剩余循环次数可能不足，最后一个需要前移的非零元素便来不及交换。

==答案（快慢指针）==

```cpp
class Solution {
public:
    void moveZeroes(vector<int>& nums) {
        int n = nums.size();
        int slow = 0;
        int fast = 0;

        while (fast < n) {
            if (nums[fast]) {
                swap(nums[slow], nums[fast]);
                slow++;
            }
            fast++;
        }
    }
};
```

==解析==

- **快指针 `fast`**：遍历数组，寻找所有非零元素。
- **慢指针 `slow`**：指向下一个非零元素应该放入的位置。
- **交换元素**：当 `nums[fast]` 非零时，将它与 `nums[slow]` 交换，然后移动 `slow`。被换到后面的元素是 `0`，因此不需要再单独补零。
- **保持相对顺序**：`fast` 从左到右依次处理非零元素，所以这些元素的相对顺序不会改变。
- **复杂度**：时间复杂度为 $O(n)$，额外空间复杂度为 $O(1)$。

---

### 题目 2：盛最多水的容器（LeetCode 11，中等）

==原题==

给定一个整数数组 `height`，每个元素表示一条竖线的高度。选择两条线与横轴组成容器，求容器能够盛水的最大面积。

两条线下标为 `left` 和 `right` 时，面积为：

$$
\min(height[left], height[right]) \times (right-left)
$$

==答案（左右指针）==

```cpp
class Solution {
public:
    int maxArea(vector<int>& height) {
        int left = 0, right = height.size() - 1;
        int ans = 0;

        while (left < right) {
            int area = min(height[left], height[right]) * (right - left);
            ans = max(ans, area);

            if (height[left] <= height[right]) {
                ++left;
            }
            else {
                --right;
            }
        }

        return ans;
    }
};
```

==解析==

- **初始位置**：左右指针放在数组两端，此时容器宽度最大。
- **面积由短板决定**：容器高度是左右两条线中较短的一条。
- **为什么移动短板**：移动指针后宽度一定减小。如果移动较高的一侧，较短的一侧不变，容器高度不会增加，因此面积不可能变大。只有移动较短的一侧，才可能找到更高的短板。
- **复杂度**：两个指针最多各移动 $n$ 次，时间复杂度为 $O(n)$，额外空间复杂度为 $O(1)$。

---

### 题目 3：三数之和（LeetCode 15，中等）

==原题==

给定一个整数数组 `nums`，找出所有和为 `0` 且下标互不相同的三元组。答案中不能包含重复的三元组。

- **示例**：输入 `nums = [-1, 0, 1, 2, -1, -4]`，输出 `[[-1, -1, 2], [-1, 0, 1]]`。

这道题目不可以用set来实现去重

`set<int>`：

> **去掉元素重复，同时丢失重复次数**

而 Three Sum 需要：

> **保留重复次数，但最终答案不能重复**

==答案（排序 + 左右指针）==

```cpp
class Solution {
public:
    vector<vector<int>> threeSum(vector<int>& nums) {
        int n = nums.size();
        sort(nums.begin(), nums.end());

        vector<vector<int>> ans;

        // 枚举 a
        for (int first = 0; first < n; ++first) {
            // 需要和上一次枚举的数不同
            if (first > 0 && nums[first] == nums[first - 1]) {
                continue;
            }

            // c 对应的指针初始指向数组最右端
            int third = n - 1;
            int target = -nums[first];

            // 枚举 b
            for (int second = first + 1; second < n; ++second) {
                // 需要和上一次枚举的数不同
                if (second > first + 1 && nums[second] == nums[second - 1]) {
                    continue;
                } //second > first + 1这个判定很重要，出错过，不能少

                // 保证 b 对应的指针在 c 对应的指针左侧
                while (second < third && nums[second] + nums[third] > target) {
                    --third; //关键：third只会一直向左走
                } 
                //注意这里要用while而不是if不然只会执行一次
                //注意second < third必须要加，不然可能到不了下面的if (second == third)就已经越界了

                // 指针重合后，后续不可能再找到满足条件的 c
                if (second == third) {
                    break;
                }

                if (nums[second] + nums[third] == target) {
                    ans.push_back({nums[first], nums[second], nums[third]});
                }
            }
        }

        return ans;
    }
};
```

==解析==

- **先排序**：排序后，数字从小到大排列，既能根据当前和移动 `third`，也方便跳过重复数字。
- **枚举第一个数**：`first` 对应三元组中的 $a$。确定 `nums[first]` 后，另外两个数需要满足 `b+c=-a`，所以令 `target = -nums[first]`。
- **枚举第二个数**：`second` 对应 $b$，从 `first + 1` 开始向右移动。
- **移动第三个指针**：`third` 对应 $c$，初始位于数组末尾。当 `nums[second] + nums[third] > target` 时，当前和太大，因此将 `third` 向左移动。
- **为什么 `third` 不需要重新回到末尾**：数组已经排序。随着 `second` 向右移动，`nums[second]` 只会变大，要使两数之和仍等于 `target`，`third` 只可能保持不动或继续向左移动。
- **指针不能重合**：三个数必须来自不同下标，因此需要保证 `second < third`。两者重合后，后续的 `second` 只会继续增大，可以直接结束内层循环。
- **去重**：`first` 与上一次取值相同时跳过；同一个 `first` 下，`second` 与上一次取值相同时也跳过，从而避免加入重复三元组。
- **复杂度**：排序需要 $O(n\log n)$。每个 `first` 下，`second` 向右移动，`third` 只向左移动，因此查找需要 $O(n)$，总体时间复杂度为 $O(n^2)$。忽略返回结果所占空间时，额外空间复杂度主要取决于排序实现。

---

### 题目 4：接雨水（LeetCode 42，困难）

==原题==

给定一个非负整数数组 `height`，每个元素表示宽度为 `1` 的柱子高度，计算下雨后能够接住的雨水总量。

==答案（左右指针）==

```cpp
class Solution {
public:
    int trap(vector<int>& height) {
        int left = 0;
        int right = height.size() - 1;
        int leftMax = 0;
        int rightMax = 0;
        int answer = 0;

        while (left < right) {
            leftMax = max(leftMax, height[left]);
            rightMax = max(rightMax, height[right]);

            if (height[left] < height[right]) {
                answer += leftMax - height[left];
                left++;
            } else {
                answer += rightMax - height[right];
                right--;
            }
        }

        return answer;
    }
};
```

==解析==

- **单个位置的雨水量**：当前位置能接的水由左侧最高柱和右侧最高柱中较矮的一侧决定，再减去当前位置的高度。
- **维护边界最高值**：`leftMax` 和 `rightMax` 分别记录从两端遍历到当前位置时见过的最高柱。
- **为什么移动较低的一侧**：当 `height[left] < height[right]` 时，右侧至少存在 `height[right]` 这根更高的柱子，因此左侧当前位置的水量可以由 `leftMax` 确定；另一种情况同理。
- **不会得到负数**：更新最高值后再计算，所以 `leftMax >= height[left]`，`rightMax >= height[right]`。
- **复杂度**：时间复杂度为 $O(n)$，额外空间复杂度为 $O(1)$。
