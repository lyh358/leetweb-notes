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
### 先行翻转，再主对角线转置
```
void rotate270(vector<vector<int>>& matrix) {
    int n = matrix.size();
    // 1. 每行反转
    for(int i = 0; i < n; i++) {
        reverse(matrix[i].begin(), matrix[i].end());
    }
    // 2. 主对角线转置
    for(int i = 0; i < n; i++) {
        for(int j = i; j < n; j++) {
            swap(matrix[i][j], matrix[j][i]);
        }
    }
}
```

# 180°
### 上下翻转 + 左右翻转（顺序无关）
```
void rotate180(vector<vector<int>>& matrix) {
    int n = matrix.size();
    // 1. 上下翻转
    for(int i = 0; i < n / 2; i++) {
        swap(matrix[i], matrix[n - 1 - i]);
    }
    // 2. 左右翻转（每行反转）
    for(int i = 0; i < n; i++) {
        reverse(matrix[i].begin(), matrix[i].end());
    }
}
```