题目：给 k 个**升序链表**，把它们合并成一个升序链表，返回头节点。### 方法 1：优先队列（小根堆，最直观好写）

思路核心：

1. 每个链表的头节点放入**小根堆**，堆按照节点 val 从小到大排序。
2. 每次取出堆里最小的节点，接到结果链表后面。
3. 如果取出的这个节点还有下一个节点（`node->next != nullptr`），就把它的后继节点入堆。
4. 重复直到堆为空。

> 时间复杂度：\(O(N\log k)\)，N 是总节点数，k 是链表个数。堆每次操作\(\log k\)。
> 空间：\(O(k)\)，堆最多存 k 个节点。

---

## 伪代码【优先队列版本】

```
// 链表节点定义
ListNode {
    int val
    ListNode next
}

function mergeKLists(lists):
    // 小根堆：存ListNode*，按val从小到大
    minHeap = 优先队列(比较规则：a.val < b.val)
    
    // 先把所有非空链表头入堆
    for each head in lists:
        if head != null:
            minHeap.push(head)
    
    // 虚拟头结点，方便构建结果链表
    dummy = ListNode(0)
    cur = dummy
    
    while 堆不为空:
        // 取出最小值节点
        topNode = minHeap.pop()
        cur.next = topNode
        cur = cur.next
        
        // 该链表还有后续节点，继续入堆
        if topNode.next != null:
            minHeap.push(topNode.next)
    
    return dummy.next
```
