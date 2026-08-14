### 把二维矩阵当成一维有序数组做二分查找，mid 通过 mid/n 得行号、mid%n 得列号映射回二维坐标，找到返回 true，找不到返回 false。

```
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        int m = matrix.size();
        int n = matrix[0].size();
        if(m==0||n==0) return false;
        int left = 0;
        int right = m*n-1;

        while(left<=right)
        {
            int mid = left+(right-left)/2;
            int value = matrix[mid/n][mid%n];

            if(value<target)
            {
                left=mid+1;
            }
            else if(value>target)
            {
                right=mid-1;
            }
            else{
                return true;
            }
        }
        return false;
    }
};
```