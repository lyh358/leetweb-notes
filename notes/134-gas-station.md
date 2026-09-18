## 题目简述

gas 数组：`gas[i]` 代表在第 `i` 个加油站可以加的油量
cost 数组：`cost[i]` 代表从第 `i` 开到 `i+1` 需要消耗的油量

> 环形路线：一共 n 个加油站，从 n-1 回到 0。
> 问：**从哪个起点出发，可以绕一圈回到起点**，存在则返回起点下标；不存在返回 -1。
> 前提：如果有解，解唯一。

---

# 通用思路（贪心，最优解法 O (n)）

### 核心观察

1. **总油量 ≥ 总消耗，才一定有解；否则直接返回 -1**
sum (gas) - sum (cost) < 0 → 不可能环游一圈。
2. 遍历加油站，维护**当前剩余油量 curOil**
  - 从起点 start 出发，每到一站：`curOil += gas[i] - cost[i]`
  - 如果 `curOil < 0`：说明**从 start 到 i 之间所有点都不能当起点**。把起点直接设置为 `i+1`，并且重置`curOil=0`
  
  原因：如果 start 到 i 走到 i 时油不够，那么 start, start+1 ... i 中任意一个作为起点，走到 i 都会断油，全部排除。
3. 最后判断总油量是否足够：够就返回 start，否则返回 - 1。

### 数据结构

不需要复杂数据结构！只用两个数组 gas、cost；几个变量：

- `total`：全局总剩余油量
- `curOil`：当前起点出发累计剩余油
- `start`：候选起点

> 不需要队列、栈、链表。纯贪心 + 一次遍历。

## 伪代码

```
function canCompleteCircuit(gas[], cost[]):
    n = gas.length
    total = 0
    curOil = 0
    start = 0

    for i from 0 to n-1:
        diff = gas[i] - cost[i]
        total += diff
        curOil += diff

        if curOil < 0:
            // start到i全部作废，下一个候选起点 i+1
            start = i + 1
            curOil = 0
    
    if total >= 0:
        return start
    else:
        return -1
```

### 逻辑演示小例子

gas = [1,2,3,4,5]
cost = [3,4,5,1,2]
总差值：(1-3)+(2-4)+(3-5)+(4-1)+(5-2) = -2-2-2+3+3=0 ≥0，存在解。
遍历后得到 start=3。
