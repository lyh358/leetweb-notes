# 提取字符串中的合法MAC地址

MAC 地址格式：`XX:XX:XX:XX:XX:XX` 或 `XX-XX-XX-XX-XX-XX`

- 6 组，每组 2 个十六进制字符（`0-9`, `A-F`, `a-f`）
- 分隔符为 `:` 或 `-`

---

## 手动模拟解法：

```c++
#include <bits/stdc++.h>
using namespace std;

bool isHex(char c) {
    return (c >= '0' && c <= '9') || (c >= 'A' && c <= 'F') || (c >= 'a' && c <= 'f');
}

// 检查从 pos 开始是否是合法 MAC
bool checkMAC(const string& s, int pos) {
    // 需要至少 17 个字符：XX:XX:XX:XX:XX:XX
    if (pos + 17 > s.size()) return false;
    
    char sep = s[pos + 2];  // 分隔符 : 或 -
    if (sep != ':' && sep != '-') return false;
    
    for (int i = 0; i < 6; i++) {
        int idx = pos + i * 3;
        // 每组第一个是十六进制字符
        if (!isHex(s[idx])) return false;
        // 每组第二个是十六进制字符
        if (!isHex(s[idx + 1])) return false;
        // 前5组后面必须跟分隔符，第6组后面不用
        if (i < 5 && s[idx + 2] != sep) return false;
    }
    return true;
}

vector<string> extractMAC(const string& s) {
    vector<string> res;
    for (int i = 0; i + 16 < s.size(); i++) {
        if (checkMAC(s, i)) {
            res.push_back(s.substr(i, 17));  // MAC 长度固定 17
            i += 16;  // 跳过已匹配的，避免重叠（可选）
        }
    }
    return res;
}

int main() {
    string text = "IPs: 192.168.1.1, MAC: 00:1A:2B:3C:4D:5E, 00-1A-2B-3C-4D-5F";
    auto macs = extractMAC(text);
    for (const auto& mac : macs) {
        cout << mac << endl;
    }
    return 0;
}
```

# 三个函数：

## 1.字符合法性检查函数

- ### 检查当前字符是不是（数字）（小写字母）（大写字母）

```c++
bool isHex(char c)
{
    
    return (c>='0'&&c<='9')||(c>='a'&&c<='z')||(c>='A'&&c<='Z');
   
}
```

## 2.从特定位置`pos`开始检查后面是否是一个连续的合法MAC地址

```c++
bool checkMAC(const string& s,int pos)
{
    //1.先检查后面的字母是否够长(>17个）
    if(pos+17>s.size()) return false;
    
    //2.检查格式合法性：2个合法字符+1个合法分隔符(:)(-)
    //2.1分隔符检查
    char sep = s[pos+2];//定义分隔符格式，校验通过了才能进入后续判断，也代表sep是合法分隔符了
    if(sep!=':'&&sep!='-') return false;//允许混用分隔符
    
    //循环校验：三个一组，检查6组，是否符合格式要求
    for(int i=0;i<6;i++)
    {
        int idx=pos+i*3;//每组的下标（索引）
        //校验逻辑
        if(!isHex(s[idx])) return false;
        if(!isHex(s[idx+1])) return false;
        if(i<5 && s[idx+2]!=sep) return false;//第六组后面不需要跟分隔符
    }
    
    //检查结束，通过
    return true;
    
}
```

## 3.从字符串中提取合法MAC地址的函数

```c++
vector<string> extractMAC(const string& s)
{
    vector<string> res;
    //如果长度够长，则从i=0开始判断后续字串
    for(int i=0;i+16<s.size();i++)
    {
        //如果检查通过，使用substr得到子字符串
        if(checkMAC(s,i))
        {
            res.push_back(s.substr(i,17));// MAC 长度固定 17
            i+=16;// 跳过已匹配的，避免重叠（可选）
        }
    }
}

```

