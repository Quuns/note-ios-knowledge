# 搜索插入位置

给定一个排序数组和一个目标值，在数组中找到目标值，并返回其索引。如果目标值不存在于数组中，返回它将会被按顺序插入的位置。


> 二分

```cpp
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int n = nums.size();
        int left = 0, right = n - 1;
        int ans = n;
        while (left <= right) {
            int mid = (left + right) / 2;
            int x = nums[mid];
            if (x >= target) {
                ans = mid; right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        return ans;
    }
};
```


# 搜索二维矩阵

给你一个满足下述两条属性的 m x n 整数矩阵：

每行中的整数从左到右按非严格递增顺序排列。
每行的第一个整数大于前一行的最后一个整数。
给你一个整数 target ，如果 target 在矩阵中，返回 true ；否则，返回 false 。

> 二分

```cpp
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        int m = matrix.size(), n = matrix[0].size();
        int left = 0, right = m * n - 1;
        while (left <= right) {
            int mid = (left + right) / 2;
            int x = matrix[mid / n][mid % n];
            if (x == target) {
                return true;
            }
            (x < target ? left : right) = mid + (x < target ? 1 : -1);
        }
        return false;
    }
};
```


# 在排序数组中查找元素的第一个和最后一个位置

给你一个按照非递减顺序排列的整数数组 nums，和一个目标值 target。请你找出给定目标值在数组中的开始位置和结束位置。

如果数组中不存在目标值 target，返回 [-1, -1]。

> 二分

```cpp
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int sz = nums.size();
        int l = 0;
        int r = sz - 1;
        int pos = -1;
        while (l <= r) {
            int mid = (l + r) / 2;
            if (nums[mid] >= target) {
                r = mid - 1;
                pos = mid;
            } else {
                l = mid + 1;
            }
        }
        int pos1 = -1;
        l = 0;
        r = sz - 1;
        while (l <= r) {
            int mid = (l + r) / 2;
            if (nums[mid] > target) {
                r = mid - 1;
                pos1 = mid;
            } else {
                l = mid + 1;
            }
        }
        if (pos == -1) return {-1, -1};
        else {
            if (pos1 == -1) {
                return {pos, sz - 1};
            }
            if ((pos1 - 1 >= 0 && nums[pos1 - 1] == target)) {
                return {pos, pos1-1};
            } else {
                return {-1, -1};
            }
        }
    }
};
```


# 搜索旋转排序数组

给你 旋转后 的数组 nums 和一个整数 target ，如果 nums 中存在这个目标值 target ，则返回它的下标，否则返回 -1 。

输入：nums = [4,5,6,7,0,1,2], target = 0

输出：4

> 二分 二分求出旋转数组的最小值下标 原题153. 寻找旋转排序数组中的最小值

```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int l = 0;
        int r = nums.size() - 1;
        while (l <= r) {
            int mid = (l + r) / 2;
            if (nums[mid] <= nums.back()) {
                // 在右端，右端整体值比较小, 往左靠
                r = mid - 1;
            } else {
                // 在左端现在，左端的值比较大，往右靠
                l = mid + 1;
            }
        }
        return l;
    }
    int search(vector<int>& nums, int target) {
        // 两次二分
        int l = 0;
        int r = nums.size() - 1;
        int index = findMin(nums);// 二分求出旋转数组的最小值下标 原题153. 寻找旋转排序数组中的最小值
        if (target > nums[r]) {
            // 在前面那一段
            r = index - 1;
        } else {
            // 在后面一段
            l = index;
        }
        int ans = -1;
        while (l <= r) {
            int mid = (l + r) / 2;
            if (nums[mid] >= target) {
                r = mid - 1;
                if (nums[mid] == target) {
                    ans = mid;
                }
            } else {
                l = mid + 1;
            }
        }
        return ans;
    }
};
```


# 寻找旋转排序数组中的最小值

给你一个元素值 互不相同 的数组 nums ，它原来是一个升序排列的数组，并按上述情形进行了多次旋转。请你找出并返回数组中的 最小元素 。

> 二分 上道题的子问题

```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int l = 0;
        int r = nums.size() - 1;
        while (l <= r) {
            int mid = (l + r) / 2;
            if (nums[mid] <= nums.back()) {
                // 在右端，右端整体值比较小, 往左靠
                r = mid - 1;
            } else {
                // 在左端现在，左端的值比较大，往右靠
                l = mid + 1;
            }
        }
        return nums[l];
    }
};
```


# **寻找两个正序数组的中位数**

给定两个大小分别为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。请你找出并返回这两个正序数组的 中位数 。

算法的时间复杂度应该为 O(log (m+n)) 。

输入：nums1 = [1,3], nums2 = [2]

输出：2.00000

解释：合并数组 = [1,2,3] ，中位数 2

> 

```cpp
class Solution {
public:
    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
        if (nums1.size() > nums2.size()) {
            swap(nums1, nums2);
        }
        int n = nums1.size();
        int m = nums2.size();
        int tot = n + m;
        int l = 0;
        int r = n;
        int half = ((tot + 1) / 2);
        while (l <= r) {
            int mid = (l + r) / 2;
            // 以 mid 为切割点  范围在[0,n] 切割成[0,mid-1] [mid,n-1]
            double right1 = (mid >= n) ? 0x3f3f3f3f : nums1[mid];
            double left1 = (mid - 1 < 0) ? -0x3f3f3f3f : nums1[mid-1];
            double left2 = (half - mid - 1 < 0) ? -0x3f3f3f3f : nums2[half - mid - 1];
            double right2 = (half - mid >= m) ? 0x3f3f3f3f : nums2[half - mid];
            if (left1 <= right2 && left2 <= right1) {
                if (tot % 2 == 1) return max(left1, left2);
                else return (max(left1, left2) + min(right1, right2)) / 2.0;
            } 
            if (left1 > right2) { // 上边的元素放太多了，少放点
                r = mid - 1;
            } else if (left2 > right1) {// 上边元素放太少了，多放点  等价于直接else
                l = mid + 1;
            }
        }
        return 0;
    }
};
```

# 有效的括号

> 栈

```cpp
class Solution {
public:
    bool isValid(string s) {
        stack<int>st;
        for (char c : s) {
            if (st.empty()) {
                st.push(c);
            } else {
                if ((st.top() == '(' && c == ')') || (st.top() == '[' && c == ']') || (st.top() == '{' && c == '}')) {
                    st.pop();
                } else {
                    st.push(c);
                }
            }
        }
        if (st.empty()) return true;
        else return false;
    }
};
```

# 最小栈

> 栈

```cpp
class MinStack {
public:
    stack<pair<int,int>>st;
    MinStack() {
        while(!st.empty())st.pop();
        st.push({0, INT_MAX});
    }
    
    void push(int val) {
        st.push({val, min(getMin(), val)});
    }
    
    void pop() {
        st.pop();
    }
    
    int top() {
        return st.top().first;
    }
    
    int getMin() {
        return st.top().second;
    }
};

/**
 * Your MinStack object will be instantiated and called as such:
 * MinStack* obj = new MinStack();
 * obj->push(val);
 * obj->pop();
 * int param_3 = obj->top();
 * int param_4 = obj->getMin();
 */
```


# 字符串解码

> 递归

输入：s = "3[a2[c]]"

输出："accaccacc"

```cpp
class Solution {
public:
    string decodeString(string s) {
        string ans;
        int len = s.size();
        if (len == 0) return "";
        if (s[0] >= '1' && s[0] <='9') {
            // 首位是数字
            int ilen = 0;
            for (int i = 0; i < len; ++i) {
                if (s[i] >= '0' && s[i] <= '9') {
                    ilen++;
                } else {
                    break;
                }
            }
            int cnt = stoi(s.substr(0, ilen));
            int sum = 0; // [ +1  ] -1
            int lpos = ilen;
            int rpos = -1;
            for (int  i = ilen; i < len; ++i) {
                if (s[i] == '[') sum++;
                else if (s[i] == ']')sum--;
                if (sum == 0) {
                    rpos = i;
                    break;
                }
            }
            string k = decodeString(s.substr(lpos + 1, rpos - 1 - lpos)); // 递归花括号内的子问题
            for (int i = 0; i < cnt; ++i) ans += k;  
            ans += decodeString(s.substr(rpos + 1)); //递归后边子问题
        } else if (s[0] >= 'a' && s[0] <= 'z') {
            // 首位是英文字母
            ans += s[0];
            ans += decodeString(s.substr(1)); // 递归后面的子问题
        }
        return ans;
    }
};
```


# 每日温度

给定一个整数数组 temperatures ，表示每天的温度，返回一个数组 answer ，其中 answer[i] 是指对于第 i 天，下一个更高温度出现在几天后。如果气温在这之后都不会升高，请在该位置用 0 来代替。

> 单调栈

```cpp
class Solution {
public:
    vector<int> dailyTemperatures(vector<int>& temperatures) {
        vector<int>ans;
        stack<int>st;
        int sz = temperatures.size();
        for (int i = sz - 1; i >= 0; i--) {
            while (!st.empty() && (temperatures[st.top()] <= temperatures[i])) {
                st.pop();
            }
            if (!st.empty()) {
                ans.push_back(st.top() - i);
            } else {
                ans.push_back(0);
            }
            st.push(i);
        }
        ranges::reverse(ans);
        return ans;
    }
};
```


# 柱状图中最大的矩形

> 单调栈 两次遍历 找左右两边的第一个小于当前元素的索引

```cpp
class Solution {
public:
    int largestRectangleArea(vector<int>& heights) {
        int ans = 0;
        stack<int>st;
        int sz = heights.size();
        vector<int>pre(sz, 0);
        
        for (int i = 0; i < sz; ++i) {
            while (!st.empty() && heights[st.top()] >= heights[i]) {
                st.pop();
            }
            pre[i] = (st.empty() ? -1 : st.top());
            st.push(i);
        }
        while (!st.empty())st.pop();
        for (int i = sz - 1; i >= 0; --i) {
            while (!st.empty() && heights[st.top()] >= heights[i]) {
                st.pop();
            }
            int val = (st.empty() ? sz : st.top());
            // cout << pre[i] << " " << val << endl;
            ans = max(ans, (val - pre[i] - 1) * heights[i]);
            st.push(i);
        }
        return ans;
    }
};
```


# 

> 

```cpp

```