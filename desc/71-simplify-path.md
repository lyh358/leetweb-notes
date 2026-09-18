```
#include <vector>
#include <string>
#include <sstream>
using namespace std;

class Solution {
public:
    string simplifyPath(string path) {
        vector<string> st; // vector模拟栈
        stringstream ss(path);
        string part;

        // 按 '/' 分割字符串
        while(getline(ss, part, '/'))
        {
            if(part == "" || part == ".")
            {
                continue;
            }
            else if(part == "..")
            {
                if(!st.empty())
                {
                    st.pop_back();
                }
            }
            else
            {
                st.push_back(part);
            }
        }

        // 拼接答案
        string ans = "/";
        for(int i=0; i<st.size(); i++)
        {
            if(i>0) ans += "/";
            ans += st[i];
        }
        return ans;
    }
};
```
