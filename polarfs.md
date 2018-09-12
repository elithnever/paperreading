# PolarFS: An Ultralow Latency and Failure Resilient Distributed File System for Shared Storage Cloud Database

PolarFS是阿里云基于NVMe SSD和RDMA网络构建的具备低延迟的访问的新一代分布式文件系统. 这篇论文发表于2018年VLDB上. 本文对论文的要点和自身的理解进行记录, 起到帮助大家快速了解论文的作用.

## 背景
现代存储系统的研究开始密切关注新硬件对于存储系统的影响, PolarFS就是构建在新一代硬件上的分布式文件系统. 作为新一代SSD, NVMe SSD可以提供500K的iops, 平均延迟100us左右, 最新的3D XPoint SSD甚至可以将延迟降低到10us左右. 特别是最近Intel发布的SPDK开发套件, 可以bypass掉内核从而降低内核层面的软件开销. RDMA提供了低延迟的网络通信接口, 远远低于TCP/IP协议栈的网络接口. 应用程序通过API操作RDMA的三个队列, 分别是发送队列, 接收队列和完成队列. 发送和接收队列负责进行数据传输, 完成队列用来轮询数据传输完成的事件. 为了减少线程间切换的开销, PolarFS轮询完成队列.

## 架构说明

![image](http://note.youdao.com/yws/public/resource/4daef1dc09efae12290cb8e2d4b3209e/5A3FF9599E9947DA98FD0B1B26585953?ynotemdtimestamp=1536742335888)

如上图所示, PolarFS由4个部分组成, 分别是libpfs, PolarSwitch, ChunkServer和PolarCtrl. 从逻辑上说, PolarFS架构分成两大部分, 上层是文件系统层, 由libpfs和PolarCtrl共同完成文件系统层的抽象功能, 其中libpfs主要负责缓存元数据和向存储层发送IO请求, PolarCtrl主要负责处理元数据的读写请求(注: 这里说的元数据仅仅是与chunk server有关的元数据而不是文件系统层面的元数据). 下层是数据存储层, 由PolarSwitch, ChunkServer和PolarCtrl组成. 其中PolarSwitch负责缓存元数据, 并且转发IO请求, ChunkServer基于raft协议处理数据读写请求, PolarCtrl负责维护ChunkServer的状态和处理元数据的读写请求. 下面我们分别看看每个部分的详细功能.

   ### &ensp;&ensp;&ensp; 1.libpfs 
   libpfs以一个程序库的形式存在, 提供posix兼容的API接口, 同时缓存所有的元数据, 将posix接口的文件读写请求, 转化为下层的IO存储请求, IO存储请求由(volumeid, offset, size)这个三元组组成. volume在PolarFS中是一个逻辑概念, 类似于Linux LVM中的逻辑卷. 物理上volume由一组chunk组成, volume的大小可以从10GB到100TB之间. chunk是ChunkServer的数据管理单元, 也是raft复制组的最小单元, 每个chunk的大小为10GB. 在每个chunk内部, 划分成一系列定长的block, 每个block的大小为64KB. block是数据读写的最小粒度. 在使用文件系统之前, 需要调用pfs_mount接口, pfs_mount从PolarCtrl加载所有的元数据, 在内存中构建文件系统层元数据的数据结构(注: 论文中没有说元数据的大小, 能否cache所有元数据). 元数据结构如下图所示:
   
   ![image](http://note.youdao.com/yws/public/resource/4daef1dc09efae12290cb8e2d4b3209e/EDCB5182E1C24132BD8948F436DADA46?ynotemdtimestamp=1536742335888)
   
   文件系统层的元数据主要包括3个部分, 分别是dir entry, inode和block tag. 这3部分和普通的本地文件系统类似, 比较容易理解. 根据这些元数据, libpfs就可以把posix接口的读写请求, 转化成ChunkServer层的读写请求了(volumeid, offset, size). 所有的元数据都用metaobject来表示和持久化, 为了修改或者更新一个或者多个metaobject, 需要将metabobject的修改封装成一个事务来执行. 为此PolarFS将事务的写请求保存在journal文件里, journal文件逻辑是一个循环队列, 同时为了确保多个进程安全的并发读写journal, PolarFS使用Disk Paxos算法实现了分布式抢锁的逻辑. (注: 从上面的图上推测, metaobject最终也是要持久化到chunk上的, 只不过为了保证原子性, 在每个libpfs进程里使用journal文件实现类似WAL的效果. 同时为了实现并发控制, 基于DiskPaxos实现了分布式锁.) Disk Paxos算法是paxos算法的一个变种, paxos算法加锁每个节点只能读写自己的磁盘或者内存, 使用消息传递的方式交换信息. 而Disk Paxos算法假设每个节点可以读写所有节点的磁盘, 从而将节点之前的消息传递改成读写本地和其他节点的磁盘数据, 这个算法对节点进程是否存活没有要求, 只要能成功写入多数派磁盘就可以保证信息一致, 适用于类似SAN这种共享存储的场景, 详细的算法大家可以搜索原始论文(笔者猜测是因为PolarFS基于RDMA通信, 延迟和可靠性和SAN类似, 所以采用了Disk Paxos算法, 不过没想明白为什么只是用于抢锁而不是也一起传输journal, journal是如何同步到每个libpfs进程的? 如果有同学有想法, 感谢回复我. 有一种可能是journal是存储在共享存储上, 这样方便所有的libpfs节点同步元数据的更新). 上面的图也说明了元数据的更新过程:  
      &ensp;&ensp;&ensp; 1. Node 1分配块201至文件316后, 请求互斥锁并获得锁  
      &ensp;&ensp;&ensp; 2. Node 1开始记录事务到journal中, 最后的写入项标记为pending. 当所有的项记录之后(注: 这里不太确定是不是说所有libpfs进程都收到记录之后), pending tail标记成journal的有效tail.  
      &ensp;&ensp;&ensp; 3. Node 1将元数据持久化到superblock(注: 应该是chunkserver中的block), 更新内存. 于此同时, node 2尝试获取互斥锁, node 2会失败重试.  
      &ensp;&ensp;&ensp; 4. Node 2在Node 1释放互斥锁之后抢锁成功, 读取journal的数据, 更新内存中的元数据.  
      &ensp;&ensp;&ensp; 5. Node 3抢锁成功之后, 读取journal更新本地内存.  
   
   ### &ensp;&ensp;&ensp; 2. PolarSwitch
   PolarSwitch是一个daemon进程, 相当于一个代理, 部署在每个libpfs的节点上. PloarSwtich接收到libpfs的请求后, 根据请求的信息, 转发给一个或者多个chunk server. PolarSwitch会混存元数据信息, 并且和PolarCtrl同步. 为了提升libpfs和PolarSwitch之间数据传递的性能, libpfs和PolarSwitch之间使用共享内存, 组织成一个环形缓存.
   
   ### &ensp;&ensp;&ensp; 3. ChunkServer
   ChunkServer是数据的持久化层, 为了减少线程同步和资源争抢的开销, 每块NVMe SSD盘部署一个ChunkServer进程. 每个ChunkServer都有一个Write Ahead Log, 每个写请求都会先写入WAL, 然后在更新到block里, 为了降低延迟, PolarSwtich使用3D Xpoint SSD来保存WAL. ChunkServer之间使用ParallelRaft协议来实现数据副本的复制. 当ChunkServer之间负载不均匀或者局部故障时, chunk可以在不同的ChunkServer之间移动.
   
   ### &ensp;&ensp;&ensp; 4. PolarCtrl
   PolarCtrl是PolarFS的总控模块, 主要职责包括:  
   - 追踪ChunkServer的列表和存活性, 对ChunkServer进行负载均衡
   - 维护volume的状态和chunk位置信息
   - 创建volume, 给ChunkServer分配chunk
   - 向PolarSwitch同步元数据
   - 收集chunk的延迟和iops
   - 周期性的发起副本内和副本间的CRC数据校验
   
## IO执行流程
![image](http://note.youdao.com/yws/public/resource/4daef1dc09efae12290cb8e2d4b3209e/A1D9F7DDA1F847DD81B73B3313FF2639?ynotemdtimestamp=1536742335888)

了解了上述每个部分之后, 我们看看整体的IO执行流程. 详细的IO流程在论文里写的比较清楚了, 大家可以参考论文的说明. 为了提升IO的性能, PolarFS没有使用linux的文件系统, 而是使用SPDK读写NVMe SSD, 这样通过bypass内核的方式来提升IO性能, 基本上已经成为NVMe SSD使用的主流方式了.

## ParallelRaft
数据一致性协议基本上就是paxos和raft了, 但是众所周知, raft协议不允许日志出现空洞, 也就是说无论是leader还是follower, 都必须按顺序提交或者确认, 所以并发的写请求无形中就被串行化了, 这也是为什么很多数据库采用paxos而不是raft的原因. PolarFS设计了ParallelRaft协议来解决这个问题. ParallelRaft的核心思路是对于没有重叠的写请求, 可以允许出现空洞从而并行提交, 不会影响正确性, 但是对于有重叠的写请求, 则必须严格按照顺序提交. 判断是否重叠的方法是给每个log entry增加LBA信息, LBA包含之前还所有还没有提交的写请求的区间, 从而提供了判断写请求是否重叠的方法. 在工程实践上, 可以设置一个参数N, 作为允许的最大空洞的log entry个数, 这样可以保证空洞不会被无限放大, 减少LBA的维护成本. 在PolarFS的RDMA网络环境下, N设置成2就足够了.

在选举Leader的时候, Raft协议要求新当选的Leader必须包括最新的term和最长的log entry, 但是和Raft不同, 新当选的Leader可能存在空洞, 所以ParallelRaft需要一个merge过程来弥补空洞数据. 在merge过程中, leader处于leader candidate状态, 论文中详细的对需要考虑的各种边界情况进行了描述, 详细过程可以参考原文. 对于follow的数据复制过程, PolarFS同样业设计了fast-catch-up机制和streaming-catch-up机制来追赶leader的数据, 思路和一般的raft算法类似. 下图是raft算法和ParallelRaft的性能对比数据, 从数据上看吞吐提升还是挺明显的.

![image](http://note.youdao.com/yws/public/resource/4daef1dc09efae12290cb8e2d4b3209e/BF1E8E74E33B4FDA9E9E5B9ECA4ACDC7?ynotemdtimestamp=1536742335888)

## 性能测试
测试环境由6个存储节点和1个客户端节点组成, 节点之前使用RDMA通信, 使用fio产出测试数据.

![image](http://note.youdao.com/yws/public/resource/4daef1dc09efae12290cb8e2d4b3209e/1ABE0EF2519D4F19B654E3458DDDD63C?ynotemdtimestamp=1536742335888)

从论文上的数据来看, IO的延迟已经比较接近本地文件系统了, 而且比CephFS要好不少.

![image](http://note.youdao.com/yws/public/resource/4daef1dc09efae12290cb8e2d4b3209e/644CA18ACE5A46F2B78DC1EE2EC652BA?ynotemdtimestamp=1536742335888)

吞吐上看也远远好于CephFS, 和本地文件系统的差距也不大. 论文中还有阿里云RDS, PolarDB on PolarFS和PolarDB on ext4的性能对比数据, 详细信息可以参考论文.

## 设计抉择和经验教训
   - 中心化还是去中心化. PolarFS的chunk server层采用了中心化的设计思想, 上层文件系统层采用了去中心化的设计思想, 可以说是中心化和去中心化的折中.
   - 从底向上的snapshot. 快照是数据库的普遍需求, PolarFS实现了所谓的disk outage consistency snapshot机制, 并且制作快照期间不会影响用户的读写请求. ChunkServer基于Copy-On-Write机制实现了快照功能, 论文中写的不是很详细, 细节不太明白.
   - 外部服务和内部可靠性. 新节点的加入或者chunk的迁移, 需要同步大量数据, 这往往需要消耗较多资源. 为了减少对外服务的影响, PolarFS将每个chunk切分成128KB的数据块, 进而将数据同步任务切分成很多子任务. 这些小的子任务执行的快, 可预期性好, 更有利于ChunkServer进行调度. 其他类似的后台维护工作, 也采用类似的方法解决.

## 总结和思考
PolarFS结合新型硬件设计了更符合现代硬件的分布式存储系统, 其设计思路和工程实践都有很高的参考价值. 不过需要注意的是, PolarFS的应用场景不是为了替换类似GFS/HDFS这种大规模分布式文件系统的, 更多的是为了满足分布式NewSQL数据库而设计的, 从支持最大的volume是100TB就可见一斑.
读完论文之后, 还有一些疑问, 记录在最后, 欢迎大家和笔者讨论.
   - 去中心化的文件系统层. 当节点较多时, 同步文件系统元数据可能开销较大, 这里面可能需要考虑比较多的工程问题.
   - 文件系统元数据的更新和同步机制比较复杂, 不太确定journal文件是存储在每个libpfs节点上, 还是存储在小型高性能共享存储上(比如SAN)? 如果不是存储在共享存储上, 那journal数据的同步机制是什么? 为什么不用一致性协议来实现数据复制?
   - 多个libpfs为什么使用disk paxos协议进行抢锁, 选择disk paxos协议的初衷是什么?
   - 既然ParallelRaft主要是为了解决日志空洞问题, 那为什么不直接使用Paxos协议?

## 参考文献
   - PolarFS: An Ultralow Latency and Failure Resilient Distributed File System for Shared Storage Cloud Database
   - [面向云数据库，超低延迟文件系统PolarFS诞生了](https://mp.weixin.qq.com/s/4s7lDKlQjV1mUoVv558Y7Q)
