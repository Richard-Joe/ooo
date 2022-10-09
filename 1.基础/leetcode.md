# 编程技巧

## 二分查找

```c
#define len 10
int arr[len] = {1, 7, 19, 48, 48, 48, 48, 50, 63, 100};

// 获取大于target的最小索引
int getMinGreaterIndex(int target) {
    int left = 0, right = len;
    while (left < right) {
        int mid = left + ((right - left) >> 1); 
        if (arr[mid] < target) left = mid + 1;
        else right = mid;
    }   
    while (left < len && arr[left] <= target)
        left++;
    if (left == len) return -1; 
    return left;
}

// 获取小于target的最大索引
int getMaxLessterIndex(int target) {
    int left = 0, right = len;
    while (left < right) {
        int mid = left + ((right - left) >> 1); 
        if (arr[mid] >= target) right = mid - 1;
        else left = mid + 1;
    }   
    while (right >= 0 && arr[right] >= target)
        right--;
    if (right < 0) return -1; 
    return right;
}
```
