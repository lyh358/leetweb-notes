```
class Solution {
public:
    // 哈希表：记录中序遍历中每个值对应的索引位置
    // 键：节点值，值：该值在中序数组中的下标
    unordered_map<int,int> findInorderIndex;
    
    /*
    递归构建二叉树
    @param preorder 前序遍历数组
    @param inorder  中序遍历数组
    @param pL       当前子树在前序中的左边界（preorder Left）
    @param pR       当前子树在前序中的右边界（preorder Right）
    @param iL       当前子树在中序中的左边界（inorder Left）
    @param iR       当前子树在中序中的右边界（inorder Right）
    @return         构建好的当前子树的根节点
    */
    TreeNode* build(vector<int>& preorder,vector<int>& inorder,int pL,int pR,int iL,int iR)
    {
        // 【剪枝】如果前序数组为空，直接返回空（其实可省略，因为 buildTree 里 n==0 不会进入递归）
        if(preorder.size()==0) return nullptr;
        // 【递归终止】区间不合法，说明当前子树为空
        if(pL>pR || iL> iR) return nullptr;
        // 1. 确定根节点：前序遍历的第一个元素就是当前子树的根
        int preRoot = preorder[pL];
        // 2. 在中序遍历中定位根节点的位置（O(1) 哈希查找）
        int inRootPos = findInorderIndex[preRoot];
        // 3. 创建根节点
        TreeNode* root = new TreeNode(preRoot);
        // 4. 计算左子树的节点个数
        int subTreeSize = inRootPos-iL;
        // 5. 递归构建左子树
        root->left = build(preorder,inorder,
                            pL+1,
                            pL+subTreeSize,
                            iL,
                            inRootPos-1);
        // 6. 递归构建右子树
        root->right = build(preorder,inorder,
                            pL+subTreeSize+1,
                            pR,
                            inRootPos+1,
                            iR);
        return root;
    }
    //入口函数
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) 
    {
        int n = preorder.size();
        // 预处理：建立中序值 → 索引的映射，后续 O(1) 查找根节点位置
        for(int i=0;i<n;i++)
        {
            findInorderIndex[inorder[i]] = i;
        }
        // 初始调用：整棵树对应前序[0, n-1] 和中序[0, n-1]
        return build(preorder,inorder,0,n-1,0,n-1);
    }   
};
```