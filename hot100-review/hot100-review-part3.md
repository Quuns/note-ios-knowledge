# 二叉树的中序遍历

> 递归

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    void dfs(TreeNode *x, vector<int> &ans) {
        if (!x) {
            return ;
        }
        if (x->left) {
            dfs(x->left,ans);
        }
        ans.push_back(x->val);
        if (x->right) {
            dfs(x->right,ans);
        }
    }
    vector<int> inorderTraversal(TreeNode* root) {
        vector<int>ans;
        dfs(root,ans);
        return ans;
    }
};
```

# 二叉树的最大深度

> 递归

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int dfs(TreeNode *x, int dep) {
        int ans = 0;
        if (x && !x->left && !x->right) return dep;
        if (x->left)ans = max(ans, dfs(x->left,dep+1));
        if (x->right)ans = max(ans, dfs(x->right,dep+1));
        return ans;
    }
    int maxDepth(TreeNode* root) {
        if (!root) return 0;
        else return dfs(root,1);
    }
};
```

# 翻转二叉树

> 递归

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    TreeNode* dfs(TreeNode *x) {
        if (x->left)dfs(x->left);
        if (x->right)dfs(x->right);
        swap(x->left,x->right);
        return x;
    }
    TreeNode* invertTree(TreeNode* root) {
        if (!root)return root;
        return dfs(root);
    }
};
```

# 对称二叉树

> 递归

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    bool dfs(TreeNode* x, TreeNode* y) {
        if (!x || !y) return x == y;
        return x->val == y->val && dfs(x->left,y->right) && dfs(x->right,y->left);
    }
    bool isSymmetric(TreeNode* root) {
        if (!root)return true;
        return dfs(root->left,root->right);
    }
};
```

# 二叉树的直径

> 

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int dfs(TreeNode *root, int &ans) {
        if (!root) {
            return 0;
        }
        int left_res = 0;
        int right_res = 0;
        if (root->left) {
            left_res = dfs(root->left, ans) + 1;
        } 
        if (root->right) {
            right_res = dfs(root->right, ans) + 1;
        }
        ans = max(ans, left_res + right_res);
        return max(left_res, right_res);
    }
    int diameterOfBinaryTree(TreeNode* root) {
        int ans = 0;
        dfs(root,ans);
        return ans;
    }
};
```

# 二叉树的层序遍历

> BFS

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>>ans;
        if (!root)return ans;
        vector<TreeNode *>q;
        q.emplace_back(root);
        vector<int>bas;
        while (!q.empty()) {
            vector<TreeNode *>nxt;
            bas.clear();
            for (auto it : q) {
                bas.emplace_back(it->val);
                if (it->left)nxt.emplace_back(it->left);
                if (it->right)nxt.emplace_back(it->right);
            }
            ans.emplace_back(bas);
            q = move(nxt);
        }
        return ans;
    }
};
```


# 将有序数组转换为二叉搜索树

> 递归

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    TreeNode* dfs(vector<int>& nums,int x, int y) {
        if (x >= y) {
            return nullptr;
        }
        int mid = (x + y - 1) / 2;
        return new TreeNode(nums[mid], dfs(nums, x, mid), dfs(nums, mid + 1, y));
    }
    TreeNode* sortedArrayToBST(vector<int>& nums) {
        // 递归建树
        int sz = nums.size();
        return dfs(nums, 0, sz);
    }
};
```


# 验证二叉搜索树

> 递归

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    bool dfs(TreeNode* root, long long minValue, long long maxValue) {
        if (!root) {
            return true;
        }
        bool flag = true;
        flag &= (root->val > minValue && root->val < maxValue);
        if (root->left) flag &= dfs(root->left, minValue, root->val);
        if (root->right) flag &= dfs(root->right, root->val, maxValue);
        return flag;
    }
    bool isValidBST(TreeNode* root) {
        return dfs(root, -1e11, 1e11);
    }
};
```


# 二叉搜索树中第K小的元素

> 

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int ans = 0, kth = 0;
    void dfs(TreeNode* root) {
        if (root->left) dfs(root->left);
        kth--;
        if (kth == 0) {
            ans = root->val;
        }
        if (root->right) dfs(root->right);
    }
    int kthSmallest(TreeNode* root, int k) {
        kth = k;
        dfs(root);
        return ans;
    }
};
```



# 二叉树的右视图

> 

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    map<int,int>mp;
    void dfs(TreeNode* root, int dep) {
        mp[dep] = root->val;
        if (root->left)dfs(root->left, dep+1);
        if (root->right)dfs(root->right, dep+1);
    }
    vector<int> rightSideView(TreeNode* root) {
        vector<int>ans;
        if (!root) return ans;
        dfs(root,1);
        for (auto it : mp) {
            ans.push_back(it.second);
        }
        return ans;
    }
};
```



# 二叉树展开为链表

> 

```cpp
class Solution {
    TreeNode* head;
public:
    void flatten(TreeNode* root) {
        if (root == nullptr) {
            return;
        }
        flatten(root->right);
        flatten(root->left);
        root->left = nullptr;
        root->right = head; // 头插法，相当于链表的 root->next = head
        head = root; // 现在链表头节点是 root
    }
};
```



# 从前序与中序遍历序列构造二叉树

> 

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    TreeNode* dfs(vector<int>&preorder, int pl, int pr, vector<int>&inorder, int il, int ir) {
        if (pl >= pr) {
            return nullptr;
        }
        if (pl + 1 == pr) {
            return new TreeNode(preorder[pl]);
        }
        TreeNode *root = new TreeNode(preorder[pl]);
        // 找根
        int pos = 0;
        for (int i = il; i < ir; ++i) {
            if (inorder[i] == preorder[pl]) {
                pos = i;
            }
        }
        int leftsz = pos - il;
        int right = ir - pos;
        root->left = dfs(preorder, pl + 1, pl + 1 + leftsz, inorder, il, pos);
        root->right = dfs(preorder, pl + 1 + leftsz, pr, inorder, pos + 1, ir);
        return root;
    }
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        return dfs(preorder, 0, preorder.size(), inorder, 0, inorder.size());
    }
};
```

# 路径总和 III

给定一个二叉树的根节点 root ，和一个整数 targetSum ，求该二叉树里节点值之和等于 targetSum 的 路径 的数目。

路径 不需要从根节点开始，也不需要在叶子节点结束，但是路径方向必须是向下的（只能从父节点到子节点）。

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    unordered_map<long long,int>sum;
    int dfs(TreeNode* root, long long preS, int targetSum) {
        int ans = 0;
        preS += root->val;
        if (sum.contains(preS - targetSum)) {
            ans += sum[preS - targetSum];
        }
        sum[preS] += 1;
        if (root->left) {
            ans += dfs(root->left, preS, targetSum);
        }
        if (root->right) {
            ans += dfs(root->right, preS, targetSum);
        }
        sum[preS] -= 1;
        return ans;
    }
    int pathSum(TreeNode* root, int targetSum) {
        if (!root) return 0;
        sum[0] = 1;
        return dfs(root, 0, targetSum);
    }
};
```

# 二叉树的最近公共祖先

> 

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    unordered_map<int,int>mp;
    unordered_map<int,TreeNode *>fa;

    void dfs(TreeNode *root, int dep, TreeNode* fat) {
        fa[root->val] = fat;
        if (root->left) dfs(root->left, dep + 1, root);
        if (root->right) dfs(root->right, dep + 1, root);
    }
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        dfs(root, 0, nullptr);
        while (p) {
            mp[p->val] = 1;
            p = fa[p->val];    
        }
        while(q) {
            if (mp.contains(q->val)) {
                return q;
            }
            q = fa[q->val];
        }
        return NULL;
    }
};
```

# 二叉树中的最大路径和

> 树dp

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int ans = -1e9;
    int dfs(TreeNode* root) {
        int leftRes = 0;
        int rightRes = 0;
        if (root->left) leftRes = dfs(root->left);
        if (root->right) rightRes = dfs(root->right);
        ans = max({ans, leftRes + rightRes + root->val, root->val + leftRes, root->val + rightRes, root->val});
        return max({leftRes + root->val, rightRes + root->val, root->val});
    }
    int maxPathSum(TreeNode* root) {
        dfs(root);
        return ans;
    }
};
```

# 

> 

```cpp

```