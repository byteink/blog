+++
title = "UNIX 网络编程中SOCKET错误小结"
tags = [ "LINUX", "NETWORK" ]
date = "2012-12-05"
categories = [
    "Development",
    "Network"
]
slug = "unp-socket-error"
+++


## UDP
### write
- 写一个数据报大小大于发送缓冲区 EMSGSIZE     
- 数据链路层输出队列空间不足 ENOBUFS          

## TCP
### 产生RST的三个条件:
- 目的地为某端口的SYN到达，但服务器没有在监听
- TCP想取消一个已有链接
- TCP收到一个不存在的连接上的分节           

### connet
- 客户未收到SYN的响应，规定时间内重发SYN，失败返回 ETIMEDOUT      
- 对客户SYN的响应是RST，表示在指定的端口上没有进程在监听，马上返回 ECONNREFUSED       
- 客户的SYN在中间路由器引发“destination unreachable”ICMP错误， 规定时间内重发SYN，失败返回 EHOSTUNREACH 或 ENETUNREACH            

### listen
- 客户SYN到达，listen队列是满的，TCP就忽略该分节，不发送RST

### accpet
- 三路握手完成，调用accpet之前，收到客户的RST： accept返回 ECONNABORTED 错误

### write
- 向一个已收到RST的套接字写操作，内核发送SIGPIPE信号，写操作返回 EPIPE        
- 服务器崩溃，规定时间内重传，失败后返回 ETIMEOUT，若中间路由判定主机不可达，返回 EHOSTUNREACH 或 ENETUNREACH     
- 写半部已关闭，EPIPE     

### read
- 非阻塞，无数据 EAGAIN  or  EWOULDBLOCK
