---
title: Redis技术总结8-其他
date: 2022-02-15 15:50:35
tags:
    - 服务器
    - 客户端
    - 事件
categories:
    - 数据库
    - NoSQL
    - Redis
---

### 1.  发布与订阅

Redis提供了基于发布/订阅模式的消息机制，此种模式下，消息发布者和订阅者不直接通信。发布者客户端向指定频道发送消息，订阅该频道的客户端都可以收到该消息。

Redis的发布与订阅由PUBLISH，SUBSCRIBE，UNSUBSCRIBE，PSUBSCRIBE等命令组成。

通过SUBSCRIBE命令，客户端可以订阅一个或多个频道：

​                                   
![img](1.png)<center>*news.it频道和他的三个订阅者*</center>


![img](2.png)<center>*向news.it频道发送消息*</center>



通过PSUBSCRIBE命令可以订阅一个或多个模式，如下图：

​     
![img](3.png)<center>*频道订阅与模式订阅*</center>


 

#### 1.1  频道订阅与退订

Redis将所有频道订阅关系保存在redisServer中的pubsub_channels字典里。这个字典的键是某个被订阅的频道，而值就是一个链表，链表里记录了所有订阅这个频道的客户端。如下图：

​     
![img](4.png)<center>*pubsub_channels结构示意*</center>


 

当订阅频道时，如果频道已经在pubsub_channels中，那么只需在链表末尾加上新的客户端即可。如果频道未在pubsub_channels中，那么需要先在字典中为频道创建一个键，值为空链表，然后将客户端插入链表作为第一个元素。

当退订频道时，在pubsub_channels中找到对应的频道及链表，将链表中客户端删除。如果删除完之后链表元素个数为0，那么在pubsub_channels移除这个频道。

 

#### 1.2  模式订阅与退订

Redis将所有模式订阅关系保存在redisServer中的pubsub_patterns链表（而不是字典）里。链表中的每个元素都是一个pubsub_pattern结构，这个结构的pattern属性记录了被订阅的模式，client属性则记录了订阅模式的客户端。如下图：

​     
![img](5.png)<center>*pubsubPattern代码示意*</center>


![img](6.png)<center>*pubsub_pattern链表*</center>




当订阅模式时，新增一个pubsubPattern结构，将pattern属性设置为被订阅的模式，将client属性设置为订阅模式的客户端。然后将pubsubPattern结构添加到pubsub_patterns链表的表尾。

当退订模式时，服务器将在链表中查找并删除模式和客户端分别都匹配的pubsubPattern结构。

 

#### 1.3  发送消息

当一个Redis客户端执行PUBLISH <channel> <message>命令将消息发送给频道时，服务器需要执行以下两个动作：

**1)**    将消息message发送给channel的所有订阅者。

**2)**    将消息message发送给所有与channel匹配的模式的订阅者。

 

#### 1.4  查看订阅信息

PUBSUB命令可以查看频道和模式的订阅信息，它有三个子命令：

**1)**    PUBSUB CHANNELS [pattern]

返回服务器当前被订阅的频道。也可以根据模式pattern进行匹配返回。

**2)**    PUBSUB NUMSUB [channel1,channel2…]

返回参数中频道的订阅者数量

**3)**    PUBSUB NUMPAT

返回服务器被订阅模式的数量（即pubsub_patterns的长度）。

 

#### 1.5  关于订阅

客户端在执行订阅命令之后进入了订阅状态，只能接收PUBLISH命令发送的消息以及SUBSCRIBE，PSUBSCRIBE，UNSUBSCRIBE，PUNSUBSCRIBE四个命令。新开启的客户端，无法接收该频道之前的消息，因为Redis不会对发布的消息进行持久化。

 

 

### 2.  事务

Redis通过MULTI,EXEC,WATCH等命令实现事务功能。

事务以一个MULTI命令开始，接着将多个命令放入事务中，最后以EXEC提交给服务器结束。

​     
![img](7.png)<center>*事务示例*</center>



 

事务以MULTI命令的执行标记为开始：在redisClient中将flags属性里打开REDIS_MULTI标识。

当事务开始后，服务器会根据收到的命令的不同，执行不同的操作：如果是MULTI、WATCH、DISCARD、WATCH，那么直接执行。如果不是，则会放到事务队列中，返回给客户端QUEUED。如下图：

​     
![img](8.png)<center>*事务状态的服务器判断命令*</center>



 

每个Redis客户端都有自己的事务状态，保存在redisClient中。即multiState结构的msate属性。如下图：

​     
![img](9.png)<center>*multiState代码示意*</center>


​     
![img](10.png)<center>*multiCmd代码示意*</center>


 
![img](11.png)<center>*事务状态*</center>

​     


EXEC命令会遍历事务队列，执行事务队列中的所有命令，最后将执行结果全部返回给客户端。

DISCARD命令会放弃事务，清空事务队列。

WATCH命令是一个乐观锁，他会在EXE命令执行之前，监视若干键，并在EXE命令执行之时，检查监视的键是否至少有一个已经被修改过了。如果是，那么服务器拒绝执行事务，并返回失败。

每个redisDb都保存着一个watched_keys字典。这个字典的键是某个被WATCH命令监视的数据库键，而字典的值是一个链表，链表中记录了所有监视相应数据库键的客户端。

​     
![img](12.png)<center>*redisDb代码示意*</center>


 

所有对数据库进行修改的命令，执行之后都会调用touchWatchKey函数对watched_key字典进行检查，查看是否有客户端正在监视刚刚被修改的键。如果有的话，那么会将监视的客户端的flags中的REDIS_DIRTY_CAS标识打开，证明事务安全性已经被破坏。

当服务器收到客户端发来的EXEC命令时，会判断该客户端的REDIS_DIRTY_CAS标识否已经已经打开，如果打开，证明客户端所监视的键，至少一个有一个被修改过了，客户端提交的事务已经不再安全，所以服务器会拒绝执行客户端的事务。

 

### 3.  慢查询日志

Redis的慢查询日志可以记录执行时长超过设定时长的命令，用户可以通过这个功能来监视和优化查询速度。

Redis执行一条命令分为4步：

**1)**    发送命令

**2)**    命令排队

**3)**    命令执行

**4)**    返回结果

慢查询只统计**命令执行**这一步的时间。

可配置项包括：超时时间与日志数量。

使用SLOWLOG GET命令可以查看服务器所保存的慢查询日志，如下图：

​     
![img](13.png)<center>*慢查询命令显示结果*</center>



使用SLOWLOG LEN可以获取慢查询日志当前列表长度，使用SLOWLOG RESET可以清空慢查询日志列表。

 
服务器状态redisServer中保存了几个和慢查询功能有关的属性：

​     
![img](14.png)<center>*redisServer代码示意*</center>



 

其中slowlog链表，保存了所有慢查询日志，以表头插入的方式进行新增，每个节点都是一个slowlogEntry结构，如下图：

​     
![img](15.png)<center>*slowlogEntry代码示意*</center>



 

每次执行命令的之前和之后，程序都会以微妙格式记录当前UNIX时间戳，这两个时间戳之差就是服务器执行命令所耗费的时长。

执行命令之后，程序检查执行时长是否超过slowlog-log-slower-than的值，如果是，就会创建一个新的日志，并添加到slowlog链表的表头。之后检查slowlog链表的长度是否超过slowlog-max-len选项设置的长度，如果是，那么多出来的日志会从链表中删除。

配置建议：

**1)**    slowlog-log-slower-than:未修改时，默认为10ms。对高流量场景，如果命令执行时间超过1ms的话，那么Redis最多可支持的QPS不超过1000，不符合Redis的期望。因此高QPS的场景应当将超时时间设置为1ms。

**2)**    slowlog-max-len：线上建议调大慢查询列表，记录慢查询时Redis会将长命令做截断，不会占用太多内存。线上建议设置为1000以上。

**3)**    慢查询只记录命令执行时间。因此客户端输入命令到得到回复的时间，大于命令执行时间。当客户端出现请求超时，慢查询可能只是其中一种原因。

 

### 4.  Pipeline

关于命令的4个步骤的执行时间：

**1)**    发送命令

**2)**    命令排队

**3)**    命令执行

**4)**    返回结果

其中第1)和第4)的时间之和为Round Trip Time（RTT，往返时间）。Redis提供了批量操作的命令(mset,mget等)，但并不是所有命令都支持批量操作。当服务端与客户端相距较远并且网络延迟较高时，会严重影响Redis服务器接收命令、返回结果的速度。而Redis服务器执行命令的时间通常都在微秒级别。所以有种说法：Redis性能瓶颈是网络。

Redis提供了一个Pipeline机制（流水线），能够将一组Redis命令进行组装，通过1次RTT（而不是N次）传输、处理并按顺序返回结果。下表显示了在不同网路下，10000条set语句使用非Pipeline和Pipeline执行时间对比：

| 网络       | 延迟   | 非pipeline | pipeline |
| ---------- | ------ | ---------- | -------- |
| 本机       | 0.17ms | 573ms      | 134ms    |
| 内网服务器 | 0.41ms | 1610ms     | 240ms    |
| 异地机房   | 7ms    | 78499ms    | 1104ms   |

 

Pipeline与原生的批量命令有不同的地方：

**1)**    原生的批量命令是原子性的，Pipeline不是。

**2)**    原生的批量命令是一个命令对应多个key，Pipeline是支持多个命令。

**3)**    原生的批量命令是服务器端实现，而Pipeline需要服务器端和客户端共同实现。

Pipeline在很多语言中都有实现。但Pipeline不能滥用，如果一次组装的数据量过大，一方面会增加客户端等待时间，另一方面会造成网络阻塞。所以建议将大量命令的Pipeline拆分成较小的Pipeline来完成。

 

### 5.  监视器

客户端通过执行Monitor命令，可以将自己变为一个监视器，实时接收并打印出服务器当前处理的命令：

​     
![img](16.png)<center>*Monitor命令示意*</center>



每当一个客户端向服务器发送一条命令请求，服务器除了处理这条请求之外，还会将关于这条命令请求的信息发给所有监视器。

客户端发送MONITOR命令后，自身flags标识REDIS_MONITOR会被打开。redisServer维护一个monitors链表，每个节点就是客户端本身。收到命令后，链表中会增加这个客户端。之后服务器收到命令都会对链表进行遍历，然后发送给链表中的所有客户端。
