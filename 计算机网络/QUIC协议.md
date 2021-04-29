https://www.cnblogs.com/imteck4713/p/11777310.html

这里有一个大的命题：如何用UDP实现一个http协议



![image-20210313215045985](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210313215045985.png)



QUIC 与现有 TCP + TLS + HTTP/2 方案相比，有以下几点主要特征：

1）利用缓存，显著减少连接建立时间；

2）改善拥塞控制，拥塞控制从内核空间到用户空间；

3）没有 head of line 阻塞的多路复用；

4）前向纠错，减少重传；

5）连接平滑迁移，网络状态的变更不会影响连接断线