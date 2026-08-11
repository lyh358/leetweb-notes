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
### `ls.emplace_front(key,value)`;可以把emplace换成更常用的方法吗?
可以的，你可以使用更常见的 push_front 方法。

替换方法：

ls.push_front({key, value});
// 或者
ls.push_front(std::make_pair(key, value));
应用代码
简单说明：

push_front 会先构造好 pair 对象，然后再将其拷贝或移动到链表头部，写法更直观，也是大家最常用的方式。
emplace_front 则是直接在链表头部原地构造对象，省去了拷贝/移动的开销，性能上会有微小优势。
在 LeetCode 中，两者的性能差异通常可以忽略不计，使用 push_front 完全没问题！