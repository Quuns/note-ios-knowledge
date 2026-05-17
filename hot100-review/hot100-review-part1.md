# 两数之和
> 哈希map
```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> hashtable;
        for (int i = 0; i < nums.size(); ++i) {
            auto it = hashtable.find(target - nums[i]);
            if (it != hashtable.end()) {
                return {it->second, i};
            }
            hashtable[nums[i]] = i;
        }
        return {};
    }
};
```

# 字母异位词分组

```cpp
class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string>& strs) {
        vector<vector<string>>s;
        map<string,vector<string>>mp;
        for (auto it : strs) {
            int len = it.size();
            vector<char>s;
            for (auto i : it) {
                s.push_back(i);
            }
            sort(s.begin(),s.end());
            string ss;
            for (auto f : s) {
                ss+=f;
            }
            mp[ss].push_back(it);
        }
        for (auto it : mp) {
            s.push_back(it.second);
        }
        return s;
    }
};
```

# 最长数字连续序列
> 输入 [100,4,200,1,3,2] 输出 4 最长连续序列是 [1,2,3,4] 长度为 4

```cpp
class Solution {
public:
    int longestConsecutive(vector<int>& nums) {
        unordered_set<int>st(nums.begin(),nums.end());
        int ans = 0;
        for(int x : st) {
            if (st.contains(x-1)) {
                continue;
            }
            int y = x + 1;
            while (st.contains(y)) {
                y++;
            }
            ans = max(ans, y - x);
            if (ans * 2 >= st.size()) { // 如果一个最长连续序列的长度大于等于数组长度的一半，那么就直接返回最长连续序列的长度
                break;
            }
        }
        return ans;
    }
};
```

# 移动零 【很巧妙的解法】

给定一个数组 nums，编写一个函数将所有 0 移动到数组的末尾，同时保持非零元素的相对顺序。

请注意 ，必须在不复制数组的情况下原地对数组进行操作。

```cpp
class Solution {
public:
    void moveZeroes(vector<int>& nums) {
        int i0 = 0; // 即将交换的0的下标
        for (int& x : nums) { // 注意 x 是引用
            if (x) {
                swap(x, nums[i0]); // 交换非零元素到 i0 位置
                i0++;
            }
        }
    }
};
```

# 盛水最多的容器
> 头尾双指针, 移动小的那个指针
```cpp
class Solution {
public:
    int maxArea(vector<int>& height) {
        int ans = 0;
        int left = 0;
        int right = height.size() - 1;
        while (left < right) {
            ans = max(ans, (right - left) * min(height[left], height[right]));
            if (height[left] < height[right]) {
                left++;
            } else {
                right--;
            }
        }
        return ans;
    }
};
```

# 三数之和

求所有满足 nums[i] + nums[j] + nums[k] == 0 且 i != j 且 j != k 的三元组 [nums[i], nums[j], nums[k]] 。
> 哈希map
```cpp
class Solution {
public:
    vector<vector<int>> threeSum(vector<int>& nums) {
        vector<vector<int>>ans;
        unordered_map<int,int>mp;
        map<pair<int,int>,int>ck;
        int sz = nums.size();
        for (int j = 0; j < sz; ++j) {
            for (int k = j + 1; k < sz; ++k) {
                int mi = min({-nums[j]-nums[k], nums[j], nums[k]});
                int mx = max({-nums[j]-nums[k], nums[j], nums[k]});
                if (mp.contains(nums[j] + nums[k]) && !ck.contains({mi,mx})) {
                    ans.push_back({-nums[j] - nums[k], nums[j], nums[k]});
                    ck[{mi,mx}] = 1;
                }
            }
            mp[-nums[j]]++;
        }
        return ans;
    }
};
```

# 接雨水
> 前缀最大值
> 后缀最大值
> 遍历数组，计算每个位置的接水量量  
> 累加接水量量 
> 或者用单调栈做 针对前面比当前数小的数计算贡献
```cpp
class Solution {
public:
    int trap(vector<int>& height) {
        int ans = 0;
        int n = height.size();
        vector<int>suf(n + 1, 0);
        int pre = height[0];
        suf[n] = height[n - 1];
        for (int i = n - 1; i >= 0; --i) {
            suf[i] = max(height[i], (i == n - 1) ? height[i] : suf[i + 1]);
        }
        for (int i = 0; i < n; ++i) {
            // cout << pre << " " << suf[i+1] <<endl;
            ans += max(0, min(pre, suf[i + 1]) - height[i]);
            pre = max(pre, height[i]);
        }
        return ans;
    }
};
```

# 无重复字符的最长子串
> 滑动窗口 双指针 哈希map
```cpp
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        // 双指针 滑动串口
        int len = s.size();
        int l = 0;
        int ans = 0;
        unordered_map<int,int>mp;
        for (int i = 0; i < len; ++i) {
            if (mp[s[i]] == 0) {
                mp[s[i]]++;
                ans = max(ans, i - l + 1);
            } else {
                mp[s[i]]++;
                while (l < i && s[l] != s[i]) {
                    mp[s[l]]--;
                    l++;
                }
                mp[s[l++]]--;
                ans = max(ans, i - l + 1);
            }
        }
        return ans;
    }
};
```

# 找到字符串中所有字母异位词
> 滑动窗口 + 桶
> s = "cbaebabacd", p = "abc"
> 输出: [0,6] 返回起始索引为 0 和 6 的子串
> 解释: 子串 "cba" 和 "bac" 是 "abc" 的字母异位词。

```cpp
class Solution {
public:
    vector<int> findAnagrams(string s, string p) {
        // 定长滑动窗口
        array<int, 26> mps{},mpp{};
        vector<int>ans;
        int lens = s.size();
        int lenp = p.size();
        if (lenp > lens) {
            return ans;
        }
        for (char i : p) {
            mpp[i - 'a']++;
        }
        for (int i = 0; i < lens; ++i) {
            if (i >= lenp) {
                mps[s[i]-'a']++;
                mps[s[i - lenp]-'a']--;
            } else {
                mps[s[i]-'a']++;
            }
            int flag = 1;
            if (mps == mpp) {
                ans.push_back(i-lenp+1);
            }
        }
        return ans;
    }
};
```

# 和为k的子数组
> 前缀和 + 哈希map
> nums = [1,1,1], k = 2 
> 输出: 2
> 解释: 子数组 [1,1]  [1,1] 为两种不同的情况。
```cpp
class Solution {
public:
    int subarraySum(vector<int>& nums, int k) {
        unordered_map<long long,int>mp;
        int ans = 0;
        long long sum = 0;
        mp[k] = 1;
        for (int x : nums) {
            sum += x;
            if (mp.contains(sum)) {
                ans += mp[sum];
            }

            mp[sum + k] += 1;
        }
        return ans;
    }
};
```

# 滑动窗口最大值
> 单调队列
```cpp
class Solution {
public:
    vector<int> maxSlidingWindow(vector<int>& nums, int k) {
        int left = 0;
        int right = -1;
        int sz = nums.size();
        vector<int>q(sz, 0);
        vector<int>ans; // 维护单调递减队列，队头为最大值
        for (int i = 0; i < sz; ++i) {
            while (right >= left && nums[q[right]] <= nums[i]) {
                right--;
            }
            q[++right] = i;
            if (right >= left && i - q[left] + 1 > k) {
                left++;
            }
            if (i >= k - 1) {
                ans.push_back(nums[q[left]]);
            }
        }
        return ans;
    }
};
```


# 最小覆盖子串
> 双指针 哈希map
> s = "ADOBECODEBANC", t = "ABC"
> 输出: "BANC"
> 解释：最小覆盖子串 "BANC" 包含来自字符串 t 的 'A'、'B' 和 'C'。
```cpp
class Solution {
public:
    string minWindow(string s, string t) {
        int lens = s.size();
        int lent = t.size();
        int mt[129]{};
        int kinds = 0;
        for(auto it : t) {
            if (mt[it] == 0) kinds++;
            mt[it]--; // 做差
        }

        int left = 0;
        int mlen = 0x3f3f3f3f;
        int st = 0;
        int cnt = 0;
        for (int right = 0; right < lens; ++right) { // 右节点一点点右移
            mt[s[right]]++;
            if (mt[s[right]] == 0) cnt++;
            // 每右移一次校验可行性，可行的话尝试压缩
            while (cnt == kinds) {
                if (right - left + 1 < mlen) {
                    mlen = right - left + 1;
                    st = left;
                }
                if (mt[s[left]] == 0) cnt--;
                mt[s[left]]--;
                left++;
            }
        }   
        if (mlen == 0x3f3f3f3f) return "";
        return s.substr(st, mlen);
    }
};
```


# 最大字数和
> dp 选或者不选
```cpp
class Solution {
public:
    int maxSubArray(vector<int>& nums) {
        int pre = 0, maxAns = nums[0];
        for (int x : nums) {
            pre = max(pre + x, x);
            maxAns = max(maxAns, pre);
        }
        return maxAns;
    }
};
```

# 合并区间
> 排序 + 贪心

> 输入：intervals = [[1,3],[2,6],[8,10],[15,18]]
> 输出：[[1,6],[8,10],[15,18]]
> 解释：区间 [1,3] 和 [2,6] 重叠, 将它们合并为 [1,6].
```cpp
class Solution {
public:
    vector<vector<int>> merge(vector<vector<int>>& intervals) {
        ranges::sort(intervals); // 按照左端点从小到大排序
        vector<vector<int>> ans;
        for (auto& p : intervals) {
            if (!ans.empty() && p[0] <= ans.back()[1]) { // 可以合并
                ans.back()[1] = max(ans.back()[1], p[1]); // 更新右端点最大值
            } else { // 不相交，无法合并
                ans.emplace_back(p); // 新的合并区间
            }
        }
        return ans;
    }
};
```

# 轮转数组
> 翻转结论题

> 输入: nums = [1,2,3,4,5,6,7], k = 3
> 输出: [5,6,7,1,2,3,4]
```cpp
class Solution {
public:
    void rotate(vector<int>& nums, int k) {
        k %= nums.size();
        reverse(nums.begin(),nums.end());
        reverse(nums.begin(),nums.begin()+k);
        reverse(nums.begin()+k,nums.end());
    }
};
```


# 除了自身以外数组的乘积    
> 前缀和
> 输入: nums = [1,2,3,4]
> 输出: [24,12,8,6]
> 解释: 除 nums[i] 之外的所有元素的乘积为 [24,12,8,6]。
```cpp
class Solution {
public:
    vector<int> productExceptSelf(vector<int>& nums) {
        int n = nums.size();
        vector<int> suf(n);
        suf[n - 1] = 1;
        for (int i = n - 2; i >= 0; i--) {
            suf[i] = suf[i + 1] * nums[i + 1];
        }

        int pre = 1;
        for (int i = 0; i < n; i++) {
            // 此时 pre 为 nums[0] 到 nums[i-1] 的乘积，直接乘到 suf[i] 中
            suf[i] *= pre;
            pre *= nums[i];
        }

        return suf;
    }
};
```

# 【***】缺失的第一个正数
> 原地负数标记法求mex

输入：nums = [1,2,0]
输出：3

输入：nums = [3,4,-1,1]
输出：2

```cpp
class Solution {
public:
    int firstMissingPositive(vector<int>& nums) {
        // 原地负数标记法求mex
        int sz = nums.size();
        for (auto & x : nums) {
            if (!(x >= 1 && x <= sz)) {
                x = sz + 1; // 把超出【1，n】的修改为n+1
            }
        }
        for (auto & x : nums) {
            int absx = abs(x); // 取绝对值
            if (absx <= sz && nums[absx - 1] > 0) {
                // 把对应的下标的数设置成负数
                nums[absx - 1] = -nums[absx - 1];
            }
        }
        for (int i = 0; i < sz; ++i) {
            if (nums[i] > 0) {
                // 第一个大于0的数 i + 1 就是mex
                return i + 1;
            }
        }
        // 如果全是负数，那么mex就是sz + 1
        return sz + 1;
    }
};
```


# 矩阵置零
> 原地标记法 如果一个元素为 0 ，则将其所在行和列的所有元素都设为 0 

> 利用第一行第一列作为标记，避免重复标记
```cpp
class Solution {
public:
    void setZeroes(vector<vector<int>>& matrix) {
        int n = matrix.size(), m = matrix[0].size();
        bool col0 = 0, row0 = 0;
        for (int i = 0; i < m; ++i) {
            if (matrix[0][i] == 0) row0 = 1;
        }
        for (int i = 0; i < n; ++i) {
            if (matrix[i][0] == 0) col0 = 1;
        }
        for (int i = 1; i < n; ++i) {
            for (int j = 1; j < m; ++j) {
                if (matrix[i][j] == 0) {
                    matrix[i][0] = 0;
                    matrix[0][j] = 0;
                }
            }
        }
        for (int i = 1; i < n; ++i) {
            for (int j = 1; j < m; ++j) {
                if (!matrix[i][0] || !matrix[0][j]) {
                    matrix[i][j] = 0;
                }
            }
        }
        if (row0) {
            for (int i = 0; i < m; ++i) {
                matrix[0][i] = 0;
            }
        }
        if (col0) {
            for (int i = 0; i < n; ++i) {
                matrix[i][0] = 0;
            }
        }
    }
};
```


# 螺旋矩阵
>
```cpp
class Solution {
public:
    int dx[4] = {0, 1, 0, -1};
    int dy[4] = {1, 0, -1, 0};
    vector<int> spiralOrder(vector<vector<int>>& matrix) {
        int n = matrix.size();
        int m = matrix[0].size();
        int sz = n * m;
        vector<int>ans;
        int st = 0;
        int x = 0;
        int y = -1;
        for (int dr = 0, j = 0; j < sz; dr = (dr + 1) %4) {
            for (int i = 0; i < m; ++i) {
                int nx = x + dx[dr];
                int ny = y + dy[dr];
                x = nx; y = ny;
                ans.push_back(matrix[nx][ny]);
                j++;
            }
            n--;
            swap(n,m);
        }
        return ans;
    }
};
```


# 旋转图像
> 顺时针旋转90度，要求原地旋转。矩阵转置 +行翻转 = 顺时针旋转90度
```cpp
class Solution {
public:
    void rotate(vector<vector<int>>& matrix) {
        int n = matrix.size();
        // 转置
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < i; ++j) {
                swap(matrix[i][j], matrix[j][i]);
            }
        }
        // 行翻转
        for (auto& row : matrix) {
            ranges::reverse(row);
        }
    }
};
```

# 搜索二维矩阵 II
编写一个高效的算法来搜索 m x n 矩阵 matrix 中的一个目标值 target 。该矩阵具有以下特性：

每行的元素从左到右升序排列。
每列的元素从上到下升序排列。
> 排除法。用右上角的元素来排除
```cpp
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        // 排除法
        int n = matrix.size();
        int m = matrix[0].size();
        int x = 0;
        int y = m - 1;
        while (x <= n - 1 && y >= 0) {
            if (matrix[x][y] == target) return true;
            if (matrix[x][y] < target) x++;//排除行
            else y--;//排除列
        }
        return false;
    }
};
```


# 
>
```cpp

```
