给定两个由一些 **闭区间** 组成的列表，`firstList` 和 `secondList` ，其中 `firstList[i] = [starti, endi]` 而 `secondList[j] = [startj, endj]` 。每个区间列表都是成对 **不相交** 的，并且 **已经排序** 。

返回这 **两个区间列表的交集** 。

形式上，**闭区间** `[a, b]`（其中 `a <= b`）表示实数 `x` 的集合，而 `a <= x <= b` 。

两个闭区间的 **交集** 是一组实数，要么为空集，要么为闭区间。例如，`[1, 3]` 和 `[2, 4]` 的交集为 `[2, 3]` 。

**示例 1：**

![image](https://assets.leetcode.com/uploads/2019/01/30/interval1.png)```
输入：firstList = [[0,2],[5,10],[13,23],[24,25]], secondList = [[1,5],[8,12],[15,24],[25,26]]
输出：[[1,2],[5,5],[8,10],[15,23],[24,24],[25,25]]

```markdown

**示例 2：**
```

输入：firstList = [[1,3],[5,9]], secondList = []
输出：[]

```markdown

**示例 3：**
```

输入：firstList = [], secondList = [[4,8],[10,12]]
输出：[]

```markdown

**示例 4：**
```

输入：firstList = [[1,7]], secondList = [[3,10]]
输出：[[3,7]]

```undefined
思路 + 数据结构算法：双指针
不需要复杂数据结构，只用两个指针 i,j，分别指向两个列表当前区间；结果存放在二维 vector。核心规则：
两个区间 [a1,a2] 和 [b1,b2] 的交集：

交集起点：max(a1,b1)
交集终点：min(a2,b2)
只有 start <= end，代表确实存在交集，把这个区间加入答案。

指针移动规则：谁的区间结束更早，谁往后移

理由：早结束的区间，不可能再和对方后面任何区间相交；晚结束的区间，还有机会和对方下一个区间相交。

循环终止：i 或 j 遍历完对应列表。

时间复杂度：\(O(m+n)\)，m、n 分别是两个区间数组长度
空间复杂度：\(O(1)\)（不算输出数组）
伪代码function intervalIntersection(firstList, secondList):
    i = 0, j = 0
    res = 空二维数组
    m = firstList.size()
    n = secondList.size()

    while i < m && j < n:
        a_start = firstList[i][0], a_end = firstList[i][1]
        b_start = secondList[j][0], b_end = secondList[j][1]

        inter_start = max(a_start, b_start)
        inter_end = min(a_end, b_end)

        if inter_start <= inter_end:
            res.push_back( [inter_start, inter_end] )
        
        // 结束早的指针前移
        if a_end < b_end:
            i += 1
        else:
            j += 1
    return res
```
