```
class Solution {
    
public:
    unordered_map<int, int> index;
    TreeNode* build(vector<int>& preorder,vector<int>& inorder,int preorder_left,int preorder_right,int inorder_left,int inorder_right)
    {
        if(preorder_left>preorder_right || inorder_left>inorder_right)
        {
            return nullptr;
        }
        int inorder_root = index[preorder[preorder_left]];
        TreeNode* root = new TreeNode(inorder[inorder_root]);
        int leftsize = inorder_root-inorder_left;
        root->left = build(preorder,inorder,preorder_left+1,inorder_root-inorder_left+preorder_left,inorder_left,inorder_root-1);
        root->right = build(preorder,inorder,inorder_root-inorder_left+preorder_left+1,preorder_right,inorder_root+1,inorder_right);
        return root;
    }
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        
        for(int i=0;i<preorder.size();i++)
        {
            index[inorder[i]]=i;
        }
        return build(preorder,inorder,0,preorder.size()-1,0,preorder.size()-1);
    }
};