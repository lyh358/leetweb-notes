```
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if(!root || root==p || root==q) return root;
        TreeNode* left = lowestCommonAncestor(root->left,p,q);
        TreeNode* right = lowestCommonAncestor(root->right,p,q);

        if(left && right) return root;
        else if(left) return left;
        else if(right) return right;
        else 
        {
            return nullptr;
        }   
    }
};
```
## 从root往下搜索p和q：一共三种情况
- 空树：返回本身null（）