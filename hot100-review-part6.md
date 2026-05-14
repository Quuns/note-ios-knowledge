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

> 

```cpp

```

# 完全平方数

> 

```cpp

```

# 零钱兑换

> 

```cpp

```

# 单词拆分 

> 

```cpp

```

# 最长递增子序列

> 

```cpp

```

# 乘积最大子数组

> 

```cpp

```

# 分割等和子集

> 

```cpp

```

# 最长有效括号

> 

```cpp

```

# 

> 

```cpp

```