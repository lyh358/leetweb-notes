# 11.盛水最多的容器

给定一个长度为 `n` 的整数数组 `height` 。有 `n` 条垂线，第 `i` 条线的两个端点是 `(i, 0)` 和 `(i, height[i])` 。

找出其中的两条线，使得它们与 `x` 轴共同构成的容器可以容纳最多的水。

返回容器可以储存的最大水量。

**说明：**你不能倾斜容器。

**示例 1：**

![img](https://aliyun-lc-upload.oss-cn-hangzhou.aliyuncs.com/aliyun-lc-upload/uploads/2018/07/25/question_11.jpg)

```
输入：[1,8,6,2,5,4,8,3,7]
输出：49 
解释：图中垂直线代表输入数组 [1,8,6,2,5,4,8,3,7]。在此情况下，容器能够容纳水（表示为蓝色部分）的最大值为 49。
```

**示例 2：**

```
输入：height = [1,1]
输出：1
```

**提示：**

- `n == height.length`
- `2 <= n <= 105`
- `0 <= height[i] <= 104`

----

# 核心思路：贪心+反向双指针

- ### 维护一个最大值`res`

- ### 定义左指针`i = 0`,右指针`j = height.size()-1`

- ### 当左右指针不重叠`while(i<j)`

  - #### 计算当前的面积`int area = (j-i)*min(height[i],height[j]);`

  - #### 维护最大面积`res = max(area,res);`

- ### 左右边界哪边更低，哪边就收缩

  - #### 因为收缩后宽度肯定持续降低，且新容器的短板《=旧容器的长板

- ```c++
  if(height[i]>=height[j]) 
              {
                  j--;
              }
              else
              {
                  i++;
              }
  ```

---

# 完整代码

```c++
class Solution {
public:
    int maxArea(vector<int>& height) 
    {
        int res=0;
        int i = 0;
        int j=height.size()-1;

        while(i<j)
        {
            int area = (j-i)*min(height[i],height[j]);
            res = max(area,res);
            if(height[i]>=height[j]) 
            {
                j--;
            }
            else
            {
                i++;
            }
        }
        return res;
    }
};
```

