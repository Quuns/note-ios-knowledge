# 岛屿数量

> dfs

```cpp
class Solution {
public:
    int dx[4] = {1,0,-1,0}, dy[4] = {0,1,0,-1};
    void dfs(int x,int y, vector<vector<char>>& grid) {
        int n = grid.size(), m = grid[0].size();
        grid[x][y] = '2';
        // cout<<"x="<<x<<" y="<<y<<endl;
        for (int i = 0; i < 4; ++i) {
            int nx = x + dx[i];
            int ny = y + dy[i];
            if (nx >= n || nx < 0 || ny >= m || ny < 0 || grid[nx][ny] == 0) continue;
            dfs(nx,ny,grid);
        }
    }
    int numIslands(vector<vector<char>>& grid) {
        int n = grid.size(), m = grid[0].size();
        int ans = 0;
        for (int i = 0; i < n; ++i ) {
            for (int j = 0; j < m; ++j) {
                if (grid[i][j] == '1') {
                    dfs(i,j,grid);
                    ans++;
                }
            }
        }
        return ans;
    }
};
```

# 腐烂的橘子

在给定的 m x n 网格 grid 中，每个单元格可以有以下三个值之一：

值 0 代表空单元格；
值 1 代表新鲜橘子；
值 2 代表腐烂的橘子。
每分钟，腐烂的橘子 周围 4 个方向上相邻 的新鲜橘子都会腐烂。

返回 直到单元格中没有新鲜橘子为止所必须经过的最小分钟数。如果不可能，返回 -1 。

> bfs

```cpp
class Solution {
public:
    int dx[4] = {1,0,-1,0}, dy[4] = {0,1,0,-1};
    int orangesRotting(vector<vector<int>>& grid) {
        int ans = 0;
        int n = grid.size();
        if (n == 0) return -1;
        int m = grid[0].size();
        vector<pair<int,int>>q;
        int num = 0;
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (grid[i][j] == 2) q.emplace_back(i, j);
                else if (grid[i][j] == 1) num++;
            }
        }
        if (q.empty()) {
            if (num == 0) return 0;
            else return -1;
        }

        while (!q.empty()) {
            ans++;
            vector<pair<int,int>>nxt;
            for (auto& [x, y] : q) {
                for (int i = 0; i < 4; ++i) {
                    int nx = x + dx[i];
                    int ny = y + dy[i];
                    if (nx < 0 || ny < 0 || nx >= n || ny >= m || grid[nx][ny] != 1)continue;
                    grid[nx][ny] = 2;
                    num--;
                    nxt.emplace_back(nx, ny);
                }
            }
            q = move(nxt);
        }
        if (num > 0) return -1;
        else return ans - 1;
    }
};
```

# 课程表

> bfs + 拓扑排序

```cpp
class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
        vector<int>d(numCourses, 0);
        vector<int>v[numCourses];
        for (auto x : prerequisites) {
            v[x[1]].emplace_back(x[0]);
            d[x[0]] += 1;
        }
        queue<int>q;
        for (int i = 0; i < numCourses; ++i) {
            if (d[i] == 0) q.push(i);
        }
        while (!q.empty()) {
            int top = q.front();
            q.pop();
            for (auto it : v[top]) {
                d[it]--;
                if (d[it] == 0) {
                    q.push(it);
                }
            }
        }
        bool ans = true;
        for (int i = 0; i < numCourses; ++i) {
            ans &= (d[i] == 0);
        }
        return ans;
    }
};
```

# 实现Trie(前缀树)

> trie 动态空间 + 析构函数释放

```cpp
struct Node {
    Node* son[26]{};
    bool end = false;
};

class Trie {
    Node* rt;
    void destroy(Node* node) {
        if (node == nullptr) {
            return;
        }
        for (Node* son : node->son) {
            destroy(son);
        }
        delete node;
    }
public:
    ~Trie() {
        destroy(rt);
    }

    Trie() {
        rt = new Node();
    }
    
    void insert(string word) {
        Node *pos = rt;
        for (char c : word) {
            int val = c - 'a';
            if (!pos->son[val]) {
                pos->son[val] = new Node();
            }
            pos = pos->son[val];
        }
        pos->end = true;
    }
    
    bool search(string word) {
        Node *pos = rt;
        for (char c : word) {
            int val = c - 'a';
            pos = pos->son[val];
            if (!pos) {
                return false;
            }
        }
        return pos->end;
    }
    
    bool startsWith(string prefix) {
        Node * pos = rt;
        for (char c : prefix) {
            int val = c - 'a';
            pos = pos->son[val];
            if (!pos) {
                return false;
            }
        }
        return true;
    }
};

/**
 * Your Trie object will be instantiated and called as such:
 * Trie* obj = new Trie();
 * obj->insert(word);
 * bool param_2 = obj->search(word);
 * bool param_3 = obj->startsWith(prefix);
 */
```

# 全排列

给定一个不含重复数字的数组 nums ，返回其 所有可能的全排列 。你可以 按任意顺序 返回答案。

> dfs

```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int>& nums) {
        int n = nums.size();
        vector<vector<int>>ans;
        vector<int>path(n);
        vector<bool>vis(n);
        auto dfs = [&](this auto&& dfs, int x) -> void {
            if (x == n) {
                ans.push_back(path);
                return ;
            }
            for (int i = 0; i < n; ++i) {
                if (!vis[i]) {
                    path[x] = nums[i];
                    vis[i] = true;
                    dfs(x + 1);
                    vis[i] = false;
                }
            }
        };
        dfs(0);
        return ans;
    }
};
```


# 子集

给你一个整数数组 nums ，数组中的元素 互不相同 。返回该数组所有可能的子集（幂集）。

解集 不能 包含重复的子集。你可以按 任意顺序 返回解集。

> dfs

```cpp
class Solution {
public:
    vector<vector<int>>ans;
    int sz;
    vector<int>a;
    void dfs(int x, vector<int>& nums) {
        if (x == sz) {
            ans.push_back(a);
            return ;
        }
        dfs(x+1, nums);

        a.push_back(nums[x]);
        dfs(x+1, nums);
        a.pop_back();
    }
    vector<vector<int>> subsets(vector<int>& nums) {
        sz = nums.size();
        dfs(0, nums);
        return ans;
    }
};
```


# 电话号码的字母组合

给定一个仅包含数字 2-9 的字符串，返回所有它能表示的字母组合。答案可以按 任意顺序 返回。

给出数字到字母的映射如下（与电话按键相同）。注意 1 不对应任何字母。

![img](https://pic.leetcode.cn/1752723054-mfIHZs-image.png)

输入：digits = "23"

输出：["ad","ae","af","bd","be","bf","cd","ce","cf"]


> dfs

```cpp
class Solution {
public:
    vector<char> getChars(int number) {
        if (number == 2) {
            return {'a','b','c'};
        } else if (number == 3) {
            return {'d','e','f'};
        } else if (number == 4) {
            return {'g','h','i'};
        } else if (number == 5) {
            return {'j','k','l'};
        } else if (number == 6) {
            return {'m','n','o'};
        } else if (number == 7) {
            return {'p','q','r','s'};
        } else if (number == 8) {
            return {'t','u','v'};
        } else if (number == 9) {
            return {'w','x','y','z'};
        }
        return {};
    }
    int sz;
    vector<string>ans;
    void dfs(int x, string a, string digits) {
        if (x == sz) {
            ans.push_back(a);
            return ;
        }
        vector<char>v = getChars(digits[x] - '0');
        for (auto it : v) {
            dfs(x + 1, a + it, digits);
        }
    }
    vector<string> letterCombinations(string digits) {
        string a;
        sz = digits.size();
        dfs(0, a, digits);
        return ans;
    }
};
```


# 组合总和

给你一个 无重复元素 的整数数组 candidates 和一个目标整数 target ，找出 candidates 中可以使数字和为目标数 target 的 所有 不同组合 ，并以列表形式返回。你可以按 任意顺序 返回这些组合。

candidates 中的 同一个 数字可以 无限制重复被选取 。如果至少一个数字的被选数量不同，则两种组合是不同的。 

对于给定的输入，保证和为 target 的不同组合数少于 150 个。


输入：candidates = [2,3,6,7], target = 7

输出：[[2,2,3],[7]]

> dfs 

```cpp
class Solution {
public:
    vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
        sort(candidates.begin(), candidates.end());
        vector<vector<int>>ans;
        vector<int>v;
        auto dfs = [&](this auto&& dfs, int pos, int target, int sum) {
            if (sum >= target) {
                if (sum == target) {
                    ans.push_back(v);
                }
                return ;
            }
            for (int i = pos; i < candidates.size() && sum + candidates[i] <= target; ++i) {
                v.push_back(candidates[i]);
                dfs(i, target, sum + candidates[i]);
                v.pop_back();
            }
        };
        dfs(0, target, 0);
        return ans;
    }
};
```


# 括号生成

数字 n 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 有效的 括号组合

> dfs

```cpp
class Solution {
public:
    vector<string> generateParenthesis(int n) {
        vector<string>ans;
        string path(n * 2, 0);
        auto dfs = [&] (this auto&& dfs, int left, int right) -> void {
            if (right == n) {
                ans.push_back(path);
                return ;
            }
            if (left < n) {
                path[left + right] = '(';
                dfs(left + 1, right);
            }
            if (left > right) {
                path[left + right] = ')';
                dfs(left, right + 1);
            }
        };
        dfs(0, 0);
        return ans;
    }
};
```

# 单词搜索

给定一个 m x n 二维字符网格 board 和一个字符串单词 word 。如果 word 存在于网格中，返回 true ；否则，返回 false 。

单词必须按照字母顺序，通过相邻的单元格内的字母构成，其中“相邻”单元格是那些水平相邻或垂直相邻的单元格。同一个单元格内的字母不允许被重复使用。


> dfs

```cpp
class Solution {
public:
    int dx[4] = {1,0,-1,0};
    int dy[4] = {0,1,0,-1};
    bool exist(vector<vector<char>>& board, string word) {
        int n = board.size();
        int m = board[0].size();
        int flag = 0;
        vector<vector<bool>>vis(n, vector<bool>(m, false));
        auto dfs = [&] (this auto && dfs, int x, int y, int len) -> bool {
            if (board[x][y] != word[len]) {
                return false;
            }
            if (len == (int)word.size() - 1) {
                if (board[x][y] == word[len])return true;
                return false;
            }
            for (int i = 0; i < 4; ++i) {
                int nx = x + dx[i];
                int ny = y + dy[i];
                if (nx < 0 || nx >= n || ny < 0 || ny >= m || vis[nx][ny])continue;
                vis[nx][ny] = true;
                if (dfs(nx, ny, len + 1)) {
                    return true;
                }
                vis[nx][ny] = false;
            }
            return false;
        };
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                vis[i][j] = true;
                if (dfs(i, j, 0)) return true;
                vis[i][j] = false;
            }
        }
        return false;
    }
};
```

# 分割回文串

给你一个字符串 s，请你将 s 分割成一些 子串，使每个子串都是 回文串 。返回 s 所有可能的分割方案。

> 上一个回文子串终点枚举搜索 dfs

```cpp
class Solution {
public:
    vector<vector<string>> partition(string s) {
        vector<string>path;
        vector<vector<string>>ans;
        int sz = s.size();
        auto dfs = [&](this auto&& dfs, int pos) -> void {
            if (pos >= sz - 1) {
                ans.push_back(path);
                return ;
            }
            for (int i = pos + 1; i < sz; ++i) {
                int len = i - pos;
                string t = s.substr(pos + 1, len);
                string rt = t;
                reverse(rt.begin(), rt.end());
                
                if (t == rt) {
                    path.push_back(t);
                    dfs(i);
                    path.pop_back();
                }
            }
        };
        dfs(-1);
        return ans;
    }
};
```

# N 皇后

> dfs 行递归 列枚举

```cpp
class Solution {
public:
    vector<vector<string>> solveNQueens(int n) {
        int mpx_add_y[20] = {0},mpx_del_y[20] = {0};
        int visy[10] = {0};
        vector<string>s(n, string(n, '.'));
        vector<vector<string>>ans;
        auto dfs = [&](this auto && dfs,int y, int st) -> void {
            if (st == n) {
                ans.push_back(s);
                return ;
            }
            for (int j = 0; j < n; ++j) {
                int stj = st - j + n - 1; // 转正数两倍扩充
                if (visy[j]) continue;
                if (mpx_add_y[st+j] || mpx_del_y[stj]) continue;
                visy[j] = 1;
                mpx_add_y[st+j] = 1;
                mpx_del_y[stj] = 1;
                s[st][j] = 'Q';
                dfs(j, st + 1);
                s[st][j] = '.';
                visy[j] = 0;
                mpx_add_y[st+j] = 0;
                mpx_del_y[stj] = 0;
            }
        };
        dfs(0, 0);
        return ans;
    }
};
```

# 

> 

```cpp

```