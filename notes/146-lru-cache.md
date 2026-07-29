# LeetCode 146 - LRU 缓存（LRU Cache）

> **标签**：哈希表、双向链表、设计  
> **难度**：中等  
> **核心考点**：哈希表 + 双向链表实现 O(1) 的 get 和 put

---

## 一、题目描述

请你设计并实现一个满足 **LRU (最近最少使用) 缓存** 约束的数据结构。

实现 `LRUCache` 类：
- `LRUCache(int capacity)`：以正整数作为容量 `capacity` 初始化 LRU 缓存
- `int get(int key)`：如果关键字 `key` 存在于缓存中，则返回关键字的值，否则返回 `-1`
- `void put(int key, int value)`：如果关键字 `key` 已经存在，则变更其数据值 `value`；如果不存在，则向缓存中插入该组 `key-value`。如果插入操作导致关键字数量超过 `capacity`，则应该**逐出最久未使用**的关键字。

函数 `get` 和 `put` 必须以 **O(1)** 的平均时间复杂度运行。

**示例**：
```
输入:
["LRUCache", "put", "put", "get", "put", "get", "put", "get", "get", "get"]
[[2], [1, 1], [2, 2], [1], [3, 3], [2], [4, 4], [1], [3], [4]]

输出:
[null, null, null, 1, null, -1, null, -1, 3, 4]
```

---

## 二、核心思路：哈希表 + 双向链表

### 2.1 为什么需要两个数据结构？

| 需求 | 哈希表 | 双向链表 |
|------|--------|---------|
| O(1) 查找 key | ✅ | ❌ 需要遍历 O(n) |
| O(1) 移动/删除节点 | ❌ | ✅ `splice`/`erase` |
| 维护使用顺序 | ❌ | ✅ 头部最新，尾部最旧 |

**结论**：两者结合，哈希表负责查找，双向链表负责维护顺序。

### 2.2 数据结构图解

```
┌─────────────────────────────────────────────────────────────┐
│  双向链表 (list)                    哈希表 (unordered_map)    │
│                                                             │
│  front (最近使用 MRU)                                        │
│    ↓                                                        │
│  [k1,v1] ↔ [k2,v2] ↔ [k3,v3]       key → 链表节点迭代器      │
│    ↑                       ↓                                │
│  back (最久未使用 LRU)                                       │
│                                                             │
│  作用：维护使用顺序                  作用：O(1) 定位节点       │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、代码实现

### 3.1 你的代码 ⭐

```cpp
class LRUCache {
public:
    // 初始化数据结构，一共三个，核心是双向链表和哈希表
    int cap;
    // 双向链表：元素是 pair<书名，书的内容>，用于管理"书"，实现最近被访问的放到顶端
    list<pair<int, int>> ls;
    // 哈希表：用于 O(1) 查找"书"，根据书名（key）得到"书的位置"（pair 的迭代器指向）
    unordered_map<int, list<pair<int, int>>::iterator> map;

    // 构造函数：初始化容量
    LRUCache(int capacity) {
        cap = capacity;
    }

    // 查找书的内容
    int get(int key) {
        // 利用 map 的迭代器找 key 对应的书 pair 的位置（从头到尾）
        auto it_m = map.find(key);
        // 找了一遍没找到 it 会移动到末尾位置停止
        if (it_m == map.end()) {
            return -1;
        }
        // 没返回 -1 说明找到了，把当前的书放到链表头
        // splice 剪切：把 A 链表（ls）的节点（it_m->second）剪切到 B 链表的某个位置（ls.begin()）
        ls.splice(ls.begin(), ls, it_m->second);

        // 返回得到的书的内容
        return it_m->second->second;
    }

    // 把 {key, value} 存入 ls，map 也要更新
    void put(int key, int value) {
        auto it_m = map.find(key);  // 先找是否存在
        // 如果存在
        if (it_m != map.end()) {
            it_m->second->second = value;   // 覆盖书的内容
            ls.splice(ls.begin(), ls, it_m->second);  // 更新书的位置
        } else {  // 如果不存在
            // 判断缓存是否已满
            if (map.size() == cap) {
                // 删除 ls 底部的书，以及 map 中对它的记录
                map.erase(ls.back().first);
                ls.pop_back();
            }

            // 在 ls 头放入新书
            ls.emplace_front(key, value);
            // 在 map 中记录新书
            map[key] = ls.begin();
        }
    }
};

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache* obj = new LRUCache(capacity);
 * int param_1 = obj->get(key);
 * obj->put(key, value);
 */
```

---

## 四、关键方法详解

### 4.1 `list::splice` — 核心操作

```cpp
void splice(iterator position, list& other, iterator i);
```

**作用**：把 `other` 链表中 `i` 指向的节点，**剪切**到当前链表的 `position` 位置。

**特点**：
- **O(1)** 时间复杂度
- **不创建/销毁**节点，只修改指针
- 迭代器**不失效**（除了被移动的节点）

```cpp
// 把 it 指向的节点移到链表头部
ls.splice(ls.begin(), ls, it);

// 图解：
// 移动前: [A] ↔ [B] ↔ [C]    it 指向 B
// 移动后: [B] ↔ [A] ↔ [C]
```

### 4.2 `list::emplace_front` — 头部插入

```cpp
ls.emplace_front(key, value);
```

**作用**：在链表头部**原地构造**新节点。

**与 `push_front` 的区别**：

| 方法 | 用法 | 说明 |
|------|------|------|
| `push_front` | `ls.push_front({key, value})` | 先构造临时对象，再拷贝/移动到链表 |
| `emplace_front` | `ls.emplace_front(key, value)` | **原地构造**，直接传入构造参数，更高效 |

```cpp
// push_front：先创建 pair，再拷贝
pair<int, int> p(key, value);
ls.push_front(p);  // 拷贝

// emplace_front：直接在链表节点内存中构造
ls.emplace_front(key, value);  // 原地构造，无拷贝
```

### 4.3 `list::back` 和 `pop_back`

```cpp
// 获取链表尾部元素（最久未使用）
pair<int, int>& last = ls.back();  // 返回引用
int lastKey = last.first;          // 获取 key

// 删除尾部元素
ls.pop_back();  // O(1)
```

### 4.4 `unordered_map::erase`

```cpp
// 根据 key 删除
map.erase(key);  // O(1)

// 根据迭代器删除
map.erase(it);   // O(1)
```

---

## 五、执行流程可视化

### get 操作

```
缓存: cap=2, 当前: {1:10, 2:20}

链表: [2,20] ↔ [1,10]
       ↑MRU      ↑LRU

map: 1 → 指向 [1,10]
     2 → 指向 [2,20]

get(1):
  1. map.find(1) → 找到，it 指向 [1,10]
  2. splice 到头部: [1,10] ↔ [2,20]
  3. 返回 10

结果: 链表变为 [1,10] ↔ [2,20]
```

### put 操作（已存在）

```
put(1, 100):
  1. map.find(1) → 找到
  2. 更新 value: [1,10] → [1,100]
  3. splice 到头部

结果: 链表 [1,100] ↔ [2,20]
```

### put 操作（新 key，满容量）

```
缓存: cap=2, 当前: {1:10, 2:20}

put(3, 30):
  1. map.find(3) → 未找到
  2. map.size() == cap (2==2) → 需要淘汰
  3. ls.back() = [2,20] → key=2
  4. map.erase(2) → 删除 key=2
  5. ls.pop_back() → 删除 [2,20]
  6. ls.emplace_front(3, 30) → [3,30]
  7. map[3] = ls.begin()

结果: 缓存 {1:10, 3:30}
      链表 [3,30] ↔ [1,10]
```

---

## 六、复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 | 说明 |
|------|-----------|-----------|------|
| `get` | **O(1)** | O(1) | map 查找 O(1) + splice O(1) |
| `put` | **O(1)** | O(1) | map 查找 O(1) + splice/emplace O(1) |
| 总体 | - | **O(capacity)** | 最多存储 capacity 个元素 |

---

## 七、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **splice 的迭代器** | `it_m->second` 是 `list::iterator`，指向链表节点 |
| **map 的 value 类型** | 存的是 `list::iterator`，不是 `pair` 或 `int` |
| **淘汰顺序** | 淘汰 `ls.back()`（尾部，最久未使用），不是头部 |
| **emplace vs push** | `emplace_front` 原地构造，比 `push_front` 更高效 |
| **容量判断** | `map.size() == cap`，不是 `ls.size()`（虽然一样） |

---

## 八、相关方法对比总结

| 方法 | 用法 | 时间 | 说明 |
|------|------|------|------|
| `splice` | `ls.splice(pos, ls, it)` | O(1) | 剪切节点到指定位置 |
| `emplace_front` | `ls.emplace_front(args...)` | O(1) | 头部原地构造 |
| `push_front` | `ls.push_front(val)` | O(1) | 头部插入（拷贝） |
| `pop_back` | `ls.pop_back()` | O(1) | 删除尾部 |
| `back` | `ls.back()` | O(1) | 获取尾部元素 |
| `begin` | `ls.begin()` | O(1) | 获取头部迭代器 |

---

## 九、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 460 | LFU 缓存 | 频率 + 时间双维度淘汰 |
| 432 | 全 O(1) 的数据结构 | 复杂链表操作 |

---

## 十、记忆口诀

> **"LRU 缓存双结构，哈希链表来配合；哈希 O(1) 找节点，链表 splice 调顺序；头部 MRU 尾部 LRU，满了淘汰尾部去；emplace 原地构造快，splice 剪切不创建"**

---

## 十一、面试话术模板

> "这道题我使用哈希表 + 双向链表实现。哈希表存储 key 到链表节点的映射，实现 O(1) 查找；双向链表维护使用顺序，头部是最近使用的，尾部是最久未使用的。get 时通过哈希表找到节点，用 splice 移到头部；put 时如果 key 存在就更新并移到头部，不存在就插入头部，如果满了就淘汰尾部节点。两个操作都是 O(1)。"

---

*整理日期：2026-07-29*
