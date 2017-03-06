+++
title = "LEVELDB中LRU的实现"
# description = "LEVELDB中LRU的实现"
tags = [ "C++", "Algorithm" ]
date = "2014-09-13"
categories = [
    "Development",
    "C/C++"
]
+++

LRU即Least Recently Used（最久未使用），是一种常用的页面或Cache置换算法。

其需要对外提供的接口也颇为简单，一般会有两个接口：    
1. 一个 get 操作：用于获取某个 key 对应的 value；     
2. 一个 set 操作：用于设置某个 key 对应的 value。    
remove 操作一般在元素满的时候自动触发。

对元素的访问操作（set 和 get）会引起其排序提升，以降低其被替换出去的可能性。可以看出以链表来实现是最合适的，元素访问时只需要删除对应节点并把节点放到链表头部完成提升；而当链表中元素个数达到其容量时，只需要删除链表尾部的元素；在链表中这两个操作都是可以做到O(1)的。

另一方面对链表中元素的查找操作是O(n)的，所以一般会以哈希表做为辅助，哈希表中存放每个 key 对应的元素在链表中的 handle（指针、迭代器等），以实现在O(1)的时间内完成对元素的访问。同时也因为是链表，对一个 key 的元素插入和删除操作不会影响哈希表中其他 key 的 handle。

所以 LRU 算法的实现一般是 Double-linked List + Hash Table，Leveldb 中的实现也正是如此。

首先是 handle 即链表中节点的结构，包含必须的前继、后继、数据等必须的成员和一些辅助的成员：
```cpp
struct LRUHandle {
    void* value;            // 节点数据
    void (*deleter)(const Slice&, void* value);    // 定制节点释放函数指针
    LRUHandle* next_hash;   // hashtable内部冲突排解链的单链表后继
    LRUHandle* next;        // 双链表的后继节点
    LRUHandle* prev;        // 双链表的前继节点
    size_t charge;
    size_t key_length;      // key 的长度
    uint32_t refs;          // 引用计数，小于等于0后，调用 deleter 释放节点
    uint32_t hash;          // hash 值
    char key_data[1];       // key 的数据，类同 char *，只是因为实现上的方便

    // 返回key，Slice可以看作类似string的结构
    Slice key() const {
        if (next == this) {
            return *(reinterpret_cast<Slice*>(value));
        } else {
            return Slice(key_data, key_length);
        }
    }
};
```

leveldb 中 hashtable 的实现是基于数组＋链表的，即使用链表来排解冲突。
数组的秩为key的 hash，数组的元素为一个单链表，单链表的节点就是 LRUHandle，以 next_hash 成员串联起来。其中 key 的 hash 值的计算没有通过某个具体的方法来每次计算，而是通过直接由外部给出的方式。
```cpp
class HandleTable {
public:
    // 根据 key 和 hash 值查找 handle
    LRUHandle* Lookup(const Slice& key, uint32_t hash);  
    // 插入 handle
    LRUHandle* Insert(LRUHandle* h);
    // 删除 handle
    LRUHandle* Remove(const Slice& key, uint32_t hash);
private:
    uint32_t length_;   // list_数组的长度
    uint32_t elems_;    // 表中handles的个数
    LRUHandle** list_;  // 冲突排解单链表链的数组，数组元素为 LRUHanle*
};
```
最终的 LRUCache 的实现就是基于上面的 HandleTable 和双链表来实现的。
```cpp
class LRUCache {
public:
    // handle的插入，类似set操作
    Cache::Handle* Insert(const Slice& key, uint32_t hash,
            void* value, size_t charge,
            void (*deleter)(const Slice& key, void* value));

    // handle的查找，类似get操作
    Cache::Handle* Lookup(const Slice& key, uint32_t hash);

    // 根据 handle 直接从Cache中删除
    void Release(Cache::Handle* handle);

    // 查找并删除
    void Erase(const Slice& key, uint32_t hash);
private:
    size_t capacity_;    // cache 的容量
    RUHandle lru_;       // 双链表
    HandleTable table_;  // hashtable
};
```
