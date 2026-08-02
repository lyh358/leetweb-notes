# 核心思想：

## 双指针扫描+谁先结束谁先按走

- #### 两个区间列表都是**已排序**且**内部不重叠**的。用两个指针 `i` 和 `j` 分别扫描两个列表，就像合并有序链表一样。

---

## （1）关键观察：两个区间的交集怎么求？

- ### 对于区间 `A = [a1, a2]` 和 `B = [b1, b2]`：

```plain
交集起点 = max(a1, b1)  ← 取较晚开始的
交集终点 = min(a2, b2)  ← 取较早结束的
```

- ### 有交集的条件：**起点 ≤ 终点**

```plain
A: |-------|
B:      |-------|
交集:   |--|      ← max(a1,b1) 到 min(a2,b2)
```

------

## （2）指针移动策略：谁结束早谁走

#### 这是本题最精髓的地方：

```cpp
if (firstList[i][1] < secondList[j][1]) {
    i++;  // first结束得更早，往后走
} else {
    j++;  // second结束得更早，往后走
}
```

### 为什么？

- #### 如果 `first[i]` 结束得更早，它**不可能**再和 `second[j]` 之后的区间有交集（因为后面的区间开始时间更晚）

- #### 所以 `first[i]` 的使命完成了，换 `first[i+1]` 来和当前的 `second[j]` 尝试交集

---

# 完整代码

```c++
class Solution {
public:
    vector<vector<int>> intervalIntersection(vector<vector<int>>& firstList, vector<vector<int>>& secondList) {
        //创建结果数组
        vector<vector<int>> result;

        int i=0,j=0;
        while(i<firstList.size()&&j<secondList.size())
        {
            //创建可能重叠区域的头和尾
            int start=max(firstList[i][0],secondList[j][0]);
            int end=min(firstList[i][1],secondList[j][1]);
			
            //判断是否形成重叠区间
            if(start<=end)
            {
                result.push_back({start,end});
            }
            
            //谁先结束谁先走
            if(firstList[i][1]<secondList[j][1])
            {
                i++;
            }
            else
            {
                j++;
            }
        }
        return result;
    }
};
```

