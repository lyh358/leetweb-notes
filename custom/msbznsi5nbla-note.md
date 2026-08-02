# 核心思路：

- ## 使用  vector模拟栈   通过 弹出 实现路径切换（..）

  - #### 这里使用vector，如果使用stack最后拼接是

- ## 把字符串变成流`stringstream ss(path);`

- ## 按  ‘  /  ‘ 分割读取`getline(ss, s, '/')`

## 核心思路：栈模拟目录切换

#### 把路径按 `/` 分割成若干段，遇到目录名**入栈**（进入目录），遇到 `..`**出栈**（返回上级），`.` 和空段忽略。

```plain
/home/user/../documents
  ↓ 分割
["home", "user", "..", "documents"]
  ↓ 处理
入栈 home → 入栈 user → 出栈（user被pop）→ 入栈 documents
  ↓ 结果
/home/documents
```

---

# 实现步骤

## 1. 分割路径（`stringstream` + `getline`）

```cpp
stringstream ss(path);        // 把字符串变成流
while (getline(ss, s, '/')) {  // 按 '/' 分割读取
    // s 就是 "home", "user", "..", 等
}
```

## 2. 栈的四种处理逻辑

```cpp
if (s.empty() || s == ".") {
    continue;              // 情况1：空串（连续/导致）或当前目录，忽略
}
if (s != "..") {
    stk.push_back(s);      // 情况2：正常目录名，入栈（进入目录）
} else if (!stk.empty()) {
    stk.pop_back();        // 情况3：返回上级，出栈（如果栈非空）
}
// 情况4：s==".." 但栈空，已经在根目录，无法回退，忽略
```

## 3. 结果拼接

```cpp
for (string& s : stk) {
    ans += '/';
    ans += s;
}
return stk.empty() ? "/" : ans;
```

**注意**：

- 每个目录前加 `/`，确保格式正确
- 栈空时返回 `"/"`（根目录），不能返回空串

---

# 完整代码

```c++
class Solution {
public:
    string simplifyPath(string path) {
        
        vector<string> stk;
        
        stringstream ss(path); 
        string s;
        
        while (getline(ss, s, '/')) 
        {
            //如果是空值（可能是多个/）或者一个.  直接忽略
            if (s.empty() || s == ".") //注意这里的是字符串对比，要用双引号，如果用单引号就是字符串和char对比会报错
            {
                continue;
            }
            //如果不是返回上一级（..），正常入数组
            if (s != "..") 
            {
                stk.push_back(s);
            } 
            //上面是if这里是else if,只有s是..的时候才会执行elseif
            else if (!stk.empty()) 
            {
                stk.pop_back();//弹出上一级，模拟返回
            }
        }

        string ans;
        for (string& s : stk) //只有vector才能使用这个遍历，stack不行，因为它是容器适配器没有迭代器
        {
            //组装操作
            ans += '/';
            ans += s;
        }
        //如果为空就返回一个/就行
        return stk.empty() ? "/" : ans;
    }
};
```

---

# 什么是模拟栈？

### **模拟栈**（Simulated Stack）是指**用其他支持动态数组操作的数据结构**（通常是 `vector`、`deque` 或普通数组）**手动实现栈的 LIFO（后进先出）逻辑**，而不是直接使用标准库的 `std::stack`。

## 为什么叫"模拟"？

标准库的 `std::stack` 是一个**容器适配器**，它封装了底层的容器（默认是 `deque`），但**只暴露栈的接口**：

- `push()` / `pop()` / `top()` / `empty()`

**关键限制**：`std::stack` **没有迭代器**，你不能遍历它、不能随机访问、不能用范围 for 循环。

而**模拟栈**就是直接用底层容器（如 `vector`）自己操作，**保留栈的功能，同时获得容器的灵活性**。

## 什么时候必须"模拟栈"？

### 1. **需要遍历栈内元素**（最常见）

如单调栈问题需要查看栈中多个元素，或像路径简化这样最后要拼接结果。

### 2. **需要随机访问栈中元素**

```cpp
// 查看栈底元素（标准 stack 做不到）
string bottom = stk[0];  // vector 可以，stack 不行
```

### 3. **需要同时维护多个指针/索引**

如双栈实现队列时，经常需要操作栈的特定位置。

## 一句话总结

> **模拟栈 = 用 vector/数组/链表手动实现 `push`/`pop`，既保持栈的 LIFO 特性，又获得遍历、随机访问等额外能力。** 当你发现 `std::stack` 的功能不够用（特别是需要遍历时），就该考虑用 `vector` 来模拟。
