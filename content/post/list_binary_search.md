+++
title = "为什么不要对STD::LIST使用二分搜索"
# description = "为什么不要对STD::LIST使用二分搜索"
tags = [ "C++", "Algorithm", "Binary Search" ]
date = "2013-07-18"
categories = [
    "Algorithm",
    "C/C++",
]
+++
一篇讨论std::list与binary_search的文章：     
[Why std::binary_search of std::list Works,But You Shouldn't Use It!](http://www.umich.edu/~eecs381/handouts/binary_search_std_list.pdf)    


如文中所述，STL中二分搜索的大概实现是这样的（以`lower_bound`为例）：
```cpp
template <class ForwardIterator, class T>
ForwardIterator lower_bound(ForwardIterator first,
 ForwardIterator last, const T& value)
{
    typedef typename iterator_traits<ForwardIterator>::difference_type difference_type;
    difference_type len = distance(first, last);
    while (len > 0)
    {
        ForwardIterator i = first;
        difference_type len2 = len / 2;
        advance(i, len2);
        if (*i < value)
        {
            first = ++i;
            len -= len2 + 1;
        }
        else
            len = len2;
    }
    return first;
}
```
算法的主要耗时操作是distance、advance和比较操作。  

对于比较操作，无论 vector 还是 list ，线性搜索的最坏时间复杂度都是O(n)，二分搜索为O(logn)；   

而distance操作，对于 vector 这类容器支持随机访问的迭代器，只是类似指针加减一样的算术操作，时间复杂度为O(1)， 而 list 不支持随机访问，其distance操作，是从到到尾逐一访问、累加长度，时间复杂度为O(n)；  

advance操作类似distance，对于链表其时间复杂度为一次循环中搜索区间长度的一半，而这样的advance操作多达log(n)次。  

可以看出来，对于链表来说，二分搜索的比较操作要比线性搜索少（O(logn) vs O(n) ）。  
但对链表元素的访问操作却要比普通的线性搜索多出来很多。  
### 一个实验
一段用来比较std::list之上线性搜索与二分搜索效率的代码：
```cpp
#include <sys/time.h>
#include <list>
#include <iostream>
#include <algorithm>

#define BENCHMARK_COUNT (5000*10000)

void print_elapse(timeval& start, timeval& end)
{
    timeval res;
    timersub(&end, &start, &res);
    std::cout << res.tv_sec << "s " <<
        res.tv_usec / 1000 << "ms." << std::endl;
}

void compare_search(const std::list<int>& lst, int target)
{
    std::cout << "target: " << target << std::endl;

    timeval start, end;
    gettimeofday(&start, NULL);
    binary_search(lst.begin(), lst.end(), target);
    gettimeofday(&end, NULL);
    std::cout << "binary search: ";
    print_elapse(start, end);
    gettimeofday(&start, NULL);
    for (std::list<int>::const_iterator it = lst.begin();
             it != lst.end(); ++it) {
        if (*it == target) break;
    }
    gettimeofday(&end, NULL);
    std::cout << "linear search: ";
    print_elapse(start, end);
    std::cout << std::endl;
}

int main(int argc, char *argv[])
{
    std::list<int> lst;
    for (int i = 0; i < BENCHMARK_COUNT; ++i)
        lst.push_back(i);
    compare_search(lst, BENCHMARK_COUNT - 1);
    compare_search(lst, BENCHMARK_COUNT / 2);
    compare_search(lst, BENCHMARK_COUNT / 5);
    return 0;
}
```
编译，优化级别为O2，运行结果：
![](http://7xawiv.com1.z0.glb.clouddn.com/list-binary-search.jpg)
可以看出即使是在线性搜索的最坏情况下（目标为最后一个元素或者不存在），   
链表的线性搜索也要比二分搜索快得多！

究其原因，除了上述复杂度的分析外，顺序查找对缓存更为友好这也是很重要的一方面。

