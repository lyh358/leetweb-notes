力扣 560：和为 K 的子数组 — 学习笔记
一、题目概述
给定：整数数组 nums 和整数 k
求：数组中连续子数组的个数，使得子数组内所有元素之和等于 k
关键难点：数组中含有负数，子数组和不是单调递增的，滑动窗口失效。
二、核心思路：前缀和 + 哈希表
2.1 前缀和定义
prefix[j] = 从索引 0 到 j 的所有元素之和
子数组 [i, j] 的和可以表示为：
plain
sum(i, j) = prefix[j] - prefix[i-1]
2.2 目标转化
我们要找满足 sum(i, j) = k 的所有 (i, j)，即：
plain
prefix[j] - prefix[i-1] = k
=> prefix[i-1] = prefix[j] - k
核心洞察：遍历到位置 j 时，如果前面有 m 个位置的前缀和等于 prefix[j] - k，那就说明有 m 个子数组以 j 结尾、和为 k。
三、代码逐行解析
cpp
class Solution {
public:
    int subarraySum(vector<int>& nums, int k) {
        unordered_map<int,int> countPrefix;  // <前缀和, 出现次数>
        countPrefix[0] = 1;                  //  crucial！处理从索引0开始的子数组
        
        int ans = 0;      // 答案计数
        int sum = 0;      // 当前前缀和（即 prefix[j]）

        for(auto num : nums)
        {
            sum += num;   // 更新当前前缀和

            // 查前面有多少个前缀和等于 sum - k
            // 这些位置都能和当前位置构成和为 k 的子数组
            ans += countPrefix[sum - k];
            
            // 把当前前缀和加入哈希表，供后面的位置查询使用
            countPrefix[sum]++;
        }
        return ans;
    }
};
表格
代码行	作用
unordered_map<int,int> countPrefix	哈希表，存储"历史前缀和 → 出现次数"
countPrefix[0] = 1	初始化 crucial：前缀和为 0 出现了 1 次（空前缀），否则漏掉从索引 0 开始的子数组
sum += num	维护当前前缀和
ans += countPrefix[sum - k]	核心操作：找前面有几个 prefix = sum - k，就有几个合法子数组以当前位置结尾
countPrefix[sum]++	把当前前缀和"存档"，供后续位置查询
四、推演示例
nums = [1, 2, 3], k = 3
表格
步骤	num	sum	sum-k	countPrefix[sum-k]	ans	countPrefix 存档后
初始	-	0	-	-	0	{0: 1}
1	1	1	-2	0	0	{0:1, 1:1}
2	2	3	0	1	1	{0:1, 1:1, 3:1}
3	3	6	3	1	2	{0:1, 1:1, 3:1, 6:1}
答案 = 2，对应子数组：[1,2](0~1)、[3](2~2)
五、两个最容易错的点
⚠️ 1. 必须初始化 countPrefix[0] = 1
如果不加，当某个前缀和刚好等于 k 时（子数组从索引 0 开始），sum - k = 0 在 map 里找不到，会漏解。
例：nums = [1, 2], k = 1，第一步 sum = 1，需要 map 里有 {0:1} 才能识别出子数组 [1]。
⚠️ 2. 顺序：先查表，再更新
cpp
ans += countPrefix[sum - k];   // ✅ 先查"历史"
countPrefix[sum]++;            // ✅ 再记录"当前"
如果反过来，当前前缀和会被当成"之前"的用，导致把当前位置和自己配对，重复计数。
六、复杂度分析
表格
维度	复杂度	说明
时间	O(n)	一次遍历，哈希操作均摊 O(1)
空间	O(n)	最坏情况下所有前缀和都不同
七、一句话记忆口诀
当前前缀和减 k，查查前面出现过几次，就是有几个子数组以当前位置结尾和为 k。
哈希表存历史，初始塞 {0:1} 防漏头；先查后存，顺序别搞混。