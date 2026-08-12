```
class Solution {
public:
    unordered_map<int,int> findInorderIndex;
    
    TreeNode* build(vector<int>& preorder,vector<int>& inorder,int pL,int pR,int iL,int iR)
    {
        if(preorder.size()==0) return nullptr;
        if(pL>pR || iL> iR) return nullptr;

        int preRoot = preorder[pL];
        int inRootPos = findInorderIndex[preRoot];

        TreeNode* root = new TreeNode(preRoot);

        int subTreeSize = inRootPos-iL;
        root->left = build(preorder,inorder,
                            pL+1,
                            pL+subTreeSize,
                            iL,
                            inRootPos-1);
        root->right = build(preorder,inorder,
                            pL+subTreeSize+1,
                            pR,
                            inRootPos+1,
                            iR);

        return root;
    }
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) 
    {
        int n = preorder.size();
        for(int i=0;i<n;i++)
        {
            findInorderIndex[inorder[i]] = i;
        }
        return build(preorder,inorder,0,n-1,0,n-1);
    }   
};
```