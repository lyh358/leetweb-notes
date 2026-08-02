

# 核心思路：双栈法

- ## 主栈用于实现栈的基本功能`stack<int> x_stack;`

- ## 辅助栈用于维护当前状态的最小值`stack<int> min_stack;`

----

# 实现流程

## （1）初始化所需的基本数据结构

- 创建主栈
- 创建辅助栈

```c++
stack<int> x_stack;
stack<int> min_stack;
```

## （2）初始化栈对象

- ### 主要是初始化最小栈的初始值

  - #### 压入最大整数`INT_MAX`，用于基准对比

```c++
MinStack()
{
    min_stack.push(INT_MAX);
}
```

## (3)实现入栈功能

- ### 主栈正常入栈当前值

- ### 辅助栈入栈当前状态的最小值

  - #### 上一状态最小值`min_stack.top()` 和 当前值`x` 进行对比决定谁入栈

```c++
void push(int x)
{
    x_stack.push(x);
    min_stack.push(min(min_stack.top(),x));
}
```

## (4)实现出栈功能

- ### 因为栈是FILO的

- ### 主栈和辅助栈同步弹出

```c++
void pop()
{
	x_stack.pop();
	min_stack.pop();
}
```

## （5）实现获取栈顶元素功能

- ### 主栈直接`.top()`

```c++
int top() 
{
	return x_stack.top();
}
```

## (6)实现获取最小值功能

- ### 辅助栈直接`.top()`

```c++
int getMin() 
{
    return min_stack.top();
}
```

---

# 完整代码

```c++
class MinStack {
public:
    stack<int> x_stack;
    stack<int> min_stack;
    
    //构造函数：初始化辅助栈
    MinStack() {
        min_stack.push(INT_MAX);
    }
    
    void push(int val) {
        x_stack.push(val);
        min_stack.push(min(min_stack.top(),val));
    }
    
    void pop() {
        x_stack.pop();
        min_stack.pop();
    }
    
    //要有返回值
    int top() {
        return x_stack.top();
    }
    //要有返回值
    int getMin() {
        return min_stack.top();
    }
};
```

