---
title: '[DB]多类型数据库基础'
date: 2018-12-22 15:51:40
tags: DB
---

## 常用数据库
- MySQL
    - 分布式关系型数据库
    - 作为一个独立的数据库服务器，应用程序需要与MySQL守护进程通信才能访问数据库
    - 无全文本搜索

- PostgreSQL
    - 对象关系型数据库
    - 支持面向对象及关系型数据库功能
    - 并发性能好

- SQLite
    - 嵌入式关系型数据库 并不作为一个独立的进程通过某种通信协议与应用程序通信，而是作为应用程序的一部分
    - 自包含、基于文件存储  
    - 同一时刻仅允许一个写操作

- Elastic
    - 全文搜索引擎
    - 本质上是一个分布式数据库（NoSQL）

- Cassandra 
    - 分布式数据库，高度可扩展，用于管理大量的结构化数据
    - 高可用性，没有单点故障
    - [列式数据库](https://zh.wikipedia.org/wiki/%E5%88%97%E5%BC%8F%E6%95%B0%E6%8D%AE%E5%BA%93)

- MongoDB
    - 面向文档的NoSQL数据库
 
- Redis
    - 已知最快“key-value”数据库
    - 基于内存，可设置有效时间
    - 支持事务

- Memcache  
    一个高性能的分布式的内存对象缓存系统，通过在内存里维护一个统一的巨大的hash表，它能够用来存储各种格式的数据，包括图像、视频、文件以及数据库检索的结果等。简单的说就是将数据调用到内存中，然后从内存中读取，从而大大提高读取速度。

---
- [Redis/Memcache/Mongodb 对比](http://www.cnblogs.com/94cool/p/3247307.html)
    - memecache 把数据全部存在内存之中，断电后会挂掉，数据不能超过内存大小,只支持Key-Value
    - redis有部份存在硬盘上，这样能保证数据的持久性，支持数据的持久化，支持list/set/hash

---


##Mysql
###数据引擎
- MyISAM: 适合读多写少，不支持事务型处理；（MYSQL 5.5.5之前的默认存储引擎）
- InnoDB：支持事务，行锁和外键；（目前默认引擎）
- Memory：数据都存于内存，操作快但无法持续存储；

###[索引](https://www.kancloud.cn/kancloud/theory-of-mysql-index/41844)
- [B-Tree和B+Tree](https://blog.csdn.net/v_july_v/article/details/6530142) 
    - B+Tree所有数据在叶子结点
    - B+Tree一个有m个key的节点，最多有2m个子结点；B-Tree可以有2m+1个
- R-Tree和R+Tree  
    - R-Tree矩形，R+矩形可相互覆盖
- MyISAM索引  
    - 非聚集索引
    - 主、从索引没有区别
    - 叶子结点存数据指针
- InnoDB索引
    - 聚集索引
    - 必须要有主键，且以主键建索引直接存数据
    - 从键索引存主键值
    - 每次查询必须通过主键索引查

###[索引调优](https://juejin.im/post/5a6873fbf265da3e393a97fa)
- 联合索引
    - 最左前缀匹配
    - 范围列可用到索引
- 索引建立原则
    - 索引选择性
    - 前缀索引（联合前缀索引）
- InnoDB尽量用自增索引
- 排序order by
    - 合理利用索引避免排序SQL


##Redis
Redis是一个基于内存的Key-Value型的单线程高速缓存数据库。
###数据存储
- Redis使用对象表示数据库中的键值对，一个键对象，一个值对象；
- 存储数据类型：字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets，去重）， 有序集合（sorted sets，排序、去重）；
- 持久化：Redis支持持久化功能，分为RDB持久化（数据压缩）和AOF持久化（保存写命令，有点类似日志）；
- 过期策略：构建过期字典保存所有Key的过期时间，通过定期删除（默认100ms检查一次）和惰性删除（Redis命令执行前对输入的Key进行检查）清理数据；
- 内存管理：通过`引用计数`回收内存。

###集群部署策略
满足`线性拓展`和`高可用性`，`数据一种性`存在一定损失。  

- 基本策略：将数据自动切分（split）到多个节点，当部分节点失效仍可以处理命令请求；
- 集群数据分片：
    - Redis集群使用数据分片（sharding）而非一致性哈希（consistency hashing）来实现；
    - Redis集群包含16384个哈希槽（hash slot）， CRC16(key) % 16384 来计算键 key 属于哪个槽；
    - Redis集群中每个节点负责处理一部分哈希槽；
- 节点主从复制：
    - 集群中每个节点都有1个至N个复制品，并设定其中一个为主节点；
    - Redis使用异步复制策略，并不能保证数据的强一致性；

_*扩展:[5种Redis集群部署策略](https://juejin.im/post/5a707f4d5188255a8817f5b1)_

###与DB的协同策略
Redis常与DB协同并作为DB的缓存使用，使用姿势上有几点需要注意的地方。

- 数据查询：先查Redis再查DB，Redis有数据时直接返回，无数据时用DB中查到的数据更新Redis；
- 数据删除：先删除Redis再删除DB；
- 数据插入／更新策略：
    - 先更新DB再更新Redis：流程上没问题，但高并发update时由于写入时间不可控容易出现脏数据的情况；
    - 先删除Redis再更新DB：并发读写容易出现脏数据。并发读写时容易出现当写还没有完成，Redis已删除，此时读了一条数据更新Redis为脏数据的情况；
    - 先更新DB再删除Redis：较为合理，由于DB写入较查询耗时更长，所以很难出现并发读写时误更新Redis使其与DB不一致的情况。

