
```
class Solution {
public:
    vector<string> combination;
    vector<vector<string>> ans;

    bool isParall(string& str)
    {
        int left=0;
        int right=str.size()-1;
        while(left<right)
        {
            if(str[left++]!=str[right--])
            {
                return false;
            }
        }
        return true;
    }

    void dfs(string& s,const string& substr,int i)
    {
        if(i==s.size())
        {
            ans.push_back(combination);
            return;
        }

        string temp = substr+s[i];

        if(i!=s.size()-1)
        {
            dfs(s,temp,i+1);
        }

        if(isParall(temp))
        {
            combination.push_back(temp);
            dfs(s,"",i+1);
            combination.pop_back();
        }
    }
    
    vector<vector<string>> partition(string s) {
        dfs(s,"",0);
        return ans;
    }
};
```