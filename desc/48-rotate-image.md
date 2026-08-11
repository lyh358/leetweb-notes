# 顺时针90°
### 先主对角线转置，再行翻转
```
class Solution {
public:
    void rotate(vector<vector<int>>& matrix) {
        int n=matrix.size();

        for(int i=0;i<n;i++)
        {
            for(int j=i;j<n;j++)
            {
                swap(matrix[i][j],matrix[j][i]);
            }
        }
        for(int i=0;i<n;i++)
        {
            reverse(matrix[i].begin(),matrix[i].end());
        }
    }
};
```

# 逆时针90°
### 先行翻转主对角线转置，再
```