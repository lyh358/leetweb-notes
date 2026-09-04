# 394.字符串解码

# 题目描述

给定一个经过编码的字符串，返回它解码后的字符串。

编码规则为: `k[encoded_string]`，表示其中方括号内部的 `encoded_string` 正好重复 `k` 次。注意 `k` 保证为正整数。

你可以认为输入字符串总是有效的；输入字符串中没有额外的空格，且输入的方括号总是符合格式要求的。

此外，你可以认为原始数据不包含数字，所有的数字只表示重复的次数 `k` ，例如不会出现像 `3a` 或 `2[4]` 的输入。

测试用例保证输出的长度不会超过 `105`。

**示例 1：**

```bash
输入：s = "3[a]2[bc]"
输出："aaabcbc"
```

**示例 2：**

```bash
输入：s = "3[a2[c]]"
输出："accaccacc"
```

**示例 3：**

```bash
输入：s = "2[abc]3[cd]ef"
输出："abcabccdcdcdef"
```

**示例 4：**

```bash
输入：s = "abc3[cd]xyz"
输出："abccdcdcdxyz"
```

**提示：**

- `1 <= s.length <= 30`
- `s` 由小写英文字母、数字和方括号 `'[]'` 组成
- `s` 保证是一个 **有效** 的输入。
- `s` 中所有整数的取值范围为 `[1, 300]`

---

## 数字栈+字符串栈：`tempNum`累积多位数，`tempStr`累积当前层字符串，`[`压栈重置，`]`出栈拼接

```cpp
class Solution {
public:
    string decodeString(string s) {
        //做两个stack从左到右存数字和字符串
        stack<int> nums;
        stack<string> strs;
        //做两个临时变量用于存stack前组装任意长度的数字和字符串
        int tempNum = 0;
        string tempStr ="";
        //结果数组

        //for循环编列解码
        for(int i = 0;i<s.size();i++)
        {
            //四种情况
            //遇到数字（但是是字符）
            if(s[i]>='0' && s[i]<='9')
            {
                tempNum = tempNum*10 + s[i] - '0';
            }
            //2.遇到字母
            else if((s[i]>='a' && s[i]<='z' )||(s[i]>='A' && s[i]<='Z'))
            {
                tempStr += s[i];
            }
            //3.遇到左‘[’  入栈并重置缓存
            else if(s[i] == '[')
            {
                strs.push(tempStr);
                nums.push(tempNum);
                tempNum = 0;
                tempStr = "";
            }
            //4.遇到右括号,进行解码，此时最内层的字符串还没入stack，直接获取stack中的次数，对栈顶的字符串加times次，然后把它输出为当前字符串并出栈，就算解完一层码了
            else
            {
                int times = nums.top();
                nums.pop();

                for(int j=0;j<times;j++)
                {
                    strs.top()+=tempStr;
                }
                tempStr = strs.top();
                strs.pop();
            }
        }
        return tempStr;
    }
};
```
