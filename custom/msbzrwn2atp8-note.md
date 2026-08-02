# 986.区间列表的交集

给定两个由一些 **闭区间** 组成的列表，`firstList` 和 `secondList` ，其中 `firstList[i] = [starti, endi]` 而 `secondList[j] = [startj, endj]` 。每个区间列表都是成对 **不相交** 的，并且 **已经排序** 。

返回这 **两个区间列表的交集** 。

形式上，**闭区间** `[a, b]`（其中 `a <= b`）表示实数 `x` 的集合，而 `a <= x <= b` 。

两个闭区间的 **交集** 是一组实数，要么为空集，要么为闭区间。例如，`[1, 3]` 和 `[2, 4]` 的交集为 `[2, 3]` 。

**示例 1：**

![img](https://assets.leetcode.com/uploads/2019/01/30/interval1.png)

```
输入：firstList = [[0,2],[5,10],[13,23],[24,25]], secondList = [[1,5],[8,12],[15,24],[25,26]]
输出：[[1,2],[5,5],[8,10],[15,23],[24,24],[25,25]]
```

**示例 2：**

```
输入：firstList = [[1,3],[5,9]], secondList = []
输出：[]
```

**示例 3：**

```
输入：firstList = [], secondList = [[4,8],[10,12]]
输出：[]
```

**示例 4：**

```
输入：firstList = [[1,7]], secondList = [[3,10]]
输出：[[3,7]]
```

**提示**

- `0 <= firstList.length, secondList.length <= 1000`
- `firstList.length + secondList.length >= 1`
- `0 <= starti < endi <= 109`
- `endi < starti+1`
- `0 <= startj < endj <= 109 `
- `endj < startj+1`

---

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

