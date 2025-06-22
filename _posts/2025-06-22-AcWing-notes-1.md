---
title: "AcWing789.数的范围做题笔记——二分"
date: 2025-06-22 3:16:00 +0800
categories: [算法笔记]
tags: [AcWing, 二分, 算法]
---

## 原题
给定一个按照升序排列的长度为n的整数数组，以及q个查询。

对于每个查询，返回一个元素 k
 的起始位置和终止位置（位置从 0
 开始计数）。

如果数组中不存在该元素，则返回 `-1 -1`。

输入格式
第一行包含整数 n
 和 q
，表示数组长度和询问个数。

第二行包含 n
 个整数（均在 1∼10000
 范围内），表示完整数组。

接下来 q
 行，每行包含一个整数 k
，表示一个询问元素。

输出格式
共 q
 行，每行包含两个整数，表示所求元素的起始位置和终止位置。

如果数组中不存在该元素，则返回 `-1 -1`。

数据范围
1≤n≤100000

1≤q≤10000

1≤k≤10000

输入样例：
```
6 3
1 2 2 3 3 4
3
4
5
```
输出样例：
```
3 4
5 5
-1 -1
```
## 题解
```cpp
#include <bits/stdc++.h>
using namespace std;
const int N = 100010;
int n, q, k;
int arr[N];
int main(){
    int n;
    cin >> n >> q;
    for(int i = 0; i < n; i ++){
        cin >> arr[i];
    }
    for(int i = 0; i < q; i ++){
        cin >> k;
        int l = 0, r = n - 1;
        while(l < r){
            int mid = l + r >> 1;
            if(arr[mid] >= k) r = mid;
            else l = mid + 1;
        }
        if(arr[l] != k) cout << "-1 -1" << endl;
        else{
            cout << l << ' ';
            l = 0, r = n - 1;
            while(l < r){
                int mid = l + r + 1 >> 1;
                if(arr[mid] <= k) l = mid;
                else r = mid - 1;
            }
            cout << l << endl;
        }
    }
    
    return 0;
}
```
## 关键点
### 第一次二分：查找起始位置（第一个 ≥ x 的位置）
```cpp
int l = 0, r = n - 1;
while (l < r) {
    int mid = l + r >> 1;  // 向下取整
    if (q[mid] >= x) r = mid;
    else l = mid + 1;
}
// 结束时 l == r，即起始位置
```
**收缩方向**：

当 q[mid] >= x 时，r=mid：向左收缩（因为左边可能有更小的 ≥x 的值，直到探出左边界）

当 q[mid] < x 时，l=mid+1：向右跳跃（当前值太小，必须跳过）

**边界情况**：

若所有元素 < x：l 停在 n-1（需后续检查）

若所有元素 ≥ x：l 停在 0（最小位置）

### 第二次二分：查找结束位置（最后一个 ≤ x 的位置）
```cpp
int left = l;  // 从起始位置开始
int right = n - 1;
while (left < right) {
    int mid = left + right + 1 >> 1;  // 向上取整
    if (q[mid] <= x) left = mid;
    else right = mid - 1;
}
// 结束时 left == right，即结束位置
```
**收缩方向**：

当 q[mid] <= x 时，left=mid：向右试探（可能还有更大的 ≤x 的值，直到探出右边界）

当 q[mid] > x 时，right=mid-1：向左收缩（当前值太大，必须跳过）

**向上取整**：

mid = (left + right + 1)/2 防止死循环

当区间长度为2时（如 left=1, right=2）：

向下取整：mid=1 → 若执行 left=mid 则区间不变

向上取整：mid=2 → 可安全收缩区间
