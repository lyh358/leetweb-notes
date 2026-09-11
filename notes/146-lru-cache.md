# 最近最少使用缓存：一个类，两个核心数据结构，三个成员函数

# 两个核心数据结构：

## 一个双向链表list，里面存的元素是pair<int,int>
一个哈希map，key为int，value为刚才的list的迭代器
还有一个int型容量capacity

# 三个成员函数：

## 类同名构造函数LRUCache(int capacity)
get获取值
put存入值
