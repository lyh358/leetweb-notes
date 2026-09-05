## =={pink}介绍一下Vector和list容器，他们的底层分别是什么？==

- **=={yellow}vector==**=={yellow}：底层为==**=={yellow}连续动态数组==**，=={green}**内存连续，支持随机访问**==；=={green}**尾部插删效率高**==；=={green}**中间插入删除需要移动大量元素**；===={green}**扩容会重新开辟一块新内存**，**拷贝全部旧元素，释放旧内存**，会**导致迭代器失效**。==

      适用：随机访问多，尾部增删，元素数量变化不大。
- **=={yellow}list==**=={yellow}：底层为==**=={yellow}双向循环链表==**，=={green}内存不连续==，=={green}**每个节点存数据 + 前后指针**==；=={green}**不支持随机访问，任意位置插入删除只改指针，O (1)，不会造成迭代器失效**==；=={green}**遍历需要顺着指针走**==。

       适用：频繁在任意位置插入删除，很少随机访问。## =={pink}STL 常用迭代器分类，vector和list用什么迭代器==

1. **=={yellow}输入迭代器==**=={yellow}：只读，只能单向 ++==，如`istream_iterator`
2. **=={yellow}输出迭代器==**=={yellow}：只写，只能单向 ++，==如`ostream_iterator`
3. **=={yellow}前向迭代器==**=={yellow}：可读写，只能单向==`++`，=={green}单向链表==`forward_list`
4. **=={yellow}双向迭代器==**=={yellow}：支持==`++`=={yellow}、==`--`=={yellow}，==**=={yellow}list 的迭代器就是双向迭代器==**，可以前进后退，但不能直接加减数字偏移。
5. **=={yellow}随机访问迭代器==**：`++ -- + - []`=={yellow}都支持==，可以像指针一样偏移，**vector 迭代器属于随机访问迭代器**。

- =={green}vector：==**=={green}随机访问迭代器==**，支持`+ - []`等运算。
- =={green}list：==**=={green}双向迭代器==**，只支持`++ --`，不支持加减偏移。

## =={pink}vector 迭代器失效场景==

1. **=={yellow}扩容（push_back 导致空间重新分配）==**：=={green}**全部**迭代器失效==。
2. **=={yellow}erase 删除元素==**：=={green}从**被删除位置到后面所有**迭代器全部失效。==
3. =={yellow}**insert 插入**==：=={green}**扩容全部失效**==；=={green}**不扩容**，**插入点之后**迭代器失效。==

## =={pink}vector 添加元素接口==

**`push_back()`、`emplace_back()`、`insert()`、`emplace()`**
---

## =={pink}push_back 与 emplace_back 区别==

- `push_back`：=={yellow}**先构造**==对象，=={yellow}**再拷贝**== / 移动到容器。
- `emplace_back`：**=={yellow}原地直接在容器内存构造对象==**，=={yellow}**省去拷贝** / 移动==，效率更高。

## 如何限定 vector 大小

1. `reserve(n)`：分配容量 capacity，不改变元素 size，只预分配内存。
2. `resize(n)`：改变有效元素数量 size，多删少补。
