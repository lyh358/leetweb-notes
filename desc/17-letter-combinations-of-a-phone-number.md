### 
```
class Solution {
//排列式回溯问题
vector<string> combinations;
string combination;
unordered_map<char, string> phoneMap{
            {'2', "abc"},
            {'3', "def"},
            {'4', "ghi"},
            {'5', "jkl"},
            {'6', "mno"},
            {'7', "pqrs"},
            {'8', "tuv"},
            {'9', "wxyz"}
        };
public:
    vector<string> letterCombinations(string digits) {
        if(digits.empty()) return combinations;
        backtracing(digits,0);
        return combinations;
    }

private:
    void backtracing(string digits,int start)
    {
        if(start==digits.length())
        {
            combinations.push_back(combination);
            return;
        }
        string digit=phoneMap[digits[start]];
        for(int i=0;i<digit.length();i++)
        {
            combination.push_back(digit[i]);
            backtracing(digits,start+1);
            combination.pop_back();
        }
    }
};
```