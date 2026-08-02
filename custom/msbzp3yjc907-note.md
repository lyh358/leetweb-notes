# 核心思路：哈希集合（unordered_map去重）

---

# 方法一：检查有没有元素被去重

```c++
class Solution {
public:
    bool containsDuplicate(vector<int>& nums) {
        
        //将nums数组里的元素从开头到结尾加入哈希集st
        unordered_set<int> st(nums.begin(), nums.end());
        //看看有没有元素被查重过滤
        return st.size() < nums.size();
    }
};
```

---

# 方法二：使用.insert()方法的返回值结构

## 返回值类型

`unordered_set::insert(x)` 返回一个 **`pair<iterator, bool>`**：

| 成员      | 类型       | 含义                                              |
| :-------- | :--------- | :------------------------------------------------ |
| `.first`  | `iterator` | 指向元素 `x` 的迭代器（无论新旧）                 |
| `.second` | `bool`     | **是否插入成功**（`true`=新插入，`false`=已存在） |

------

## `.second` 的常见用法

### 1. 判断元素是否重复（最常用）

```cpp
unordered_set<int> st;
if (st.insert(x).second) {
    cout << x << " 是新元素，插入成功" << endl;
} else {
    cout << x << " 已存在，插入失败" << endl;
}
```

### 2. LeetCode 典型场景：找第一个重复元素

```cpp
for (int x : nums) {
    if (!st.insert(x).second) {  // 如果插入失败（已存在）
        return x;  // 找到重复元素
    }
}
```

等价于：

```cpp
if (st.count(x)) return x;  // 先查再插
st.insert(x);
```

但 `.second` 写法只需**一次哈希查找**，效率更高。

# 完整代码

```c++
class Solution {
public:
   bool containsDuplicate(vector<int>& nums) 
   {
        //哈希集合
        //先查后存，将数字挨个存入哈希集合，新数如果能在集合里面找到，就重复
        unordered_set<int> hashset;
        for(int i=0;i<nums.size();i++)
        {
           auto it=hashset.find(nums[i]);
           if(it!=hashset.end())
           {
                return true;
           }

           hashset.insert(nums[i]);
        }
        return false;
    }
   
};
```

