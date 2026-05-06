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


# xxx
>
```cpp

```
