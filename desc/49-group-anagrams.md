```
#include <vector>
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
          for(auto [key,value]:map)
        {
            ans.push_back(map[key]);
        }
        vector<vector<string>> ans;
        for(auto pair:map)
        {
            ans.push_back(pair.second);
        }
        return ans;
    }
};
```
