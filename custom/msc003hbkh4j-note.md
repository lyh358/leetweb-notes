# 快速排序（升序）

# 方法：==基准==+==双指针交换法==+==递归==

## 快排函数`void quickSort`

- ### 形参：

  - #### `vector<int>& nums`待排序数组

  - #### 左指针端点`int left`

  - #### 右指针端点`int left`

    - #### 他俩传进来之后不动了就

## 递归终止条件：递归到子区间无效或只有一个元素

- ### `if (left >= right) return;`

## 选基准点：

- ### 左端点作为基准点

  - #### `int base = nums[left]`

- ### 初始化左右指针位置为左右端点

  - #### `int i = left, j = right;`

## 双指针向中间扫描`while(i<j)`

- ### 右指针先走，(一直走，直到)找到第一个 < base 的元素

  - #### `while (i < j && nums[j] >= base) j--;`

- ### 左指针后走，找到第一个 > base 的元素

  - #### `while (i < j && nums[i] <= base) i++;`

- ### 交换这两个逆序元素

  - #### ` swap(nums[i], nums[j]);`

## 此时双指针扫描结束，i==j，此时i的位置，左边全小于base，右边全大于base

- ### 所以要把base对应的数nums[left]放到这来，也就是和nums[i]交换

- ### 将基准归位` swap(nums[left], nums[i]);`

## 一句话总结

> **双指针扫描的本质**：把数组分成 "≤base" 和 "≥base" 两部分
> **i==j 的位置**：就是这两部分的交界点，base 属于这里
> **swap 归位**：把最开始抠出来的 base，塞回这个交界点

递归处理i左右的两个子区间(i位置相当于以及排好了)

- `quickSort(nums, left, i - 1);`
- `quickSort(nums, i + 1, right);`

---

# 完整代码

```cpp
#include <iostream>
#include <vector>
using namespace std;

// 快速排序核心函数
void quickSort(vector<int>& nums, int left, int right) {
    // 1. 递归终止条件：区间无效或只有一个元素
    if (left >= right) return;
    
    // 2. 选基准点（最左元素），保存值
    int base = nums[left];
    int i = left, j = right;
    
    // 3. 双指针向中间扫描
    while (i < j) {
        // 3.1 右指针先走，找到第一个 < base 的元素
        while (i < j && nums[j] >= base) 
        {
            j--;
        }

        // 3.2 左指针后走，找到第一个 > base 的元素  
        while (i < j && nums[i] <= base)
        {
            i++;
        }
        // 3.3 交换这一对逆序元素
        swap(nums[i], nums[j]);
    }
    
    // 4. i == j，此处就是base的最终位置，把基准归位
    swap(nums[left], nums[i]);
    
    // 5. 递归处理左右两个子区间
    quickSort(nums, left, i - 1);
    quickSort(nums, i + 1, right);
}

int main() {
    vector<int> nums = {3, 2, 5, 1, 4, 6, 2};
    
    quickSort(nums, 0, nums.size() - 1);
    
    for (int x : nums) cout << x << " ";
    // 输出: 1 2 2 3 4 5 6
    return 0;
}
```

---

### 时间复杂度

| 情况     | 复杂度         | 原因                                                         |
| :------- | :------------- | :----------------------------------------------------------- |
| **最好** | *O*(*n*log*n*) | 每次 pivot 正好选中位数，数组被**均匀平分**                  |
| **平均** | *O*(*n*log*n*) | 大部分情况下分割比较均匀，综合下来是这个级别                 |
| **最坏** | *O*(*n*2)      | 数组**已经有序**，且总是选最左/最右当 pivot，每次只能分出一个元素 |

#### 为什么最好情况是 *n*log*n*？

- 每一层分区都要遍历全部 *n*  个元素（一次 *O*(*n*) ）
- 均匀平分的情况下，递归树高度是 log*n* 
- 总工作量 = *n*×log*n*=*O*(*n*log*n*) 

### 空间复杂度

| 情况          | 复杂度      | 原因                           |
| :------------ | :---------- | :----------------------------- |
| **最好/平均** | *O*(log*n*) | 递归调用栈的深度，等于树的高度 |
| **最坏**      | *O*(*n*)    | 退化成链表，递归深度为 *n*     |



### 一句话背下来

> 平均 *O*(*n*log*n*)  时间 + *O*(log*n*)  空间；最坏退化成 *O*(*n*2)  时间和 *O*(*n*)  栈空间。

**怎么避免最坏情况？**

- 随机选择基准