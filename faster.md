# Faster: A Concurrent Key-Value Store with In-Place Updates
这篇论文发表于SIGMOD 2018, 论文介绍了一种支持高并发高性能的key-value存储系统的设计思路, 测试结果声称可以达到每秒1.6亿次操作的性能, 效果非常惊人. 自论文发表以来, 一直对这篇论文的原理比较好奇, 趁着春节前后相对的空挡期, 仔细读了两遍论文, 对核心设计原理进行了总结.

## 系统架构

![image](http://note.youdao.com/yws/res/14063/9B50326C0E194514AB3C1EA8C2CC1637)

先从系统架构上整体看一下Faster的设计逻辑. 首先, Faster有一个基于hash表的内存index结构, 然后index指向一条一条record, 每条record离存放的就是key-value数据. 这些record由3类allocator分配, 分别是in-memory allocator, append-log allocator和hybird-log allocator. 整个Faster尽可能采用无锁设计, 减少锁互斥带来的开销, 提供了Read, Upsert, Read-Modify-Write, Delete这4种编程接口. 从整体架构上看, Faster核心就是一个无锁hash index加上一组不同的allocator, 没什么秘密而言, 不过声称每秒1.6亿次op的hash表, 充分体现了faster的工程能力, 下面详细看看每个部分.

## Epoch Protection Framework
为了减少锁竞争, 提升多核环境下的可扩展性, Faster基于一种叫做Epoch的设计框架. Epoch的设计原理如下:
   - 维护一个全局的原子计数器E, 叫做current epoch, E可以被任何线程增加
   - 每个线程拥有一个thread local的E的副本, 用Et表示. 每个线程周期性的更新Et. 所有的Et保存在一个共享的table中, 每个线程一个cacheline大小
   - 对于epoch c来说, 如果每个线程的Et都大于c, 那么我们认为c是安全的. 注意如果epoch c是安全的, 那么所有小于c的epoch也一定是安全的.
   - 同时维护一个全局计数器Es, 记录当前最大的安全epoch. Es的初始值通过扫描epoch table计算出来, 并且任何线程更新自己的Et的时候, 也会更新Es.
   - 当线程增加E的时候(比如从c增加到c+1), 可以注册一个回调函数, 该回调函数在c是安全的时候被调用. 所有的<epoch, action>对保存在一个叫做drain-list的地方, drain-list基于数组实现, 当Es更新的时候, drain-list扫描出来可以触发的action来执行. 我们使用CAS机制来保证每个action只执行一次.

这些就是Epoch Protection Framework的核心设计思想, 当epoch变为安全的时候, 也就意味着没有任何thread在引用他了, 这时候对应的数据就可以安全释放了. 下面看看几个主要的编程接口:
   - Acquire. 在epoch table中, 为当前线程增加一个epoch entry, 并且设置Et=E.
   - Refresh. 根据E更新Et和Es, 并且触发在drain-list中的callback.
   - BumpEpoch(action). 给E加1, 从c到c+1, 并且将<c, action>加入到drain-list中.
   - Release. 在epoch中释放epoch entry

举个例子, 考虑shared_ptr的实现方式. 标准的shared_ptr的实现方式是引用计数, 当引用计数为0时释放内存. 如果换成Epoch方法, 则基本流程如下:
   - 当前线程调用acquire.
   - 当需要释放内存时, 调用BumpEpoch(action), action就是释放内存的操作.
   - 所有线程定期调用refresh. 当epoch安全时, 调用对应的action.

## The Faster Hash Index
hash index是faster的关键组件之一, faster里实现了一个支持并发, latch-free, 可扩展, 可以resize的hash index. hash index由一个cache-aligned数组组成, 每个bucket为64byte, 下图是hash bucket的结构. 每个hash bucket由7个8 byte的entry和一个8 byte的pointer entry组成, 每个overflow bucket仍然保持cache-align特性, 并且由内存alloctor提供内存空间. 每个entry由tag(15 bit)和address(48bit)组成, 最高位如果是0, 表示这是一个空的slot. tag用来降低hash碰撞, 很多64bit的机器不需要使用64bit的地址, 比如intel是48bit. 关于hash index的操作, 需要特别注意的是当entry不存在的时候, 不能直接使用CAS来插入entry, 论文中给出了一个例子, 解法就是利用最高位的tentative bit, 实现两阶段insert来解决问题, 详细算法可以参考论文. 同时hash index基于Epoch机制和多个状态实现了相对lower cost的resize操作, 详细的算法在论文附录B中描述了.

![image](http://note.youdao.com/yws/res/14361/CC36503376FF4B53BF85E18AF5FE6FE6)

虽然hash index在论文中描述篇幅不多, 不过实现的如此高效, 我觉得还是非常值得学习的, 大家可以对着源码一起阅读和理解.

## In-memory Allocator
有了hash index, 再结合一个简单的memory allocator, 比如jemalloc, 就可以形成简单的in-memory key value store了. 不过hash index只能保证自己是多线程安全的, in-memory allocator还需要保证自己也是多线程安全的, 下图就是一个多线程竞争产生的潜在问题, 论文中通过two-phase insert配合tentative bit解决了. 算法类似于乐观锁的机制, 论文里有详细描述, 这里就不仔细说了.

![image](http://note.youdao.com/yws/res/14482/A8E9503AB52B448AA60BBF8261913E0A)

## Log Allocator
只有in-memory allocator还是显得比较单薄, 为此论文中借鉴类似log-structured tree的思路, 设计了一个log allocator来实现数据持久化功能. 数据持久化之后, hash index记录的地址就不再是内存的物理地址而是磁盘的逻辑地址了. log allocator的核心设计思想和lsm-tree非常类似, 如下图所示, 内存中维护一个大的循环队列, 所有insert和update操作都会在循环队列中append, 循环队列划分成一个一个的page frame, 方便数据持久化. 随着队列的head offset和tail offset的变化, 队列中的数据被异步的刷新到磁盘中, 这里仍然采用了epoch framework来保证多线程安全. 同样和lsm-tree类似, 删除操作仅仅是插入了一个墓碑, 配合GC机制来实现空闲磁盘空间的回收利用.

![image](http://note.youdao.com/yws/res/14494/BD91790946374D55A8750CE3A7F1E0FA)

## HybridLog
log allocator的缺点就是lsm-tree的缺点, 存在写放大问题, 对于update密集型负载来说, 不是最优化的方案. 于是论文又提出了HybridLog的设计思路对纯log allocator进行优化. 优化思路也比较直观, 和lsm-tree的设计思路类似, 对内存中的循环队列, 划分成mutable和immutable两大部分, mutable部分的update操作不在写log了, 而是直接进行原地的update. immutable部分和原来的逻辑类似, 无需修改. 不过论文中有很多细节需要考虑, 包括如何保证单调性, 如何recovery以及多线程安全等.

![image](http://note.youdao.com/yws/res/14513/457E54FEAB52457E83EA28E4E8A03B4C)

## 总结
我认为Faster论文中比较值得借鉴的是高速hash index和epoch framework的设计思路, 虽然不需要多么高深的理论, 但是工程复杂度还是挺高的, 如此高效的设计值得学习. 其他部分我个人认为不太实用, 工程实现的复杂度比较高, 需要考虑很多corner case, 不如复用已有成熟的存储引擎比较好, 特别是核心思路和lsm-tree非常接近, 当然很多细节可能需要性能调优.

## 参考
   - https://youjiali1995.github.io/storage/faster/
