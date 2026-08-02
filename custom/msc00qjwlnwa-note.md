# 三叉搜索树高度

## 方法：左中右三向递归+返回最大值

- ## 构建三叉搜索树

```c++
struce TreeNode
{
    int val;
    TreeNode* left;
    TreeNode* mid;
    TreeNode* right;
    
    //构造函数
    TreeNode(int n):val(n),left(nullptr),right(nullptr),mid(nullptr){}
};
```

- ## 三叉搜索树求高度函数（递归）

```c++
int TreeHeight(TreeNode* root)
{
    if(!root)
    {
        return 0;//空树高度为0
    }
    //高度定义:从根到最远叶子节点的边数 + 1
    int leftH=TreeHeight(root->left);//左高度向左递归
    int midH=TreeHeight(root->mid);//中高度向中递归
    int rightH=TreeHeight(root->right);//右高度向右递归
    return max({leftH,midH,rightH})+1;//得到的是边数，还得+1
}
```

- ### 结果要加一：因为高度=最大边数量+1

- ### max（{a,b,c}）:大括号用于多个数进行比较