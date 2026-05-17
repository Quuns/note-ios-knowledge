# 数组中的第K个最大元素

> 快速选择 O(n)

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        ranges::nth_element(nums, nums.end() - k);
        return nums[nums.size() - k];
    }
};

class Solution {
    // 在子数组 [left, right] 中随机选择一个基准元素 pivot
    // 根据 pivot 重新排列子数组 [left, right]
    // 重新排列后，<= pivot 的元素都在 pivot 的左侧，>= pivot 的元素都在 pivot 的右侧
    // 返回 pivot 在重新排列后的 nums 中的下标
    // 特别地，如果子数组的所有元素都等于 pivot，我们会返回子数组的中心下标，避免退化
    int partition(vector<int>& nums, int left, int right) {
        // 1. 在子数组 [left, right] 中随机选择一个基准元素 pivot
        int i = left + rand() % (right - left + 1);
        int pivot = nums[i];
        // 把 pivot 与子数组第一个元素交换，避免 pivot 干扰后续划分，从而简化实现逻辑
        swap(nums[i], nums[left]);

        // 2. 相向双指针遍历子数组 [left + 1, right]
        // 循环不变量：在循环过程中，子数组的数据分布始终如下图
        // [ pivot | <=pivot | 尚未遍历 | >=pivot ]
        //   ^                 ^     ^         ^
        //   left              i     j         right

        i = left + 1;
        int j = right;
        while (true) {
            while (i <= j && nums[i] < pivot) {
                i++;
            }
            // 此时 nums[i] >= pivot

            while (i <= j && nums[j] > pivot) {
                j--;
            }
            // 此时 nums[j] <= pivot

            if (i >= j) {
                break;
            }

            // 维持循环不变量
            swap(nums[i], nums[j]);
            i++;
            j--;
        }

        // 循环结束后
        // [ pivot | <=pivot | >=pivot ]
        //   ^             ^   ^     ^
        //   left          j   i     right

        // 3. 把 pivot 与 nums[j] 交换，完成划分（partition）
        // 为什么与 j 交换？
        // 如果与 i 交换，可能会出现 i = right + 1 的情况，已经下标越界了，无法交换
        // 另一个原因是如果 nums[i] > pivot，交换会导致一个大于 pivot 的数出现在子数组最左边，不是有效划分
        // 与 j 交换，即使 j = left，交换也不会出错
        swap(nums[left], nums[j]);

        // 交换后
        // [ <=pivot | pivot | >=pivot ]
        //               ^
        //               j

        // 返回 pivot 的下标
        return j;
    }

public:
    int findKthLargest(vector<int>& nums, int k) {
        srand(time(NULL));
        int n = nums.size();
        int target_index = n - k; // 第 k 大元素在升序数组中的下标是 n - k
        int left = 0, right = n - 1; // 闭区间
        while (true) {
            int i = partition(nums, left, right);
            if (i == target_index) {
                // 找到第 k 大元素
                return nums[i];
            }
            if (i > target_index) {
                // 第 k 大元素在 [left, i - 1] 中
                right = i - 1;
            } else {
                // 第 k 大元素在 [i + 1, right] 中
                left = i + 1;
            }
        }
    }
};

```

# 前K个高频元素

给你一个整数数组 nums 和一个整数 k ，请你返回其中出现频率前 k 高的元素。你可以按 任意顺序 返回答案

输入：nums = [1,1,1,2,2,3], k = 2

输出：[1,2]

> 哈希表 O(n) + 桶排序 O(n)

```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        unordered_map<int, int>mp;
        int cnt = 0;
        for (auto i : nums) {
            mp[i]++;
            cnt = max(cnt, mp[i]);
        }
        unordered_map<int, vector<int>>t;
        for (auto [x, c] : mp) {
            t[c].push_back(x);
        }
        vector<int>ans;
        for (int i = cnt; ans.size() < k; i--) {
            for (auto d : t[i]) ans.push_back(d);
        }
        return ans;
    }
};
```


# 数据流的中位数

> 堆 O(logn) + 对顶堆

```cpp
class MedianFinder {
public:
    // 对顶堆
    priority_queue<int> left;// 最大堆
    priority_queue<int, vector<int>, greater<>> right; // 最小堆
    MedianFinder() {
        
    }
    
    void addNum(int num) {
        if (left.size() > right.size()) {
            left.push(num);
            right.push(left.top());
            left.pop();
        } else {
            right.push(num);
            left.push(right.top());
            right.pop();
        }
    }
    
    double findMedian() {
        if (left.size() > right.size()) {
            return left.top();
        } else {
            return (left.top() + right.top()) / 2.0;
        }
    }
};

/**
 * Your MedianFinder object will be instantiated and called as such:
 * MedianFinder* obj = new MedianFinder();
 * obj->addNum(num);
 * double param_2 = obj->findMedian();
 */
```


# 买卖股票的最佳时机


给定一个数组 prices ，它的第 i 个元素 prices[i] 表示一支给定股票第 i 天的价格。

你只能选择 某一天 买入这只股票，并选择在 未来的某一个不同的日子 卖出该股票。设计一个算法来计算你所能获取的最大利润。

返回你可以从这笔交易中获取的最大利润。如果你不能获取任何利润，返回 0 。


> 动态规划 O(n)

```cpp
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        if (prices.size() == 1) return 0;
        vector<int>prmin;
        prmin.resize(prices.size());
        int index = 0;
        int mi = 0x3f3f3f3f;
        for (int i = 0; i < prices.size(); ++i) {
            mi = min(mi, prices[i]);
            prmin[i] = mi;
        }
        int ans = 0;
        for (int i = 1; i < prices.size(); ++i) {
            if (prmin[i-1] < prices[i]) {
                ans = max(ans, prices[i] - prmin[i-1]);
            }
        }
        return ans;
    }
};
```


# 跳跃游戏

给你一个非负整数数组 nums ，你最初位于数组的 第一个下标 。数组中的每个元素代表你在该位置可以跳跃的最大长度。

判断你是否能够到达最后一个下标，如果可以，返回 true ；否则，返回 false 。

输入：nums = [3,2,1,0,4]

输出：false

解释：无论怎样，总会到达下标为 3 的位置。但该下标的最大跳跃长度是 0 ， 所以永远不可能到达最后一个下标。

> 贪心 O(n)

```cpp
class Solution {
public:
    bool canJump(vector<int>& nums) {
        int sz = nums.size();
        if (sz == 1) {
            return true;
        }
        int canGet = 0;
        for (int i = 0; i < sz - 1; ++i) {
            if (i > canGet) {
                // 到了不能到的地方
                return false;
            }
            nums[i] = i + nums[i];
            canGet = max(canGet, nums[i]);
            if (canGet >= sz - 1) {
                return true;
            }
        }
        return false;
    }
};
```


# 跳跃游戏II

给定一个长度为 n 的 0 索引整数数组 nums。初始位置在下标 0。

每个元素 nums[i] 表示从索引 i 向后跳转的最大长度。换句话说，如果你在索引 i 处，你可以跳转到任意 (i + j) 处：

0 <= j <= nums[i] 且
i + j < n
**返回到达 n - 1 的最小跳跃次数。测试用例保证可以到达 n - 1。**


> 贪心 O(n)

```cpp
class Solution {
public:
    int jump(vector<int>& nums) {
        int ans = 0;
        int current_right = 0;// 已建造的最right
        int next_right = 0;// 下一个桥的最右
        for (int i = 0; i + 1 < nums.size(); ++i) {
            next_right = max(next_right, i + nums[i]);
            if (i == current_right) {// 无路可走 建桥
                ans++; 
                current_right = next_right; // 更新
            }
        }
        return ans;
    }
};
```


# 划分字母区间

给你一个字符串 s 。我们要把这个字符串划分为尽可能多的片段，同一字母最多出现在一个片段中。例如，字符串 "ababcc" 能够被分为 ["abab", "cc"]，但类似 ["aba", "bcc"] 或 ["ab", "ab", "cc"] 的划分是非法的。

注意，划分结果需要满足：将所有划分结果按顺序连接，得到的字符串仍然是 s 。

返回一个表示每个字符串片段的长度的列表。

输入：s = "ababcbacadefegdehijhklij"

输出：[9,7,8]

解释：
划分结果为 "ababcbaca"、"defegde"、"hijhklij" 。
每个字母最多出现在一个片段中。
像 "ababcbacadefegde", "hijhklij" 这样的划分是错误的，因为划分的片段数较少。 


> 贪心 O(n)

```cpp
class Solution {
public:
    vector<int> partitionLabels(string s) {
        // 区间合并
        int n = s.size();
        int last[26];
        for (int i = 0; i < n; ++i) {
            last[s[i] - 'a'] = i;
        }
        vector<int>ans;
        int st = 0, ed = 0;
        for (int i = 0; i < n; ++i) {
            ed = max(ed, last[s[i] - 'a']); // 更新当前区间右端点最大值
            if (i == ed) {// 当前区间合并完毕
                ans.push_back(ed - st + 1); // 区间长度加入答案
                st = ed + 1; // 下一个区间的左端点
            }
        }
        return ans;
    }
};
```


# 爬楼梯

> 动态规划 O(1)

```cpp
class Solution {
public:
    int climbStairs(int n) {
        int a = 1, b = 2;
        int c;
        if (n == 1) return 1;
        if (n == 2) return 2;
        for (int i = 3; i <= n; ++i) {
            c = a + b;
            a = b;
            b = c;
        }
        return c;
    }
};
```


# 杨辉三角

![img](https://pic.leetcode.cn/1626927345-DZmfxB-PascalTriangleAnimated2.gif)

> 动态规划 O(n^2)

```cpp
class Solution {
public:
    vector<vector<int>> generate(int numRows) {
        // resize 动态分配
        vector<vector<int>>ans(numRows);
        for (int i=0;i<numRows;++i) {
            ans[i].resize(i+1);
            ans[i][0] = ans[i][i] = 1;
            for (int j=1;j<i;++j){
                ans[i][j] = ans[i-1][j] + ans[i-1][j-1];
            }
        }
        return ans;
    }
};
```

# 打家劫舍

你是一个专业的小偷，计划偷窃沿街的房屋。每间房内都藏有一定的现金，影响你偷窃的唯一制约因素就是相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警。

给定一个代表每个房屋存放金额的非负整数数组，计算你 不触动警报装置的情况下 ，一夜之内能够偷窃到的最高金额。

> 动态规划 O(n)

```cpp
class Solution {
public:
    int rob(vector<int>& nums) {
        int sz = nums.size();
        vector<int>dp;dp.resize(sz);
        dp[0] = nums[0];
        int ans = dp[0];
        for(int i=1;i<sz;++i){
            if (i >= 2) {
                dp[i] = max(dp[i], dp[i-1]);
                dp[i] = max(dp[i], nums[i] + dp[i-2]);
            } else {
                dp[i] = max(dp[i], nums[i]);
                dp[i] = max(dp[i], nums[i-1]);
            }
            ans = max(ans, dp[i]);
        }
        return ans;
    }
};
```

# 完全平方数

> 动态规划 O(n^2)

```cpp
class Solution {
public:
    int numSquares(int n) {
        vector<int>dp(n + 1, 0x3f3f3f3f);
        dp[0] = 0;
        for (int i = 1; i * i <= n; ++i) dp[i * i] = 1;
        for (int i = 1; i <= n; ++i) {
            for (int j = 1; j * j <= n; ++j) {
                if (i - j * j >= 0)dp[i] = min(dp[i], dp[i - j * j] + 1);
            }
        }
        return dp[n];
    }
};
```

# 零钱兑换

> 动态规划 O(n^2)

```cpp
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        // dp[i] 凑成数字 i 的最小硬币数
        vector<int>dp(amount + 1, 0x3f3f3f3f);
        int sz = coins.size();
        dp[0] = 0;
        for (int i = 1; i <= amount; ++i) {
            for (int j = 0; j < sz; ++j) {
                if (i >= coins[j]) dp[i] = min(dp[i - coins[j]] + 1, dp[i]);
            }
        }
        return (dp[amount] == 0x3f3f3f3f) ? -1 : dp[amount];
    }
};
```

# 单词拆分 

> 动态规划 O(n^2)

```cpp
class Solution {
public:
    bool wordBreak(string s, vector<string>& wordDict) {
        // dp[i] = 以i结尾的前缀是否能被完整划分
        int sz = s.size();
        vector<bool>dp(sz + 1, false);
        dp[0] = true;
        for (int i = 1; i <= sz; ++i) {
            for (auto sub : wordDict) {
                int len = sub.size();
                if (i - len >= 0 && s.substr(i - len, len) == sub) {
                    dp[i] = (dp[i] | dp[i-len]);
                }
            }
        }
        return dp[sz];
    }
};
```

# **最长递增子序列**

> 动态规划 O(n^2) 

```cpp
class Solution {
public:
    int lengthOfLIS(vector<int>& nums) {
        int sz = nums.size();
        int ans = -INT_MAX;
        vector<int>dp(sz, 0);
        for (int i = 0; i < sz; ++i) {
            dp[i] = 1;
            for (int j = 0; j < i; ++j) {
                if (nums[i] > nums[j]) {
                    dp[i] = max(dp[i], dp[j] + 1);
                }
            }
            ans = max(ans, dp[i]);
        }
        return ans;
    }
};


class Solution {
public:
    int lengthOfLIS(vector<int>& nums) {
        // 二分
        vector<int> g;
        for (int x : nums) {
            auto it = ranges::lower_bound(g, x);
            if (it == g.end()) {
                g.push_back(x); // >=x 的 g[j] 不存在
            } else {
                *it = x;
            }
        }
        return g.size();
    }
};

```

# 乘积最大子数组

> 动态规划 O(n)

```cpp
class Solution {
public:
    int maxProduct(vector<int>& nums) {
        int n = nums.size();
        vector<int> f_max(n), f_min(n);
        f_max[0] = f_min[0] = nums[0];
        for (int i = 1; i < n; i++) {
            int x = nums[i];
            // 把 x 加到右端点为 i-1 的（乘积最大/最小）子数组后面，
            // 或者单独组成一个子数组，只有 x 一个元素
            f_max[i] = max({f_max[i - 1] * x, f_min[i - 1] * x, x});
            f_min[i] = min({f_max[i - 1] * x, f_min[i - 1] * x, x});
        }
        return ranges::max(f_max);
    }
};
```

# 分割等和子集

> bitset + dp

```cpp
class Solution {
public:
    bool canPartition(vector<int>& nums) {
        int sum = 0;
        for (auto it : nums) {
            sum += it;
        }
        if (sum % 2) return false;
        int sz = nums.size();
        bitset<10001>bs;
        bs[0] = 1;
        for (auto it : nums) {
            bs |= bs << it;
            if (bs[sum/2]) return true;// 优化
        }
        return false;
    }
};
```

# 最长有效括号

> 动态规划 O(n)

```cpp
class Solution {
public:
    int longestValidParentheses(string s) {
        int len = s.size();
        int ans = 0;
        vector<int>dp(len + 1, 0);
        dp[0] = 0;
        for (int i = 1; i < len; ++i) {
            if (s[i] == ')') {
                if (s[i-1] == '(') {
                    if (i >= 2)dp[i] = dp[i-2] + 2;
                    else {
                        dp[i] = 2;
                    }
                } else {
                    int pre = i - dp[i-1] - 1;
                    if (pre >= 0 && s[pre] == '(') {
                        if (pre - 1 >= 0)dp[i] = dp[pre - 1] + 2 + dp[i-1];
                        else dp[i] = dp[i-1] + 2;
                    }
                }
            }
            ans = max(ans, dp[i]);
        }
        return ans;
    }
};
```

# 不同路径

> 动态规划 O(mn)

```cpp
class Solution {
public:
    int uniquePaths(int m, int n) {
        vector<vector<int>>dp(m,vector<int>(n, 0));
        dp[0][0] = 1;
        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j) {
                if (i - 1 >= 0) dp[i][j] += dp[i-1][j];
                if (j - 1 >= 0) dp[i][j] += dp[i][j-1];
            }
        }
        return dp[m - 1][n - 1];
    }
};
```

# 最小路径和

给定一个包含非负整数的 m x n 网格 grid ，请找出一条从左上角到右下角的路径，使得路径上的数字总和为最小。

说明：每次只能向下或者向右移动一步。

> 动态规划 O(mn)

```cpp
class Solution {
public:
    int minPathSum(vector<vector<int>>& grid) {
        int n = grid.size();
        int m = grid[0].size();
        vector<vector<int>>dp(n, vector<int>(m, 0x3f3f3f3f));
        dp[0][0] = grid[0][0];
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (i - 1 >= 0) dp[i][j] = min(dp[i][j], dp[i-1][j] + grid[i][j]);
                if (j - 1 >= 0) dp[i][j] = min(dp[i][j], dp[i][j-1] + grid[i][j]);
            }
        }
        return dp[n-1][m-1];
    }
};
```


# 最长回文子串

给你一个字符串 s，找到 s 中最长的 回文 子串。

> 中心扩展法 O(n^2)

```cpp
class Solution {
public:
    string longestPalindrome(string s) {
        int len = s.size();
        string ans;
        int mx = 0;
        // 奇长度中点枚举
        for (int i = 0; i < len; ++i) {
            int L = i;
            int R = i;
            while (L >= 0 && R < len) {
                if (s[L] == s[R]) {
                    L--;
                    R++;
                } else {
                    break;
                }
            }
            L++;
            R--;
            if (R - L + 1 >= mx && L >= 0 && R < len) {
                mx = max(mx, R - L + 1);
                ans = s.substr(L, R - L + 1);
            }
        }
        // 偶长度中点枚举
        for (int i = 0; i < len - 1; ++i) {
            int L = i;
            int R = i + 1;
            if (s[L] != s[R]) continue;
            while (L >= 0 && R < len) {
                if (s[L] == s[R]) {
                    L--;
                    R++;
                } else {
                    break;
                }
            }
            L++;
            R--;
            if (R - L + 1 >= mx && L >= 0 && R < len) {
                mx = max(mx, R - L + 1);
                ans = s.substr(L, R - L + 1);
            }
        }
        return ans;
    }
};
```

# 最长公共子序列

> 动态规划 O(n^2)

```cpp
class Solution {
public:
    int longestCommonSubsequence(string text1, string text2) {
        int n = text1.size();
        int m = text2.size();
        vector<vector<int>>dp(n + 1, vector<int>(m + 1, 0));
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (text1[i] == text2[j]) {
                    dp[i+1][j+1] = dp[i][j] + 1;
                } else {
                    dp[i+1][j+1] = max(dp[i][j+1], dp[i+1][j]);
                }
            }
        }
        return dp[n][m];
    }
};
```


# 编辑距离


给你两个单词 word1 和 word2， 请返回将 word1 转换成 word2 所使用的最少操作数  。

你可以对一个单词进行如下三种操作：

插入一个字符

删除一个字符

替换一个字符


> 动态规划 O(n^2) + 记忆化搜索

```cpp
class Solution {
public:
    int minDistance(string word1, string word2) {
        int n = word1.size();
        int m = word2.size();
        vector<vector<int>>memo(n, vector<int>(m, 0x3f3f3f3f));
        auto dfs = [&](this auto&& dfs, int i, int j) -> int {
            if (i < 0) {
                // i 空 补
                return j + 1;
            }
            if (j < 0) {
                // j 空 删
                return i + 1;
            }
            int& res = memo[i][j];
            if (res != 0x3f3f3f3f) {
                return res;
            }
            if (word1[i] == word2[j]) {
                res = min(res, dfs(i-1,j-1)); // 匹配
            } else {
                res = min({res, dfs(i-1,j) + 1, dfs(i, j-1) + 1, dfs(i-1,j-1) + 1}); // 删除-插入-替换
            }
            return res;
        };
        return dfs(n-1,m-1);
    }
};
```