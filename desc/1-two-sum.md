class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> hash_map;  // 值 -> 下标

        for (int i = 0; i < nums.size(); i++) {
            int complement = target - nums[i];
            if (hash_map.count(complement)) {
                // 找到 complement，且它是在当前元素之前出现的
                return {hash_map[complement], i};
            }
            // 将当前元素存入哈希表，供后面的元素查找
            hash_map[nums[i]] = i;
        }

        return {};
    }
};