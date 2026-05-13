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

> 

```cpp

```


# 在排序数组中查找元素的第一个和最后一个位置

> 

```cpp

```


# 搜索旋转排序数组

> 

```cpp

```


# 寻找旋转排序数组中的最小值

> 

```cpp

```


# 寻找两个正序数组的中位数

> 

```cpp

```

# 

> 

```cpp

```