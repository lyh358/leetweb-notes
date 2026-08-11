```
/*
// Definition for a Node.
class Node {
public:
    int val;
    Node* next;
    Node* random;
    
    Node(int _val) {
        val = _val;
        next = NULL;
        random = NULL;
    }
};
*/

class Solution {
public:
    Node* copyRandomList(Node* head) {
        //key：旧链表的结点，value：新链表的节点
        unordered_map<Node*,Node*> map;
        Node* cur=head;

        while(cur!=nullptr)
        {
            map[cur]=new Node(cur->val);
            cur=cur->next;
        }

        cur=head;
        while(cur!=nullptr)
        {
            map[cur]->next=map[cur->next];
            map[cur]->random=map[cur->random];
            cur=cur->next;
        }
        return map[head];
    }
};
```