+++
title = "CPU CACHE的前前后后"
# description  = "CPU CACHE的前前后后"
tags = [ "CPU" ]
date = "2014-09-14"
categories = [
    "COMPUTER SYSTEM",
]
+++

CPU Cache 即高速缓存，在计算机系统的存储器层次结构中位于 CPU 寄存器与内存之间，其存在是为了缓解 CPU 寄存器与主存之间逐渐增大的速度差异。CPU Cache 再按照具体的层次细分，常见的有 L1 高速缓存、L2高度缓存和L3高度缓存三层，L1在存储器的层次结构中离CPU 寄存器最近，速度最快，造价最高，容量也最小，L2、L3分别次之。

CPU Cache 按照其保存的数据类型，又可分为只保存指令的 i-cache、只保存程序数据的 d-cache 和既保存指令又保存数据的统一的高速缓存（unified cache）。例如在一种常见的 CPU 架构中，L1 层有 L1 d-cache 和 L2 i-cache 两种不同的用途的 Cache，L2、L3 均为统一的高速缓存。

在多核机器中，每个 CPU 芯片的核心可以有自己私有的高速缓存，也可以有和其他核心共享的高速缓存。例如在 Intel Core i7 处理器上，每个核心有自己私有的 L1 i-cache、L1 d-cache 和 L2 统一的高速缓存，而 L3 统一的高速缓存是所有核共享的。

# Cache 的内部结构
为了高速地利用和管理 Cache，Cache 的内部会被组织成多个高速缓存组（cache set）；每个组包含若干个高速缓存行（cache line），cache line 是 cache 缓存、置换的最小单位，每个 cache line 包含实际存储数据的数据块（block）和一些用来管理 cache 的有效位、标记位。
![](http://7xawiv.com1.z0.glb.clouddn.com/cpu-cache1.png)

如上图所示，高速缓存被组成成了 S = 2的s次方 个组；每个高速缓存行有 1 位有效位（valid bit）指明这个行是否有意义、有 t 位标记位（tag bit）和 B = 2b 字节的数据块组成。cache 的总容量就等于 S、E、B 三者的乘积，而 s + b + t 恰好等于存储器地址的长度。

当需要根据某个地址从 cache 访问某一个字时，会把这个地址划分为 t 位的标记、s 位的组索引和 b 位的块偏移三部分，查找过程如下：   
1. 根据组索引查找到对应的组；  
2. 根据标记位找到在组中的哪一行，并检查有效位；  
3. 根据块偏移在行内的数据块中定位到这个字。  

# 在 Linux 系统中查看 CPU Cache 的相关信息
高速缓存的相关信息可以在 Linux 系统中`/sys/devices/system/cpu/cpu{i}/cache` 下的相关文件里获取到，i 位 CPU 核心的编号，一个核心会有多个 cache，会组织成名为 index{i} 的多个目录。以 cpu0 的编号为 1 的 cache 为例：
![](http://7xawiv.com1.z0.glb.clouddn.com/cpu-cache2.png)
其中的 level 文件表示 cache 的层次，内容为 1 表示 L1，依次类推；     
number_of_sets 文件里的值表示有多少个组，即 S；coherency_line_size 表示一个 cache line 的数据块大小，即 B；ways_of_associativity 表示每组有多少行，即 E；size 文件表示 cache 的容量。可以验证前三个文件中的值的乘积就等于 size 文件中的值。
![](http://7xawiv.com1.z0.glb.clouddn.com/cpu-cache3.png)

# 缓存命中与局部性
如果 CPU Cache 中缓存了要访问的数据，那么就是缓存命中（cache hit）；    
反之则称作缓存不命中。缓存不命中一般可能会由以下几个原因导致：    

- 冷不命中（cold miss）：例如某个 cache line 是空的，通常是短暂的，会在对存储器的反复访问后得到修复，这一过程也被称作缓存暖身（warmed up）；     
- 冲突不命中（conflict miss）：这种情况一般是间隔访问映射到同一个缓存块的不同存储器位置；    
- 容量不命中（capacity miss）：简单来说就是缓存太小了，不足以容纳某个工作集（working set）。   

利用局部性的思想有助于降低缓存不命中率，编写高速缓存友好的代码。    
程序的局部性可以分为时间局部性和空间局部性：   
1）时间局部性指同一数据对象尽可能地被多次使用，这样一旦缓存后，对同一数据对象后面的一系列访问都可以命中，有效地利用了 Cache 加速了访问；    
2）空间局部性指连续访问的数据对象在存储器物理上的分布是相邻或紧邻的。例如一个 cache line 一般有多个数据块，当我们访问某一个数据块时，会把这个数据块连着其后的数个数据块一起缓存起来，这样如果接下来要访问下一个相邻的数据块时，它已经在缓存中了，即访问命中。    

一旦从存储器中读取了一个数据对象，尽可能多地使用它，从而使得程序中的时间局部性最大；   
通过按照数据对象在存储器中的顺序、以步长为 1 的来访问数据，可以使得程序中的空间局部性最大。   

# 伪共享问题
False sharing，伪共享是 Cache 在多核处理器上的一个经典问题。   
简而言之是多个核心对同一个 cache line 的竞争。
为了保证 cache 的一致性，处理器会使用类似 MESI 这样的协议，
当一个核心修改某个数据时，如果其他核心也有这个数据的 cache，就把其他核心上的这个数据的缓存置为无效。

例如运行在核心 1 上的线程循环访问 cache line 上的 block A，
核心2上的线程循环访问并修改同一个 cache line 上的 block B，会反复造成彼此核心上的 cache 失效，反复造成 cache miss。   

伪共享的解决办法一般是 pading。  

wikipedia 上一段示例伪共享的代码：
```cpp
struct foo {
    int x;
    int y;
};
static struct foo f;
/* The two following functions are running concurrently: */
int sum_a(void)
{
    int s = 0;
    int i;
    for (i = 0; i < 1000000; ++i)
        s += f.x;
    return s;
}
void inc_b(void)
{
    int i;
    for (i = 0; i < 1000000; ++i)
        ++f.y;
}

```

# CPU Cache Profiling 信息的收集
在 Linux 下可以收集到程序运行过程中 Cache 的命中、不命中率、次数等指标的统计信息，以作为性能调优的参考数据。高版本的内核可以使用 Perf 工具，低版本内核可以使用 Oprofile。


**参考资料：《深入理解计算机操作系统》**



