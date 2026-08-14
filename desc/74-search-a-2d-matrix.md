```
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        int m=matrix.size();
        int n=matrix[0].size();
        if (m==0 || n==0) return false;

        int left=0;
        int right=m*n-1;

        while(left<=right)
        {
            int mid=left+(right-left)/2;
            int value_mid=matrix[mid/n][mid%n];
            if(target>value_mid)
            {
                left=mid+1;
            }
            else if(target<value_mid)
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