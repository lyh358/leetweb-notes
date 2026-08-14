### 
```
class Solution {
public:
    vector<string> ans;
    string combination;
    unordered_map<char,string> map={
        {'2',"abc"},
        {'3',"def"},
        {'4',"ghi"},
        {'5',"jkl"},
        {'6',"mno"},
        {'7',"pqrs"},
        {'8',"tuv"},
        {'9',"wxyz"},
    };
    void dfs(string& digits,int digNum)
    {
        if(digNum==digits.size())
        {
            ans.push_back(combination);
            return;
        }
        string digitStr = map[digits[digNum]];
        for(int i=0;i<digitStr.size();i++)
        {
            combination.push_back(digitStr[i]);
            dfs(digits,digNum+1);
            combination.pop_back();
        }
    }
    vector<string> letterCombinations(string digits) {
        dfs(digits,0);
        return ans;
    }
};
```