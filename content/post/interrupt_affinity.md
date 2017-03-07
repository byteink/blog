+++
title = "LINUX网卡软中断设置小结"
# description   = "LINUX网卡软中断设置"
tags = [ "LINUX","NETWORK" ]
date = "2014-03-04"
categories = [
    "Development",
]
+++
## 意义
通过设置CPU的中断亲和性，在多核心下平衡系统中断负载，   
避免因单个CPU的 si 过高而导致的系统瓶颈。   
## 设置方法
### 1. 关闭irqbalance   
```
sudo service irqbalance stop
```
### 2. 查看中断在各个CPU上的分布情况
![](http://7xawiv.com1.z0.glb.clouddn.com/interrupt-affinity-1.png)

### 3. 设定相应中断的CPU掩码
![](http://7xawiv.com1.z0.glb.clouddn.com/interrupt-affinity-2.png)
2表示分配到CPU1，第二个逻辑核心。   
CPU掩码：第n个二进制位为1，则表示分配到第n个CPU。   

设置完成后，可以看到CPU1对应的中断数在不断增加。   
![](http://7xawiv.com1.z0.glb.clouddn.com/interrupt-affinity-3.png)
各个CPU的负载:
![](http://7xawiv.com1.z0.glb.clouddn.com/interrupt-affinity-4.png)
可以看到 si 成功地转移到CPU1。

## 小结
设置是否有效，会有内核版本和网卡工作方式的限制，一般至少要求内核版本为2.6.32及以上。   
单队列网卡不能直接通过smp_affinity设置，但可以通过RPS模拟。  
多队列网卡最好一个中断号绑定到一个固定的核心，以避免cache miss和资源竞争的副作用。   

## 参考资料
1. [深度剖析告诉你irqbalance有用吗？](http://blog.yufeng.info/archives/2422)
2. [记录一个软中断问题](http://huoding.com/2013/10/30/296)
3. [Why interrupt affinity with multiple cores is not such a good thing](http://www.alexonlinux.com/why-interrupt-affinity-with-multiple-cores-is-not-such-a-good-thing)
