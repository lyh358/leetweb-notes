

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
### 左值引用问题
因为你调用时传了 ""：
`backtracing(0, s, "");`
"" 是字符串字面量，编译器会生成一个临时的 string 对象（右值）。C++ 规定：临时对象不能绑定到非 const 的左值引用。
如果你写成：
`void backtracing(int i, string& s, string& sub)  // 去掉 const`
编译会报错：
```
cannot bind non-const lvalue reference of type 'std::string&' to an rvalue of type 'std::string'
（不能将非 const 左值引用绑定到右值）
```
#### 两个参数都要加 const 的原因
**为什么加 const**
const string& s	原始字符串是输入，整个回溯过程只读不改；加 const 防误改
const string& sub	调用时传了 "" 临时量，必须加 const 才能绑定；而且 sub 在当前层也是只读（修改是通过 temp = sub + s[i] 生成新变量，不是改 sub 本身）
**一句话**
const 既保护数据不被意外修改，又让临时对象（如 ""）能绑定到引用上。 去掉 const，临时量传不进来，编译直接报错。