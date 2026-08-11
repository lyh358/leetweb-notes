```
class LRUCache {
public:
    int cap;
    list<pair<int,int>> ls;
    unordered_map<int,list<pair<int, int>>::iterator> map;


    LRUCache(int capacity) {
        cap=capacity;
    }
    
    int get(int key) {
        auto it=map.find(key);
        if(it==map.end())
        {
            return -1;
        }
        ls.splice(ls.begin(),ls,it->second);
        return it->second->second;
    }
    
    void put(int key,int value) {
        auto it=map.find(key);
        if(it!=map.end())
        {
            it->second->second=value;
            ls.splice(ls.begin(),ls,it->second);
        }
        else
        {
            if(map.size()==cap)
            {
                map.erase(ls.back().first);
                ls.pop_back();
            }
            ls.emplace_front(key,value);
            map[key]=ls.begin();
        }
        
    }
};

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache* obj = new LRUCache(capacity);
 * int param_1 = obj->get(key);
 * obj->put(key,value);
 */
```
###