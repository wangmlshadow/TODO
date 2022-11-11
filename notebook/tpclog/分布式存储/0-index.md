# **透彻理解分布式存储系列**

分布式存储，本质就是在很多台机器上通过分布式的手段来存储一些数据，分布式存储的类型非常多，比如分布式数据库，NoSQL数据库，HBase等等都属于分布式存储的范畴。

本系列，我主要讲解的是 **分布式文件系统** ，比如HDFS就是一类典型的分布式文件系统，客户端可以使用HDFS存储超大的文件（比如1TB），HDFS会自动把大文件分布式存储在各个机器上，每台机器上就存储几百MB的数据。分布式文件系统负责管理文件元数据和分散在各台机器上的文件，对于客户端来说，感觉就是像面向一个文件在操作。

其它的分布式文件系统还有 *FastDFS* ——一套基于C语言开发的分布式文件系统，你可以基于FastDFS来构建一个分布式文件系统集群。但是FastDFS的社区并不太活跃，并且有一些Bug和坑，基于C语言编写也不便于研究源码和二次开发。

所以，本系列，我将带领大家从0到1基于Java实现一套分布式文件系统，学习完本系列，你将会对Java NIO以及分布式存储的底层原理有非常深刻的认识。

本系列包含以下章节：

* [透彻理解分布式存储（一）——整体架构](https://www.tpvlog.com/article/319)
* [透彻理解分布式存储（二）——gRPC使用](https://www.tpvlog.com/article/320)
* [透彻理解分布式存储（三）——DataNode注册与心跳](https://www.tpvlog.com/article/321)
* [透彻理解分布式存储（四）——dfs客户端工程](https://www.tpvlog.com/article/322)
* [透彻理解分布式存储（五）——内存文件目录树](https://www.tpvlog.com/article/323)
* [透彻理解分布式存储（六）——写Edits Log日志](https://www.tpvlog.com/article/324)
* [透彻理解分布式存储（七）——同步Edits Log日志](https://www.tpvlog.com/article/325)
* [透彻理解分布式存储（八）——checkpoint机制](https://www.tpvlog.com/article/326)
* [透彻理解分布式存储（九）——fsimage传输与宕机恢复](https://www.tpvlog.com/article/327)
* [透彻理解分布式存储（十）——BackupNode宕机恢复](https://www.tpvlog.com/article/328)
* [透彻理解分布式存储（十一）——文件存储：整体架构](https://www.tpvlog.com/article/329)
* [透彻理解分布式存储（十二）——文件存储：NIO网络通信](https://www.tpvlog.com/article/330)
* [透彻理解分布式存储（十三）——文件存储：DataNode信息上报](https://www.tpvlog.com/article/331)
* [透彻理解分布式存储（十四）——高可用架构：文件副本重分配](https://www.tpvlog.com/article/332)
* [透彻理解分布式存储（十五）——高可用架构：文件传输中断处理](https://www.tpvlog.com/article/333)
* [透彻理解分布式存储（十六）——可扩展架构：Rebalance](https://www.tpvlog.com/article/334)
* [透彻理解分布式存储（十七）——高并发架构：Reactor模式](https://www.tpvlog.com/article/335)
* [透彻理解分布式存储（十八）——高性能架构：长连接与异步机制](https://www.tpvlog.com/article/336)
