

## 核心数据结构：栈

---



## 三种不匹配情况：

### 1.左括号多余（右括号不够，循环结束栈不为空）

### 2.括号不多于但是不匹配（直接在循环中return false）

### 3.右括号多余（循环尚未结束，栈就空了）

## 一种匹配情况：

- ### 循环中没发生不匹配

- ### 循环结束栈刚好空

---

## 核心思路：

1.进行预筛选：奇数串肯定不满足条件

```c++
if(s.size() %2 != 0)
{
	return false;
}
```

2.创建一个栈

```c++
stack<char> st;
```

3.进行for循环

```c++
    for(int i = 0; i<s.size(); i++)
    {
    //1. 如果遇到左括号，压入对应的右括号（打欠条）
        if(s[i]=='(')
        {
            st.push(')');
        }
        else if(s[i]=='[')
        {
            st.push(']');
        }
	    else if(s[i]=='{')
        {
            st.push('}');
        }
        //2. 如果没有左括号，或者压完左括号该遇到右括号时不匹配，则返回false
        else if(st.empty() ||st.top() != s[i])//注意：得先判断栈是否为空才能取栈顶
        {
            return false;
        }
        //3. 如果不是上述情况，即左括号欠下的右括号与真正的右括号能对应，直接消掉欠条
        else
        {
            st.pop();
        }
    }
    //经历完循环，判断有没有多余的左括号没被匹配
    return st.empty();
```

## 完整代码（leetcode）

```c++
class Solution {
public:
    bool isValid(string s)
    {
    if(s.size()%2!=0)return false;
    
    stack<char> st;
    
    for(int i = 0; i<s.size(); i++)
    {
    //1. 如果遇到左括号，压入对应的右括号（打欠条）
        if(s[i]=='(')
        {
            st.push(')');
        }
        else if(s[i]=='[')
        {
            st.push(']');
        }
	    else if(s[i]=='{')
        {
            st.push('}');
        }
        //2. 如果没有左括号，或者压完左括号该遇到右括号时不匹配，则返回false
        else if(st.empty() ||st.top() != s[i])
        {
            return false;
        }
        //3. 如果不是上述情况，即左括号欠下的右括号与真正的右括号能对应，直接消掉欠条
        else
        {
            st.pop();
        }
    }
    //经历完循环，判断有没有多余的左括号没被匹配
    return st.empty();
    }
};
```

