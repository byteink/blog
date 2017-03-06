+++
title = "线程安全的单例模式"
# description  = "线程安全的单例模式"
tags = [ "C++", "Boost","CONCURRENCY" ]
date = "2014-10-22"
categories = [
    "Development",
    "C/C++"
]
+++

# 一、使用pthread_once
实现起来相对容易，弊端也很明显：不具备可移植性。

```cpp
pthread_once_t once_control = PTHREAD_ONCE_INIT;

template <class T>
class Singleton {
public:
     static T* instance() {
         pthread_once(&once_control, &Singleton::init);
         return instance_;
     };

 private:
     static void init() {
         instance_ = new T();
     }

 private:
     static T* instance_;
};

template <class T>
T* Singleton<T>::instance_ = NULL;
```

# 二、著名的DCLP
Double-Checked Locking Pattern，双重检查锁定模式。   
这种方法使用了锁来保证在多线程环境下可以正常工作，   
同时使用了两次检查的方式进行了优化，使得在绝大多数情况下并不需要访问锁。   
一种很常见的实现方式：  
```cpp
class Singleton {
public:
    static T* instance() {
        if (NULL == instance_) { // 第一次检查
            Lock lock; // 伪代码
            if (NULL == instance_)  // 第二次检查
                instance_ = new T;
        }
        return instance_;
    };

private:
    static T* instance_;
};

template <class T>
T* Singleton<T>::instance_ = NULL;
```
这种看起来正确且效率足够高的实现方式，当考虑到编译器的优化时，实际上是有问题的。   

分析代码的第8行，这一行 new 的代码至少包含了三个操作：   
1. 为 class T 分配内存   
2. 在分配的内存上调用 class T 的构造函数  
3. 内存地址赋值给 instance_    
其中操作2、操作3，是有可能被编译器优化乱序的，即先执行操作3，再执行操作2。    
考虑线程 A 执行了操作1、操作3后时间片结束进入休眠，此时 instance_ 并不为 NULL；   
线程 B 也进入了 instance() 函数中，返回 instance_，但此时操作2却并未被执行到，   
因此线程 B 也就得到了一个未执行构造函数的对象指针，其行为是未定义的。   

由于旧标准中缺乏对代码执行次序的约束功能和 memory order 的概念，  
因此也决定了在 C++ 11 以前无法实现可移植的线程安全单例模式。  

一种可行的方法是利用 boost 的 atomic 库：

```cpp
template <class T>
class Singleton {
public:
    static T * instance()
    {
        T * tmp = instance_.load(boost::memory_order_consume);
        if (!tmp) {
            boost::mutex::scoped_lock lock(mutex_);
            tmp = instance_.load(boost::memory_order_consume);
            if (!tmp) {
                tmp = new T;
                instance_.store(tmp, boost::memory_order_release);
            }
        }
        return tmp;
    }
private:
    static boost::atomic<T *> instance_;
    static boost::mutex mutex_;
};

template <class T>
boost::atomic<T *> Singleton<T>::instance_(0);

template <class T>
boost::mutex Singleton<T>::mutex_;

```

**参考资料**：《C++ and the Perils of Double-Checked Locking》，Scott Meyers and Andrei Alexandrescu



# 更新
在C++11中，返回static对象是线程安全的，这无疑是最简单的实现方式。
```cpp
template <class T>
static T& Singleton::instance(){
    static T instance;
    return instance;
}
```
