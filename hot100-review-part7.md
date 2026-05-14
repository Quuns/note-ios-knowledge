# **快速排序**

> 分治治之，将数组分为左右两个部分，左部分小于等于基准，右部分大于基准，递归排序左右部分，最后合并。

```cpp
class Solution {
public:
    void quickSort(vector<int>& nums, int l, int r) {
        if (l >= r) return;
        int pivot = nums[l + rand() % (r - l + 1)];
        int L = l - 1, R = r + 1, i = l;
        while (i < R) {
            if (nums[i] < pivot) swap(nums[++L], nums[i++]);
            else if (nums[i] > pivot) swap(nums[--R], nums[i]);
            else i++;
        }
        quickSort(nums, l, L);
        quickSort(nums, R, r);
    }
};
```



# 只出现一次的数字

> 异或

```cpp
class Solution {
public:
    int singleNumber(vector<int>& nums) {
        // 异或
        int x = 0;
        for (int i : nums) {
            x ^= i;
        }
        return x;
    }
};
```


# 多数元素

给定一个大小为 n 的数组 nums ，返回其中的多数元素。多数元素是指在数组中出现次数 大于 ⌊ n/2 ⌋ 的元素。

你可以假设数组是非空的，并且给定的数组总是存在多数元素。

> 投票法

```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        int sz = nums.size();
        int mx = nums[0];
        int cnt = 1;
        for (int i = 1; i < sz; ++i) {
            if (mx == nums[i]) {
                cnt ++;
            } else {
                cnt --;
                if (cnt == 0) {
                    cnt = 1;
                    mx = nums[i];
                }
            }
        }
        return mx;
    }
};
```


# **颜色分类**

给定一个包含红色、白色和蓝色、共 n 个元素的数组 nums ，原地 对它们进行排序，使得相同颜色的元素相邻，并按照红色、白色、蓝色顺序排列。

我们使用整数 0、 1 和 2 分别表示红色、白色和蓝色。

必须在不使用库内置的 sort 函数的情况下解决这个问题。

> 三路快排

```cpp
class Solution {
public:
    void sortColors(vector<int>& nums) {
        int n = nums.size();
        int pt = 1;
        int L = -1, R = n;
        int index = 0;
        while (index < R) {
            if (nums[index] == pt) {
                index++;
            } else if (nums[index] < pt) {
                swap(nums[++L], nums[index]);
                index++;
            } else {
                swap(nums[--R], nums[index]);
            }
        }
    }
};
```


# **下一个排列**

> 从右到左找第一个非递减元素，交换其与右侧大于其的最小元素，翻转右侧递减序列

[1,2,5,4,3] -> [1,3,2,4,5]

```cpp
class Solution {
public:
    void nextPermutation(vector<int>& nums) {
        // 从右到左找到第一个x 满足 x 右边有比x大的数 （性质保证x 右边是个递减序列，所以找到的x是从右往左的第一个 num[i] < num[i+1]）
        int pos = -1;
        int sz = nums.size();
        for (int i = sz - 2; i >= 0; i--) {
            if (nums[i] < nums[i+1]) {
                pos = i;
                break;
            }
        }
        if (pos == -1) {
            // 当前就是一个递减序列，边界情况 【3，2，1】 直接翻转就好
            reverse(nums.begin(), nums.end());
            return ;
        }
        cout << pos << endl;
        // 在右侧从右往左 找到大于x = num[pos] 的最小的数的位置j，交换j 和 pos，交换后右侧还是递减序列） 这一步是让pos在增大的前提下尽可能小
        for (int i = sz - 1; i >= pos + 1; i--) {
            if (nums[i] > nums[pos]) {
                swap(nums[i], nums[pos]);
                break;
            }    
        }

        // 翻转右侧递减序列, 
        reverse(nums.begin() + pos + 1, nums.end());
    }
};
```


# 寻找重复数

给定一个包含 n + 1 个整数的数组 nums ，其数字都在 [1, n] 范围内（包括 1 和 n），可知至少存在一个重复的整数。

假设 nums 只有 一个重复的整数 ，返回 这个重复的数 。

你设计的解决方案必须 不修改 数组 nums 且只用常量级 O(1) 的额外空间。

输入：nums = [1,3,4,2,2]

输出：2

> 快慢指针 + 环形链表建图

```cpp
class Solution {
public:
    int findDuplicate(vector<int>& nums) {
        int slow = 0, fast = 0;
        while (true) {
            slow = nums[slow];
            fast = nums[nums[fast]];
            if (slow == fast) {
                // 环中的相遇节点
                break;
            }
        }
        int head = 0; // 起点出发
        while (slow != head) {
            slow = nums[slow];
            head = nums[head];
        }
        return slow; // 相遇后入环口即重复元素
    }
};
```