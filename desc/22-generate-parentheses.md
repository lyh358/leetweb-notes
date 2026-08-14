### DFS 回溯，还剩几个括号就还能放几个。优先放左括号，但只要已放左括号不少于右括号就能放右括号（left <= right 时合法），左括号用完了就只能补右括号，左右都用完收集答案。
```
class Solution {
public:
    vector<string> ans;
    string str="";

    void dfs(string& str,int left ,int right)
    {
        if(left>right)
        {
            return;
        }
        if(left==0 && right==0)
        {
            ans.push_back(str);
            return;
        }
        if (left > 0)   // ← 有剩余左括号才放
        {              
            str.push_back('(');
            dfs(str, left - 1, right);
            str.pop_back();
        }
        
        if (right > 0)   // ← 有剩余右括号才放
        {            
            str.push_back(')');
            dfs(str, left, right - 1);
            str.pop_back();
        }
    }

    vector<string> generateParenthesis(int n) {
        dfs(str,n,n);
        return ans;
    }
};
```