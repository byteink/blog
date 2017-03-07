+++
title = "BUBBLE SORT的改进"
# description   = "bubble sort的改进"
tags = [ "Algorithm" ]
date = "2013-11-04"
categories = [
    "Development",
]
+++
最近在清华大学的mooc平台上学习邓俊辉老师的《数据结构》课程，收获很多。   
比如之前认为最为简单、最没有什么好讨论的冒泡排序，原来也有着值得让人思考的地方。   
仅仅能够实现基本的算法仍然是不够的，还需要考虑算法在某些退化情况下的优化问题。  

## 一. 算法在中途已然实现整体有序

此时已经没有执行下去的必要，应该立即退出。
判断是否已经有序的标志就是扫描某趟时已经没有逆序对（没有发生交换），   
可以定义一个sorted的flag，表示整体已有序，每趟扫描开始时置为true，有发生交换置为false。   
```cpp
void bubblesort1B (int A[], int n) {
    for (bool sorted = false; !sorted; n--) {
        for (int i = 1; i < n; i++) {
            sorted = true;
            if (A[i-1] > A[i]) {
                swap(A[i-1], A[i]);
                sorted = false;
            }
        }
    }
}
```
## 二、每趟扫描区间长度的优
我们知道在最基本的冒泡排序算法中，第k趟扫描的区间是[0, n - k)。    
然后有些情况下，这个区间后面的部分是不需要扫描的，这种情况就是这部分已然有序。   
即对尾部有序这种退化情况的优化。   
假定上一趟扫描时最后一次发生交换的两个元素为A[i]和A[i+1]，则i+1后面的元素必然已经是有序且处于其最终位置。那么下一趟的扫描仅需要执行到第i个元素即可。   
```cpp
void bubble_sort(int A[], int n) {
    for (int m = n; n > 1; m = n) {
        n = 1; // 记录最后一次逆序位置
        for (int i = 1; i < m; ++i) {
            if (A[i-1] > A[i]) {
                std::swap(A[i-1], A[i]);
                n = i;
            }
        }
    }
}
```
注意到这种处理已经包含了整体有序及时退出的退化处理（其实是子集）。
