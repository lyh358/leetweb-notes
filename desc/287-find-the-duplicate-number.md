

```
// 第一阶段：找相遇点
int slow = nums[0];           // slow 先走一步到 nums[0]
int fast = nums[nums[0]];     // fast 先走两步

while (slow != fast) {
    slow = nums[slow];        // 慢：一步
    fast = nums[nums[fast]];  // 快：两步
}

// 第二阶段：找入口
int ptr = 0;                  // ptr 从起点（索引0）出发
while (ptr != slow) {
    ptr = nums[ptr];          // 各走一步
    slow = nums[slow];
}
return ptr;                   // 相遇点就是重复数
```