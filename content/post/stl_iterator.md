+++
title = "STL中迭代器失效规则及遍历删除问题"
# description = "STL中迭代器失效规则及遍历删除问题"
tags = [ "C++", "STL" ]
date = "2014-06-25"
categories = [
    "Development",
    "C/C++"
]
slug = "stl-iterator-invalidation-and-traversal-remove"
+++

主要讨论 STL 中几种容器的插入和删除操作可能引起的迭代器失效问题。       
本质在于 vector、list、deque、map、set 这几种容器的底层实现机制。

vector 基于连续内存，对于其之上的插入操作分两种情况：       
1. 若插入未引起扩容行为，则插入点之前的迭代器不会失效，插入点之后的因后移操作而失效；                
2. 插入操作引起扩容行为，则所有迭代器均失效；                 
对于其删除行为，由于前移行为，删除点之后的迭代器会失效。                     


list 基于节点，对于其之上的插入操作，所有迭代器都不会受到影响；                   
删除行为则只会导致被删除节点的迭代器失效。

deque 基于分段连续内存，对于其之上的插入操作会导致所有的迭代器都失效。                     
删除操作分两种情况：              
1）从头部或尾部删除：只会导致被删除位置的迭代器失效；                 
2）删除其他位置：所有的迭代器都会失效。             

map 和 set 都属于关联容器，其底层实现基于二叉搜索树（红黑树）。             
对于插入行为，所有迭代器都不会受影响；删除只会影响被删除位置的迭代器。             

另一问题：如何遍历容器中的每个元素，删除所有符合某个条件的元素？                
例如删除所有元素值为某个特定值的元素，或者删除所有值为偶数的元素。               

对于 vector 和 deque 基于连续内存的容器，使用 earse-remove 方法：              

```cpp
bool isEven(int a) {
    return a % 2 == 0;
}

int main()
{
    vector<int> vec({1, 2, 3, 4, 5, 6, 5, 5}); // c++11 list-initialization

    // 删除所有值为5的元素
    vec.erase(remove(vec.begin(), vec.end(), 5),
            vec.end());
    // 删除偶数元素
    vec.erase(remove_if(vec.begin(), vec.end(), isEven),
            vec.end());
}
```

对于 list，使用其自身提供的删除成员函数：

```cpp
bool isEven(int a) {
    return a % 2 == 0;
}

int main()
{
    list<int> lst({1, 2, 3, 4, 5, 6, 5, 5});
    lst.remove(5);
    lst.remove_if(isEven);
}
```

对于 map 和 set，使用遍历时迭代器后置自增的方式：

```cpp
bool isEven(int a) {
    return a % 2 == 0;
}

int main()
{
    set<int> s({1, 2, 3, 4, 5, 6, 5, 5});

    for (auto it = s.begin(); it != s.end(); ) {
        if (*it == 5) // or isEven(*it)
            s.erase(it++);
        else
            ++it;
    }
}
```

这种方法也同样适用于 list。另外 vector、deque、list 三种容器也可以使用类似的方式。
唯一的不同是在循环中当符合条件时，迭代器的更新使用 erase 成员函数的返回值，而不是后置自增。    

总的来说，相比较而言，vecot、deque、list 的删除更为灵活。   
       
参考资料：《Effective STL》      
