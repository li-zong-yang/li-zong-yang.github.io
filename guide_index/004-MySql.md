# 1 写在前面的话

想要更好地阅读本文，您可能需要自行安装 MySQL，并熟练掌握 MySQL 的基本语法和使用。

阅读全文大概需要 40 分钟。

# 2 MySQL 架构设计

## 2.1 程序是如何跟 MySQL 打交道的

MySQL 作为标准的 C/S 架构，分为客户端和服务端。我们写代码的时候，通常会在代码中引入驱动和 client，然后在配置文件中填写 server 地址和账号密码，用于连接到 MySQL 服务器。大概的流程看起来就向下面一样。

## 2.2 程序是如何跟 MySQL 打交道的图解

![01_程序是如何跟mysql打交道的.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92aab59bd62b46218e3fa8959c8d0ced~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1738&h=850&s=108143&e=png&a=1&b=ffe6d3)

## 2.3 服务端流程分析

客户端向服务端发送请求并得到回复的过程本质上是一个进程间通信的过程，这个**处理连接**的过程，MySQL 支持以下 3 种方式：

1.  TCP/IP（端口：3306）；
2.  命名管道和共享内存（这个针对于 windows 系统）；
3.  Unix 域套接字文件（这个针对于 linux 系统）。

处理连接后，MySQL 会对我们发送的请求进行**解析与优化**，这个过程大概分为 3 步：

1.  查询缓存；
2.  语法解析；
3.  查询优化。
    然后再经过**存储引擎**，最后持久化。

## 2.4 服务端流程图解

为了方便大家记忆这个过程，小七画了下面一张图


![02_mysql架构（2）.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/177b45a340114237b4a6229733175f34~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2030&h=1672&s=257870&e=png&a=1&b=fffaf9)

以上内容，作为一个 CRUD 工程师，不需要掌握得那么深，了解即可。下文将会讲述重点知识**InnoDB 存储引擎** 。


# 3 InnoDB 架构设计

## 3.1 设计思路

如果让你来设计一个数据库，你会怎么做？结合我们平常的对 MySQL 的使用，小七觉得我们至少需要实现以下几个功能：

1.  数据需要持久化；
2.  支持的并发不能太低，速度要可以；
3.  一旦宕机，需要尽可能的减少数据的丢失，能够快速恢复数据；
4.  如果某一操作有问题，应该可以快速回滚。

实现如下

1.  针对第一点，咱们可以将数据写入磁盘中（MySQL 的**磁盘文件**）；
2.  针对第二点，咱们可以考虑先基于内存处理，然后再写入磁盘（MySQL 是通过 **Buffer Pool** 缓冲池实现的）；
3.  针对第三点，咱们可以记录一下当前的操作记录，类比与 redis 的 AOF 文件（MySQL 中叫 **redo log**）；
4.  针对第四点，咱们可以设计一个文件，专门存放每条操作记录更新前的值（MySQL 中叫 **undo log**）。
    接下来，我们借助一个更新语句，来看看 InnoDB 存储引擎的架构设计。

首先我们从磁盘文件中读取数据，在更新内存数据之前，将旧数据写入 undo log，同时写入 redo log，整个流程如下（其中的 OS cache 和 Redo Log Buffer，读者可以将其看做缓存，后面有涉及，将会详细讲解）：

## 3.2 图解


![03_InnoDB存储引擎的架构设计.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/263b5732c30b42f79a6189997667141b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1719&h=1006&s=114018&e=png&a=1&b=ffcecc)

# 4 MySQL 物理数据模型

## 4.1 数据在磁盘上的存储格式

我们每一行数据在磁盘上到底是怎么存储的呢？我们以常见的varchar为例，他的存储格式大概如下图所示：

![12_VARCHAR这种变长字段，在磁盘上到底是如何存储的.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bae05bd4b94c4a0ea739ebc84549e6a6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1683&h=558&s=83609&e=png&a=1&b=fedacb)

**注意：变长字段长度实际上是倒序存储的。**

## 4.2 null列表与数据头

下图展示了，null列表与数据头的详细信息，了解即可。


![13_null值列表与数据头.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77ddf821a23740659f659d8eb2d45de4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3071&h=1232&s=258590&e=png&a=1&b=feefcb)

## 4.3 行溢出

什么叫行溢出？就是说一行数据太多了，多的一个数据页都放不下了，需要放在其他数据页里面（这些数据页是由链表串联起来的），这个就叫行溢出。（数据页的详细介绍，请参考11.1）


![14_行溢出.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93d145bec0f04f1599bc5f31b4352a55~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1316&h=975&s=76896&e=png&a=1&b=fff5f3)

# 5 BufferPool

首先我们通过下图，简单地了解一下 BufferPool 的内存数据结构


![04_Buffer Pool内存数据结构.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4b7ec7705d84c119f3c8bb6f820e569~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1210&h=1035&s=98491&e=png&a=1&b=fed7d5)

我们知道 MySQL 的数据最后都是存放在磁盘文件中的，MySQL 将这一行行数据，放入到了一个一个的叫**数据页**的数据结构中，然后**数据页**会被 MySQL 加载到 **BufferPool** 中。

**BufferPoll** 主要由**描述数据**和**缓存页**构成，默认情况下每一个**数据页**对应一个**缓存页**，每一个**缓存页**都有一个对应的**描述数据**，描述数据你可以将它看做是缓存页的概览，我们可以通过描述数据找到与之对应的缓存页。

## 5.1 free 链表

### 5.1.1 概念

在 MySQL 服务端启动的时候，MySQL 会在内存中开辟一块 BufferPool，并初始化好对应的描述数据以及缓存页。这个时候缓存页都是空的。

然后当我们进行增删改操作的时候，BufferPool 才会将数据对应的数据页读出来，放在缓存页中。这个时候，就出现一个问题了，我们怎么知道哪些缓存页是空的呢？MySQL 为我们引入了另外一个概念，**free 链表**。他是一个双向链表数据结构，每一个节点都存放了空置的描述数据的地址，并且他还有一个基础节点，存放的是控制缓存页的个数。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2f29b83f9e84fdea844c92d571197e4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=370&h=245&s=7860&e=png&b=fff8f6)
###

### 5.1.2 缓存页 hash 表

现在我们通过 free 链表，知道了哪些缓存页是空的，但是我们并不知道哪些数据页是被缓存了的呢？其实 MySQL 中有一个**缓存页 hash 表**，如果在此表中，则表明数据已经被缓存了，他的 key=表空间号+数据页号，他的 value=缓存页地址。

> \**(表空间号+数据页号,缓存页地址)**

### 5.1.3 图解

![05_free链表和hash表.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c348c1817ac04010b9c9e2d7c4a32ac4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2134&h=1093&s=176278&e=png&a=1&b=fff0ef)

## 5.2 flush 链表

### 5.2.1 概念

如果你在执行增删改的时候，发现数据页没有被缓存，那么 MySQL 就会通过 free 链表找到对应的描述数据，最后缓存到缓存页中，并且断开 free 链表中对应的描述数据节点。但是这又会引出另外一个问题，只要你改变了缓存页的数据，那么缓存页肯定就和磁盘上的数据页不一致了，这个时候需要将缓存页的数据刷到磁盘上去，那么刷哪些数据呢，总不能全量刷盘吧？于是 MySQL 引入了另外一个链表 flush 链表。他的数据结构和 **free 链表**一致，只不过，他的节点放置的是需要被刷回磁盘的描述数据地址。

### 5.2.2 图解


![06_flush链表和脏数据.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/082c0e0976154e7b9f05afae10ae2426~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2890&h=1173&s=233073&e=png&a=1&b=fef0ee)

## 5.3 LRU 链表

### 5.3.1 概念

从前面的文章中我们知道了 BufferPool 中存在缓存页，但是我们思考一下，缓存页是启动的时候就分配好了的，如果满了怎么办？要么扩容，要么淘汰。MySQL 使用的是 LRU 算法淘汰部分缓存。而这个 LRU 算法，是基于 **LRU 链表**的。最近被访问过的缓存页，会被挪到链表最前面，因此最少访问的缓存页就会在链表的最尾部，淘汰缓存时，我们只需要淘汰最后的数据页即可。

### 5.3.2 图解


![07_LRU算法淘汰部分缓存.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d5147b52df5468286ab0c86cc6359bd~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600&h=704&s=95956&e=png&a=1&b=feccd1)

### 5.3.3 LRU 链表存在的问题

#### 5.3.3.1 预读机制对 LRU 链表的影响

为了提高效率，MySQL 从磁盘上加载数据到缓存的时候，他可能会把数据页相邻的其他数据页也加载到缓存中去，这个就是 MySQL 的**预读机制**。

我们思考一下这样的预读机制会对 LRU 链表造成什么影响呢？

这些被查出来的预读数据，可能根本不常用，但是他还是被放在了 LRU 链表的前面，从而导致他们不能被及时淘汰。

#### 5.3.3.2 触发预读机制的常见情况

*   **innodb\_read\_ahead\_threshold**
    默认 56，如果顺序访问一个区里的多个数据页的数量超过了这个阀值，那么就会把相邻区中所有的数据页都加载到缓存里去。

*   **innodb\_random\_read\_ahead**
    默认 off，如果 Buffer Pool 里缓存了一个区 13 个连续的数据页，且这些数据页会被频繁访问，那么就会把这个区的其他数据页加载到缓存里去。

#### 5.3.3.3 全表扫描对 LRU 链表的影响

*   **select \* from table**
    全表扫描，会将表里所有的数据页都从磁盘加载到 Buffer Pool 里面去，导致 LRU 尾部的链表反而是频繁被访问的数据。

####

#### 5.3.3.4 图解


![08_LRU链表在Buffer Pool的问题（预读机制）.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bebbbbfd4182439497889b2cb5d10ed3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2704&h=1003&s=266404&e=png&a=1&b=fee3da)

### 5.3.4 MySQL 对 LRU 算法的优化

通过上面的分析，我们知道了 LRU 算法可能会存在的一些问题，写 MySQL 的大佬们当然也想到了这些问题，下文列举了 MySQL 对 LRU 算法的两种优化。

#### 5.3.4.1 通过冷热数据分离，优化 LRU 算法

前面的问题为什么会出现呢？很大原因是因为所有数据都放在 LRU 链表中，如果我们把他分成冷热数据两部分，预读数据、全表扫描和其他不常用的数据放在冷数据区，其他常用的放在热数据区，缓存淘汰的时候，只淘汰冷数据区的数据，是不是就解决这个问题了？这个思想跟秒杀系统中热数据放 redis，冷数据放数据库，小七感觉也是异曲同工。

以下是相关的两个参数，了解即可，一般不会去修改他。

*   **innodb\_old\_blocks\_time**
    设置冷数据豁免时间，默认 1000ms。（1 秒内，被访问，则不会转移到热数据区域）

*   **innodb\_old\_blocks\_pct**
    设置冷数据区域所占大小，默认 37%


![09_通过热冷数据分离，优化LRU算法.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97525fbb32404bf5a1adac9269ff144c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1778&h=1344&s=166034&e=png&a=1&b=fffefe)

####

#### 5.3.4.1 通过定时任务，优化 LRU 算法

为了提升效率，MySQL 开启一个后台线程，定时把冷数据尾部的一些数据输入磁盘。

![10_定时任务，优化LRU算法.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5df4415feb54fe2b31a555e1b53d090~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1778&h=1456&s=151339&e=png&a=1&b=fffdfd)

## 5.4 free 链表、flush 链表、LRU 链表，修改数据的动态联系


![11_free链表、flush链表、lru链表，修改数据的动态联系.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68eda599052e4984b8cfcd1cc68e44e0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2192&h=1471&s=155972&e=png&a=1&b=ffffff)

# 6 redo log

## 6.1 概念

第三章我们从 InnoDB 架构设计提到了 redo log，这一章我们具体来聊一聊 redo log。

redo log 是 InnoDB 独有的，本质上只是记录了一下事务对数据库做了哪些修改。与在事务提交时将所有修改过的内存中的页面刷新到磁盘中相比，只将该事务执行过程中产生的 redo 日志刷新到磁盘的好处如下 ：

*   redo 日志占用的空间非常小，内存利用率高
*   redo 日志是顺序写入磁盘的，性能较高

## 6.2 图解

redo log 里本质上记录的就是在对某个表空间的某个数据页的某个偏移量的地方修改了几个字节的值，具体修改的值是什么，他里面需要记录的就是**表空间号+数据页号+偏移量+具体的值**。redo 日志有很多种，以下是常见的一种。


![25_redo log通用结构.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3f1145e627a4037be6eff09977ff008~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1100&h=240&s=26260&e=png&a=1&b=feccd1)

## 6.3 redo log block

为了更好的进行系统崩溃恢复，MySQL 把 redo log 都放在了大小为 512 字节的 redo log block 中 。

redo log block 分为以下 3 个部分：

1.  **header**
    存放了一些管理信息。

2.  **body**
    redo log 真正存放的地方。

3.  **traller**
    存放了一些管理信息。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/571cedc6d3e242ab9d164b703de7e7a3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=267&h=300&s=6229&e=png&b=fdfdfd)

其中 header 存放的内容如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7eff1c9cc864de983a8c674d59f8026~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=308&h=380&s=11231&e=png&b=fefefe)

整个 redo log 写入的流程，总结如下：

![24_redo log block.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45648cadbe01426baa21f870b26d9e40~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2033&h=1493&s=180026&e=png&a=1&b=f6995c)


## 6.4 redo log buffer

首先让我们回顾一下下面这张图

![03_InnoDB存储引擎的架构设计 (1).png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d83543782b44bd6997a15dc9f1f0364~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1719&h=1006&s=114018&e=png&a=1&b=ffcecc)


为了增加数据更新的效率，MySQL 引入了 BufferPool 的概念；同理，为了增加 redo log 的效率，MySQL 同样引入了 redo log buffer 的概念，它其实就是 redo log 的缓冲区，它包含了若干个连续的 redo log block。最后，我们要知道 redo log 都是先进入 redo log buffer 中的一个 block，然后事务提交的时候才会刷入磁盘文件里去。那么这里会有两种情况

1、事务没提交，MySQL 挂了

这种情况，丢了就丢了，没有影响，不需要重做。

2、事务提交了，MySQL 挂了，但是已经被修改的缓存页还没有被刷入磁盘

这种情况因为有 redo log 存在，你重启 MySQL 之后，可以把没来得及刷入磁盘的事务，他们所对应的 redo log 都加载出来，再在 BufferPool 的缓存页里重做一遍，就可以保证事务提交之后，修改的数据绝对不会丢。

# 7 undo log

## 7.1 概念

通过上文我们知道了 redo log 保证了事务提交后的数据，不会丢。但是如果事务执行到一半就 GG 了怎么办？为了保证事务的原子性，我们需要把东西改回原先的样子，这个过程就称之为回滚。MySQL 的回滚主要依赖于 undo log。

undo log 记录的东西也很简单，比如插入一条记录时，至少要把这条记录的主键值记下来，之后回滚的时候只需要把这个主键值对应的记录删掉就好了。

## 7.2 图解


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ad06e92b8714eceb20008565506b0c5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=685&h=134&s=17159&e=png&b=fffcfc)

# 8 bin log

## 8.1 概念

前面我们对 redo log 做了介绍，它是一种重做日志，它主要关注“哪个数据页的哪个数据做了什么修改。

bin log 叫做归档日志，它主要关注“对哪个表的哪个数据做了什么操作，操作之后是什么”。

我们可以这样理解 bin log 是偏向于逻辑性的日志，而 redo log 更偏向于物理性。

**注意：bin log 不是 InnoDB 存储引擎特有的日志文件，是属于 MySQL server 自己的日志文件。**

## 8.2 bin log 和 redo log的区别

1.  bin log 是MySQL本身就拥有的，不管哪种存储引擎；redo log 是InnoDB独有的。
2.  bin log是一种逻辑日志，redo log 是一种物理日志。
3.  bin log没有幂等性，redo log具有幂等性，多次操作前后的状态是一致的。
4.  bin log开启事务的时候，会将每一次提交的事务一次性写入内存缓冲区，如果未开启事务，则每次进行增删改时，就会将对应事务信息写入内存缓冲区；而redo log是在数据准备修改之前，将数据写入缓冲区redo log中的，然后在缓冲区中修改数据，而且在提交事务的时候，现将redo log 写入缓冲区，写入完成后，再提交事务。
5.  bin log只会在事务提交时，一次性写入bin log；redo log最后一个提交的事务记录会覆盖之前所有未提交的事务记录，并且一个事务的redo log中间会插入其他事务的redo log。
6.  bin log是追加写入，不会覆盖；redo log是循环写入，会覆盖。
7.  bin log一般用于主从复制和数据恢复；redo log 一般用于MySQL，重启后恢复事务已提交但未写入数据表的数据。

# 9 事务

在开始新篇章之前，让我们回顾一下，下面的流程

1、MySQL 事务执行流程


![28_MySQL事务执行流程.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5135e7ae21ca45bf858524d2e447cfc6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1636&h=1862&s=186154&e=png&a=1&b=dfdfdf)

2、MySQL 事务恢复流程

![30_MySQL事务恢复流程.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/529821b0310e48a5baaa74ba622b6986~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1142&h=1418&s=108520&e=png&a=1&b=e0e0e0)

## 9.1 脏写与脏读

### 9.1.1 概念

如果一个事务修改了另一个未提交事务修改过的数据，那就意味着发生了脏写 。

如果一个事务读到了另一个未提交事务修改过的数据，那就意味着发生了脏读 。

### 9.1.2 分析

脏读

1.  原始数据为 null
2.  事务 A 更新数据为 A
3.  事务 B 查询数据为 A
4.  事务 A 这个时候回滚了，那么它用它的 undo log 去回滚，现在数据为 null
5.  事务 B 再次查询数据为 null

脏写

1.  原始数据为 null
2.  事务 A 更新数据为 A
3.  事务 B 更新数据为 B
4.  事务 A 这个时候回滚了，那么它用它的 undo log 去回滚，现在数据为 null
5.  事务 B 再次查询数据为 null，事务B修改的数据丢失了

## 9.2 不可重复读

### 9.2.1 概念

如果一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值，那就意味着发生了不可重复读。

### 9.2.2 分析

1.  原始数据为 A
2.  事务 A 查询数据为 A
3.  事务 B 更新数据为 B，并提交
4.  事务 A 查询数据为 B
5.  事务 C 更新事务为 C，并提交
6.  事务 A 查询数据为 C

## 9.3 幻读

### 9.3.1 概念

如果一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来，那就意味着发生了幻读。

### 9.3.2 分析

1.  事务 B 插入数据 1 条，总数 11 条
2.  事务 A 查询数据为 11 条
3.  事务 B 这个时候回滚了，总数变为 10 条
4.  事务 A 查询数据为 10 条

## 9.4 隔离级别

### 9 .4.1 SQL 标准中的四种隔离级别

| 隔离级别         | 隔离级别（中文） | 脏读 | 不可重复读 | 幻读 |
| :--------------- | :--------------- | :--- | :--------- | :--- |
| READ UNCOMMITTED | 读未提交         | √    | √          | √    |
| READ COMMITTED   | 读已提交         | ×    | √          | √    |
| REPEATABLE READ  | 可重复读         | ×    | ×          | √    |
| SERIALIZABLE     | 串行化           | ×    | ×          | ×    |

### 9.4.2 MySQL 标准中的四种隔离级别

| 隔离级别         | 隔离级别（中文） | 脏读 | 不可重复读 | 幻读 |
| :--------------- | :--------------- | :--- | :--------- | :--- |
| READ UNCOMMITTED | 读未提交         | √    | √          | √    |
| READ COMMITTED   | 读已提交         | ×    | √          | √    |
| REPEATABLE READ  | 可重复读         | ×    | ×          | x    |
| SERIALIZABLE     | 串行化           | ×    | ×          | ×    |

READ UNCOMMITTED 我们上文提到的几种问题，他都没有解决，所以正常人都不会使用它；

SERIALIZABLE 效率太低，也没人会用他；

READ COMMITTED 在某些需要不可重复读的情况下，会用到，但是这种情况，如果你是用的 Spring 框架，那么可以在代码里单独指定，生产中的 MySQL 数据库级别一般也不是它；

REPEATABLE READ 这个是 MySQL 默认的隔离级别，这里我们需要注意的是，在 RR 隔离级别下，MySQL 解决了幻读问题，具体是怎么解决的呢？下文将会从 undo log 版本链讲起。

## 9.5 undo log 版本链

### 9.5.1 概念

简单来说呢，我们每条数据其实都有两个隐藏字段，一个是 trx\_id，一个是 roll\_pointer，这个 trx\_id 就是最近一次更新这条数据的事务 id，roll\_pointer 就是指向你了你更新这个事务之前生成的 undo log，接着假设有一个事务 B 跑来修改了一下这条数据，把值改成了值 B，事务 B 的 id 是 15，那么此时更新之前会生成一个 undo log 记录之前的值，然后会让 roll\_pointer 指向这个实际的 undo log 回滚日志。

### 9.5.2 图解

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e98c578b38d3459fa10f2d39b437d6bd~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=612\&h=191\&s=18085\&e=png\&b=fcfcfc)

 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c79cc87652204e20a3dab3c55f376078~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=673&h=316&s=8369&e=png&b=fdfdfd)

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3cdc14b1a7084df086cfce5aed375d23~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=634&h=447&s=12736&e=png&b=fdfdfd)
## 9.6 ReadView 机制

### 9.6.1 概念

对于使用 READ COMMITTED 和 REPEATABLE READ 隔离级别的事务来说，都必须保证读到已经提交了的事务修改过的记录，也就是说假如另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录的，这里就引出另一个问题了，我们怎么知道 undo log 版本链中哪些链条是可读的，哪些链条又是不可读的呢？针对这个问题，MySQL 为我们引入了 ReadView。

### 9.6.2 组成部分

ReadView 中重要的有 4 个东西：

*   **m\_ids**
    这个就是说此时有哪些事务在 MySQL 里执行还没提交的，表示所有的活跃的读写事务 id 的集合；

*   **min\_trx\_id**
    表示在生成 ReadView 时当前系统中活跃的读写事务中最小的事务 id，也就是 m\_ids 中的最小值；

*   **max\_trx\_id**
    表示 MySQL 要生成的下一个事务 id，也就是事务最大 id；

*   **creator\_trx\_id**
    表示当前事务的 id

**注意：只有在对表中的记录做改动时（执行 INSERT、DELETE、UPDATE 这些语句时）才会为事务分配事务 id，否则在一个只读事务中的事务 id 值都默认为 0。**

### 9.6.3 判断规则

我们可以根据版本链中的 trx\_id 和 ReadView 中这几个值来判断事务是否已经被提交了。

*   如果被访问版本的 **trx\_id** 属性值与 ReadView 中的 **creator\_trx\_id** 值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
*   如果被访问版本的 **trx\_id** 属性值小于 ReadView 中的 **min\_trx\_id** 值，表明生成该版本的事务在当前事务生成 ReadView 前已经提交，所以该版本可以被当前事务访问。
*   如果被访问版本的 **trx\_id** 属性值大于或等于 ReadView 中的 **max\_trx\_id** 值，表明生成该版本的事务在当前事务生成 ReadView 后才开启，所以该版本不可以被当前事务访问。
*   如果被访问版本的 **trx\_id** 属性值在 ReadView 的 **min\_trx\_id** 和 **max\_trx\_id** 之间，那就需要判断一下 **trx\_id** 属性值是不是在 **m\_ids** 列表中，如果在，说明创建 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建 ReadView 时生成该版本的事务已经被提交，该版本可以被访问。

## 9.7 MVCC 机制

MVCC 机制，翻译成中文是多版本并发控制机制的意思。其实我们 9.5 和 9.6 两章已经将 MVCC 核心实现讲了，也就是 **undo log 版本链 + ReadView 机制**。下面我们分析一下 RC 以及 RR 分别是怎么通过 MVCC 机制实现的。

### 9.7.1 READ COMMITTED

#### 9.7.1.1 关键点

实现RC的关键点在于，每次读取数据前都生成一个 ReadView。

#### 9.7.1.2 步骤分析

1.  我们假设已经存在有一行数据txr\_id=8；
2.  然后现在有2个活跃事务，事务A（id=10），事务B（id=15）；
3.  事务B将数据更新为了值B，未提交；
4.  事务A发起一次查询，生成一个ReadView；
5.  根据ReadView判断规则，当前数据的**txr\_id=15**，在 ReadView的**min\_trx\_id** 和 **max\_trx\_id** 之间，且 **trx\_id** 属性值在 **m\_ids** 列表中，说明创建 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；
6.  undo log 版本链继续向下寻找，**txr\_id=8**小于 ReadView 中的 **min\_trx\_id** 值，表明生成该版本的事务在当前事务生成 ReadView 前已经提交，所以该版本可以被当前事务访问；
7.  所以事务A查询的时候访问到的就是已经提交过的值A了。

#### 9.7.1.3 图解


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e3703c2ad8744549bf8b84bc0641533~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1076&h=635&s=68375&e=png&b=fdfafa)

### 9.7.2 REPEATABLE READ

#### 9.7.2.1 关键点

在第一次读取数据时生成一个 ReadView

#### 9.7.2.2 步骤分析

1.  我们假设已经存在有一行数据txr\_id=8；
2.  然后现在有2个活跃事务，事务A（id=10），事务B（id=15）；
3.  事务A发起一次查询，生成一个ReadView；
4.  根据ReadView判断规则，当前**txr\_id=8**小于 ReadView 中的 **min\_trx\_id** 值，表明生成该版本的事务在当前事务生成 ReadView 前已经提交，所以该版本可以被当前事务访问；
5.  事务B将数据更新为了值B，并提交；
6.  事务A发起一次查询，还是使用第一次生成的ReadView；
7.  根据ReadView判断规则，当前数据的**txr\_id=15**，在 ReadView的**min\_trx\_id** 和 **max\_trx\_id** 之间，且 **trx\_id** 属性值在 **m\_ids** 列表中，说明创建 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；
8.  undo log 版本链继续向下寻找，**txr\_id=8**小于 ReadView 中的 **min\_trx\_id** 值，表明生成该版本的事务在当前事务生成 ReadView 前已经提交，所以该版本可以被当前事务访问；
9.  所以事务A查询的时候访问到的就是已经提交过的值A了（也就是实现了可重复读）。

#### 9.7.2.3 图解

步骤1-4

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69dedf07d2074340a4f26a02ffe1772c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=975&h=473&s=16271&e=png&b=fdfdfd)
步骤5-9

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e3703c2ad8744549bf8b84bc0641533~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1076&h=635&s=68375&e=png&b=fdfafa)

#### 9.7.2.4 幻读

解决幻读的推导和和上面解决不可重复度大同小异，小七这里留给读者，自己去推导。

# 10 MySQL 锁机制

MySQL 锁机制作是MySQL中重要的一环，但是针对小七这种CRUD开发工程师来说，并不需要了解那么深入，我们只需要知道一些常见知识即可。

1、多个事务同时更新一行数据，此时都会加锁，然后都会排队等待，必须一个事务执行完毕了，提交了，释放了锁，才能唤醒别的事务继续执行，这个时候加的锁叫独占锁；

2、当有人在更新数据时，其他事务读取这行数据的时候，默认是走MVCC机制的，也就是不加锁的；

3、当我们非要在执行查询的时候加锁呢？这个时候可以使用lock in share mode，手动加上共享锁；

4、共享锁和共享锁是不互斥的，共享锁和独占锁是互斥的，独占锁和独占锁也是互斥的。

# 11 索引

## 11.1 数据页存储结构

### 11.1.1 数据页的各个部分

在讲索引之前，让我们看看一个单独的数据页是什么样子的


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b94b8a12b7d49a9bad9227bbb4949bb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=595&h=626&s=34090&e=png&b=fff6f5)

去除掉一些我们不太需要那么关注的部分后，简化如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eab7a6c631ab4b77a289443890d4e971~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=199&h=360&s=6186&e=png&b=fdfdfd)

也就是说平时我们在一个表里插入的一行一行的数据会存储在数据页里，然后数据页里的每一行数据都会按照主键大小进行排序存储，同时每一行数据都有指针指向下一行数据的位置，组成单向链表。

### 11.1.1 页分裂

随着业务的发生，我们的数据页一般会越来越大，当大到一定程度的时候，就需要再搞一个数据页了，如下图所示


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27339b88c39c4ed6812f600b2383e30e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=690&h=665&s=38509&e=png&b=fefefe)

但是这一步骤并不是说简简单单多加一个数据页就 OK，还需要保证新加的数据页中的每一行数据的主键值都要比前面的大才行，所以数据行有可能会在数据页中挪动。具体如下图所示：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33cb0a4dcc884ba193e6c6b9bd43b138~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=975&h=640&s=25775&e=png&b=fdfdfd)

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7caec90753a64dd3958bd6fe686fb23b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=986&h=652&s=29222&e=png&b=fdfdfd)

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f70bb77323e74f809d4649eeed22ca65~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=973&h=621&s=23324&e=png&b=fcfcfc)

## 11.2 索引页存储结构

### 11.2.1 概念

上一节我们讲了数据页的存储结构，这一节我们继续学习索引页的存储结构。

我们先思考一个问题，如果我们只有一般的数据页，咱们怎么找到自己想要的数据呢？是不是要将数据页全部遍历，再在每一个页中，通过二分查找查询数据。这么做，实在是太慢了！所以 MySQL 抽象出了一个索引页的概念，它和一般的数据页差不多，只不过存放的是最小主键值和页号。然后后续你查询主键值，就可以在目录里二分查找直接定位到那条数据所属的数据页，接着到数据页里二分查找定位那条数据就可以了，如下图所示。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/298281bd9a4b4ece945da3b76eccfc92~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1371&h=743&s=43078&e=png&b=f8f8f8)

但是随着数据页越来越多，索引页也变得越来越多，这个时候怎么办呢？这个时候 MySQL 会抽象出一个更高层级的索引页，它里面记录的是最小主键值和索引页号。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7236dfd5e6a6470bae319c0f6c3db7f3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=523&h=527&s=10503&e=png&b=eeeeee)

那么现在问题再次来了，假如你最顶层的那个索引页里存放的下层索引页的页号也太多了，怎么办呢？此时可以再次分裂，再加一层索引，最后不断的向上加，索引页看起来就像下面这个样子了，也就是一颗 B+树。


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4dbf2301c9874d84af62b4b10156680f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=769&h=353&s=10328&e=png&b=fdfdfd)

### 11.2.1 例子

最后我们以最简单最基础的主键索引来举例，当你为一个表的主键建立起来索引之后，其实这个主键的索引就是一颗 B+树，然后当你要根据主键来查数据的时候，直接就是从 B+树的顶层开始二分查找，一层一层往下定位，最终一直定位到一个数据页里，在数据页内部的目录里二分查找，找到那条数据。

## 11.3 聚簇索引

### 11.3.1 特点

我们上面介绍的B+树索引，它有两个特点：

*   **使用记录主键值的大小进行记录和页的排序**
*   **B+树的叶子节点存储的是完整的用户记录**
    符合这两个特点的索引，就是聚簇索引。在 InnoDB 存储引擎中，聚簇索引就是数据的存储方式，InnoDB 存储引擎会自动的为我们创建聚簇索引。

## 11.4 二级索引

### 11.4.1 概念

聚簇索引，使用记录主键值的大小进行记录和页的排序，他是和主键强关联的。但是如果查询的条件不是主键，而是其他列呢？这个时候，就要请出咱们的二级索引了。

二级索引也是一颗B+树，但是它的数据页里存放的是**主键+目标字段值**。换句话说，将聚簇索引中的主键值替换成目标值段，且叶子节点仅存储**主键+目标字段值**这两个列的值，那么他就是二级索引了。

当你要根据目标字段来查数据的时候，直接就是从 B+树的顶层开始二分查找，一层一层往下定位，最终一直定位到一个数据页里，在数据页内部的目录里二分查找，找到那条数据。但是这条数据只有**主键+目标字段值**。

### 11.4.2 图解


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37710d972b25488b9de3b6bd19e38295~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=891&h=556&s=21084&e=png&b=fdfdfd)

### 11.4.3 回表

如果你的查询结果中还需要有其他值，那么你得再根据主键在**聚簇索引**这个B+树中，再查找一次，得到最终的结果，这个过程叫做**回表**。

## 11.5 联合索引

联合索引也是一颗B+树，但是它的数据页里存放的是**主键+多个目标字段值**。其他和二级索引类似。

## 11.6 覆盖索引与回表查询

### 11.6.1 概念

首先我们要明确一点，覆盖索引，并不是真正的索引，他其实是一种基于索引的查询方式。

不管是二级索引，还是联合索引，如果你的查询结果中没有其他值，只有索引值，那么你得不需要再在**聚簇索引**这个B+树中，再查找一次，只需要扫描当前索引的叶子节点，就能得到结果，这种就叫做覆盖索引。

### 11.6.2 例子

```sql
select xx1,xx2,xx3 from table order by xx1,xx2,xx3
```

基于xx1,xx2,xx3建立联合索引

这种情况下，你仅仅需要联合索引里的几个字段的值，那么其实就只要扫描联合索引的索引树就可以了，不需要回表去聚簇索引里找其他字段了。

## 11.7 如何更好的建立索引

通过前文我们知道了索引其实是一颗一颗的B+树，那么接下来我们介绍一下B+树索引适用的条件。

### 11.7.1 索引适用的条件

首先我们给出一张示例表，如下：

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(32) NOT NULL COMMENT '用户名',
  `age` int(3) NOT NULL COMMENT '年龄',
  PRIMARY KEY (`id`),
  KEY `index_name_age` (`name`,`age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='user示例表';
```

这个例子中，有两个索引，一个是根据id排序的聚簇索引，另一个是跟据name和age排序的二级索引。

**注意：二级索引，是先跟据name排序，如果name相同，再根据age排序的。**

#### 11.7.1.1 全值匹配

例子：

```sql
SELECT * FROM user WHERE name = '第七人格' AND age = '29';
```

解析：

*   因为B+树的数据页和记录先是按照name列的值进行排序的，所以先可以很快定位name列的值是“第七人格”的记录位置。
*   在name列相同的记录里又是按照age列的值进行排序的，所以在name列的值是“第七人格”的记录里又可以快速定位age列的值是'29'的记录。

#### 11.7.1.2 匹配左边的列

【例子】：

```sql
SELECT * FROM user WHERE name = '第七人格';
```

解析：

*   因为B+树的数据页和记录先是按照name列的值进行排序的，所以先可以很快定位name列的值是“第七人格”的记录位置。

###

【反例】：

```sql
SELECT * FROM user WHERE age = '29';
```

解析：

*   因为B+树的数据页和记录先是按照name列的值进行排序的，所以先可以很快定位name列的值是“第七人格”的记录位置，但是现在用age去找，你想想能找到吗？当然找不到了，你都没找到第一层排序的name值，怎么能找到下层的age呢？

#### 11.7.1.3 匹配列前缀

【例子】：

```sql
SELECT * FROM user WHERE name like '第七%';
```

解析：

*   因为B+树的数据页和记录先是按照name列的值进行排序的，那么的值是按照字符串排序的，字符串本质是按照字符排序的，这个例子中“第七”是被排好序了的，也可以很快定位name列的值是“第七....”的记录位置。
    【反例】：

```sql
SELECT * FROM user WHERE name like '%第七%';
```

或者

```sql
SELECT * FROM user WHERE name like '%第七';
```

解析：

*   “第七”并没有排好序，所以无法使用索引。

#### 11.7.1.4 匹配范围值

【例子】：

```sql
SELECT * FROM user WHERE name > 'Anna' AND name < 'Ziad';
```

解析：

*   name能用到索引。
    【例子】：

```sql
SELECT * FROM user WHERE name > 'Anna' AND age < '35';
```

解析：

*   name能用到索引，age不能。
    【思考】：

```sql
SELECT * FROM user WHERE name = 'Anna' AND age < '35';
```

请读者思考上面可以使用到索引吗？为什么？

#### 11.7.1.5 排序

【例子】：

```sql
SELECT * FROM user ORDER BY name,age LIMIT 1;
```

解析：

*   这个是可以用到索引的，因为他是按照联合索引的字段顺序去进行order by排序的，这样就可以直接利用联合索引树里的数据有序性，到索引树里直接按照字段值的顺序去获取数据。
    【反例】：

```sql
SELECT * FROM user ORDER BY name ASC,age DESC;
```

解析：

*   既有升序又有降序，没办法使用索引。

#### 11.7.1.6 分组

【例子】：

```sql
SELECT name,age FROM user GROUP BY name ,age;
```

解析：

*   这个是可以用到索引的，原因可以类比排序。

### 11.7.2 阿里巴巴索引规约

*   【强制】业务上具有唯一特性的字段，即使是组合字段，也必须建成唯一索引。
*   【强制】超过三个表禁止 join。需要 join 的字段，数据类型保持绝对一致；多表关联查询时， 保证被关联的字段需要有索引。
*   【强制】在 varchar 字段上建立索引时，必须指定索引长度，没必要对全字段建立索引，根据 实际文本区分度决定索引长度。
*   【强制】页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决。
*   【推荐】如果有 order by 的场景，请注意利用索引的有序性。order by 最后的字段是组合索引的一部分，并且放在索引组合顺序的最后，避免出现 file\_sort 的情况，影响查询性能。
*   【推荐】利用覆盖索引来进行查询操作，避免回表。
*   【推荐】利用延迟关联或者子查询优化超多分页场景。
*   最好。
*   【推荐】建组合索引的时候，区分度最高的在最左边。
    以上规约摘自阿里巴巴开发手册，通过前面的学习，小七相信大家可以从更深层次理解这些规约了，而不是一味的死记硬背。

## 11.8 执行计划与性能优化浅谈

### 11.8.1 总览

在生产中我们判断一个SQL写的好不好，一般都是通过EXPLAIN关键字，看他的执行计划的。

首先我们看看执行计划长什么样子，我们执行以下SQL

```sql
EXPLAIN SELECT * FROM user WHERE name = '第七人格' AND age = '29';
```

得到的执行计划如下：

### 11.8.2 要素

#### 11.8.2.1 id

这个字段对性能优化来说，不太重要，我们只需要知道，在一个大的查询语句中每个SELECT关键字都对应一个唯一的id 就行了。

#### 11.8.2.2 select\_type

SELECT关键字对应的那个查询的类型，也不重要。

#### 11.8.2.3 table

表名，意思就是查的哪个表，也不是很重要。

#### 11.8.2.4 partitions

匹配的分区信息，我们接触不到这个知识点，99.9%都是null。

#### 11.8.2.5 type

针对单表的访问方法，这个就非常重要了。

完整的类型如下：system，const，eq\_ref，ref，fulltext，ref\_or\_null，index\_merge，unique\_subquery，index\_subquery，range，index，ALL 。

这里我们针对比较常见和重要的几种类型介绍一下。

const，常量级，一般出现这个，就表示，你写的SQL非常好，性能非常快。哪些属于const呢？比如select \* from user where id=x，或者select \* from user where name=x这样的的语句，直接就可以通过聚簇索引或者二级索引+聚簇索引回表，轻松查到你要的数据。

值的一提的是，这里的二级索引必须是唯一索引。如果是普通索引，那么就是ref了，就如我们总览中的执行计划一样。当然，ref也是一种非常快的查询方式。

range也是一种常见的查询方式，一般出现在范围查询中，比如我们前面举的一个例子select \* from user where name>=x and name <=x，假设name就是一个普通索引，此时就必然利用索引来进行范围筛选，一旦利用索引做了范围筛选，那么这种方式就是range。

我们再回忆一下这个sql，select name,age from user where age = '29'，因为age不是索引最左边的值，所以它是没法从联合索引的根节点二分查找快速跳转的，但是因为他的结果和条件都在索引里，所以MySQL的优化器，会直接扫描这个联合索引，一个个的遍历。也就是说，针对这种只要遍历二级索引就可以拿到你想要的数据，而不需要回源到聚簇索引的访问方式，就叫做index访问方式，这种访问方式相对于前面几种，就要慢的多。

最后再看看ALL，顾名思义，全表扫描，一般属于我们要杜绝的情况。

#### 11.8.2.6 possible\_keys

可能用到的索引。

#### 11.8.2.7 key

实际上使用的索引。

#### 11.8.2.8 key\_len

实际使用到的索引长度。

#### 11.8.2.9 ref

当使用索引列等值查询时，与索引列进行等值匹配的对象信息。

#### 11.8.2.10 rows

预估的需要读取的记录条数，这个值按道理来说，越小越好。

#### 11.8.2.11 filtered

某个表经过搜索条件过滤后剩余记录条数的百分比，对于单表查询来说，这个值没什么意义，都是100%，但是对于多表查询就有意义了，越小越好。

#### 11.8.2.12 Extra

一些额外的信息。比如Using where，Using index，Using filesort等等。我们用group by、union、distinct之类的语法的时候，要是没法直接利用索引来进行分组聚合，那么MySQL会直接基于临时表来完成，会有大量的磁盘操作，也就是会使用文件排序（Using filesort）。这种情况一般也是需要避免的。

# 12 参考资料

*   《从根上理解 MySQL》
*   《MySQL 技术内幕》
*   《深入浅出 MySQL》
*   《从零开始带你成为 MySQL 优化实战高手》
*   《深入理解分布式事务》
*   《阿里巴巴开发手册》
*   《高性能MySQL》

# 13 写在后面的话

如果你觉得小七文章给您带来了一些收获，可以帮忙点个赞，或者关注一下小七，小七会一如既往地更新有价值的博客。如果文章存在错误，也请联系小七，小七会在看到后，第一时间修改。最后感谢大家的支持，谢谢\~