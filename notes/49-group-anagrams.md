LeetCode 49 字母异位词分组 学习笔记
题目链接：https://leetcode.cn/problems/group-anagrams/
1. 题目分析题目描述给你一个字符串数组，请你将字母异位词组合在一起。可以按任意顺序返回结果列表。
字母异位词：由原单词字母重新排列得到，字母种类、每个字母出现次数完全相同，仅顺序不同。
示例plaintext输入：strs = ["eat","tea","tan","ate","nat","bat"]
输出：[["bat"],["nat","tan"],["ate","eat","tea"]]
提示
\(1 \le strs.length \le 10^4\)
\(0 \le strs[i].length \le 100\)
字符串全部由小写英文字母组成
核心问题怎么给每一组异位词生成一个统一标识，把相同标识的字符串归到同一组。2. 思路梳理
规律：互为字母异位词的字符串，把字符排序之后得到的字符串完全相同。

eat、tea、ate排序后都是aet。


使用哈希表unordered_map做分组：

key：字符串排序后的结果（分组标识）
value：vector<string>，存放属于该组的原始字符串


遍历输入数组，每个字符串生成排序后的 key，把原字符串 push 到 map 对应 key 的数组。
遍历哈希表，取出所有 value，组装成最终答案返回。

补充另一种思路：统计 26 个小写字母出现次数，把计数数组转为字符串作为 key，可以避开排序，时间复杂度可以优化为\(O(NK)\)，代码会稍复杂。
3. 原始代码（push_back 版本）cpp运行#include <vector>
#include <string>
#include <unordered_map>
#include <algorithm>

using namespace std;

class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string>& strs) {
        unordered_map<string,vector<string>> map;

        for(auto str:strs)
        {
            string key=str;
            sort(key.begin(),key.end());
            map[key].push_back(str);
        }

        vector<vector<string>> ans;
        for(auto pair:map)
        {
            ans.push_back(pair.second);
        }
        return ans;
    }
};
代码逐行解析
unordered_map<string,vector<string>> map：哈希表，key 是排序字符串，value 存储一组异位词。
循环遍历每一个字符串str。
string key = str; sort(key.begin(),key.end())复制字符串并排序，得到分组标识 key。
map[key].push_back(str)：C++unordered_map如果 key 不存在，会自动创建空vector<string>，再执行 push_back，把原始字符串加入分组。
遍历 map，把每一组 vector 放入 ans，返回结果。
4. emplace /emplace_back 版本代码cpp运行#include <vector>
#include <string>
#include <unordered_map>
#include <algorithm>

using namespace std;

class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string>& strs) {
        unordered_map<string,vector<string>> map;

        for(auto str:strs)
        {
            string key=str;
            sort(key.begin(),key.end());
            map[key].emplace_back(str);
        }

        vector<vector<string>> ans;
        for(auto& pair:map)
        {
            ans.emplace_back(pair.second);
        }
        return ans;
    }
};

注意：map.emplace()和vec.emplace_back()是两个完全不同函数。

vec.emplace_back(args)：vector 尾部就地构造元素
map.emplace(args)：向 map 插入键值对，参数直接是 pair 的构造参数

5. 涉及知识点与方法详解5.1 STL sortsort(begin,end)，对容器区间排序，默认升序，底层快速排序。字符串可以直接 sort。5.2 unordered_map 哈希表底层哈希表，平均 O (1) 查找插入；key 不存在访问map[key]会自动默认构造 value。
对比map底层红黑树，操作复杂度\(O(logN)\)。
5.3 push_back vs emplace_back（重点）push_back接收已经构造完成的对象，把对象拷贝 / 移动到容器内。cpp运行vec.push_back(obj);
//逻辑：obj已经存在，拷贝或移动一份放到容器末尾
emplace_back接收构造对象所需要的参数，直接在容器内存上原地构造对象，减少临时对象、拷贝移动开销。cpp运行vec.emplace_back(str);
两者对比






















表格函数参数行为优势场景push_back完整对象拷贝 / 移动已有对象传入现成对象，代码简单直观emplace_back对象构造参数容器内存直接构造，减少拷贝构造开销大的对象，传入构造参数，避免临时对象
当传入的是左值对象（例如本例子中的 str），emplace_back(str)和push_back(str)性能几乎一样，都会触发拷贝。
emplace 最大收益场景：直接传入构造参数，不提前构造对象：
cpp运行vec.emplace_back("hello"); //直接在容器构造string，没有临时变量
vec.push_back(string("hello")); //先构造临时string，再移动

5.4 map.emplace () 补充说明（不要和 emplace_back 混淆）unordered_map::emplace用来插入键值对，传入pair的构造参数：cpp运行//map.emplace(键,值);
map.emplace(key, vector<string>{str});

⚠️本题代码map[key].emplace_back(str)，这里调用的是 vector 的 emplace_back，不是 map.emplace，很多同学会混淆。
5.5 auto 遍历注意引用cpp运行for(auto pair:map)     //值拷贝，每一轮复制pair，性能差
for(auto& pair:map)    //引用，直接操作原数据，推荐
for(const auto& pair:map) //只读引用，最佳实践
6. 时间、空间复杂度分析设 N：输入字符串总个数；K：字符串最大长度力扣 (L...。排序哈希解法（本笔记代码）
时间复杂度：\(O(N \cdot K \log K)\)

循环 N 次，每个字符串排序耗时 \(O(K\log K)\)；哈希表插入平均\(O(1)\)。


空间复杂度：\(O(N\cdot K)\)

哈希表存储全部 N 个字符串；同时存储排序后的 key 字符串；输出结果数组 ans 也要存储全部字符串。



拓展：字符计数法，时间复杂度\(O(N\cdot K)\)，不需要排序；用 26 字母计数作为 key，适合字符串很长场景。
7. 易错点总结
不要直接 sort 原字符串 str，要复制一份做 key，原字符串要完整保存放入结果。
auto pair:map会拷贝，数据量大性能下降，优先使用auto &。
区分emplace_back(vector方法)和map.emplace(map插入方法)。
空字符串""、单字符"a"都是合法输入，当前代码天然兼容，不需要特殊处理。