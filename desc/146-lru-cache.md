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
### `ls.emplace_front(key,value)`;可以把`emplace`换成更常用的方法吗?
可以的，你可以使用更常见的` push_front `方法。

替换方法：

`ls.push_front({key, value});`
或者
`ls.push_front(std::make_pair(key, value));`

**简单说明：**
push_front 会先构造好 pair 对象，然后再将其拷贝或移动到链表头部，写法更直观，也是大家最常用的方式。
emplace_front 则是直接在链表头部原地构造对象，省去了拷贝/移动的开销，性能上会有微小优势。
在 LeetCode 中，两者的性能差异通常可以忽略不计，使用 push_front 完全没问题！

### `ls.emplace_front(key,value);`为什么这里面不用加大括号?
原理解释：

**完美转发：**`emplace_front` 是一个可变参数模板函数，它会将你传入的参数（`key` 和 `valu`e）直接转发给链表元素类型（即 `std::pair<int, int>`）的构造函数。
`Pair` 的构造函数：`std::pair` 本身提供了一个接受两个参数的构造函数 `pair(const T1& x, const T2& y)`。因此，直接传入 `key` 和 `value`，它就能在链表头部直接构造出这个 `pair`。
**如果加了大括号：**
如果你写成 `ls.emplace_front({key, value});`，实际上是传入了一个初始化列表。虽然编译器也能处理，但这违背了` emplace` 系列函数“避免构造临时对象”的初衷，多了一层不必要的转换。

**总结：**
`push_front({key, value})`：需要先构造一个 `pair` 临时对象，再放入链表。
`emplace_front(key, value)`：直接把参数交给 `pair` 的构造函数，在链表节点内存中原地构造，效率更高且语法更简洁。