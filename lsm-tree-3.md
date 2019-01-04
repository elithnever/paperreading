# LSM-tree存储引擎的优化研究成果总结(3) -- 架构的优化
## Scaling Concurrent Log-Structured Data Stores

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/6BF0395B9FD14BF19C150C8229E687FA?ynotemdtimestamp=1546605107269)

如上图所示, LSM-DS的模型可以抽象成上图的形式, 任何数据的读写请求, 都会涉及Pd, Pm和P'm这三个指针, 同时后台的compact任务也需要访问和修改这3个指针. 那么这样一来, 这三个指针就必须进行一些同步的操作来保证正确性, 这篇论文的核心就是提供了一组算法来最大化的降低锁竞争和提升并发度, 进而提升性能. 上图中抽象的模型和三个指针的含义非常容易理解, 大家看图中的描述文字吧, 就不多解释了.

为了实现高并发, 论文设计了两个钩子函数, 分别是beforeMerge和afterMerge, 在compact(或者说merge)之前和之后调用. compact过程(或者说merge过程)结束之后, 返回一个新的指针指向disk的component, 并且作为参数传递给afterMerge函数. 如果内存中的memtable是多线程安全的, 那么get请求无须加锁, 因为即使在get操作过程中, 这3个指针发生了变化, 那么也不影响正确性, 最坏情况是有些component被访问了两次. 但是put操作就需要精心设计了, 防止put数据到无效的内存component上. 为此论文引入了读写锁来对读写操作进行同步控制. 基本的算法如下图所示:

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/75D8F320D9764BBAA28D060913315352?ynotemdtimestamp=1546605107269)

上面的算法没有考虑snapshot的功能, 类似levelDB, 我们可以用时间戳来实现snapshot功能, 但是引入snapshot功能之后, 算法需要考虑更多关于snapshot的细节, 优化之后的算法如下图:

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/44AAD29310594C89ABC2736D1064665C?ynotemdtimestamp=1546605107269)

   - 需要一个额外的active表来记录所有被snapshot的timestamp
   - put的流程没有太大的变化, 仅仅是在插入memtable之后, 多了一个getTs()的函数调用, 返回合适的ts, 并且把ts从active表中删除
   - 多了一个GetSnap函数, 论文中详细解释了如果只选择当前时间直接作为snapshot timestamp的问题, 本质上就是因为get或者put操作需要持续一段时间, 所以算法进行了优化来解决这个问题. 方法就是维护active表, 当然active表尽可能lockfree来做到nonblocking. getSnap会选择比所有active表里timestamp都小的一个作为timestamp. 考虑到getSnap也支持并发操作, 需要仔细更新snapTime变量, 为此引入了CAS操作.
   - getTs操作会选择大于snapTime的最小timestamp返回

snapshot的功能让原本简单的流程复杂化了, 论文中还给出了read-modify-write的算法, 详细算法大家阅读论文吧. 在实现上, cLSM基于levelDB修改了代码. levelDB使用了一个全局mutex来保护临界区, 因为只有单个写线程, 所以没有引入类似active表这样复杂的机制. cLSM使用上述算法支持了原生levelDB的所有接口, 尽可能的消除了代码中需要block的地方. 由于cLSM消除了单个写线程的限制, 所以log中的数据可能会出现无序现象, 不过由于每个item都关联了timestamp, 所以按照时间来recover日志也非常容易.

总结下, 论文通过巧妙的设计并发控制算法, 最大限度的减少了代码的临界区, 提升了读写并发度. 不过我认为支持并发写对性能提升帮助不大, 因为不管是SSD还是机器硬盘, 顺序写的性能都要高于并发写. 对于读性能提升来说, 从论文中给出的数据来看效果不错, 提升了一倍以上, 不过这只限于当数据读请求的瓶颈到达CPU的时候, 因为绝大部分瓶颈都在存储设备上而不是CPU. 所以整体上看论文的研究成果实用性不是特别强, 仅仅对少量情况有性能提升的效果和优势, 另外这篇论文的正确性缺乏论证, 如果论文能够提供一份机遇TLA+的形式化证明, 可能就更完善了.

## PebblesDB: Building Key-Value Stores using Fragmented Log-Structured Merge Trees
这篇论文发表在2017年的SOSP上, 读完之后发现论文的思路和dostoevsky有异曲同工之妙. 我们知道LSM的模型主要的写放大在于compact, 特别是对于类似levelDB这种leveling compact style来说, 需要多次读写level i和level i+1的数据, 因此会引入更多的IO操作. 为此RocksDB引入了tiering compact style, 仅仅在同一个level上进行compact, 不引入层与层之间的compact. 但是这种策略对读请求来说却非常不友好, 因为每次读需求多次IO. 

PebblesDB引入了所谓的Fragmented Log-Structured Merge Trees结构, 思路来源于skip list. 其实现在看来, 和tiering compact style也非常类似, 核心思路如下图所示. 在每个level中, 设置若干个guard, 同一个level上这些guard包含的key range是不能交叉的. 每个sstable根据key range来决定属于某个guard. 同一个guard内, compact的逻辑就非常简单, 仅仅进行merge形成一个新的sstable下沉到下一层就ok了. 这样每个sstable在每一个level内最多只会merge一次了, 可以减少大量的读写IO. 不过因为最后一层的sstable不能下沉了, 所以最后一层就必须进行rewrite了, 这样可以减少最后一层key range交叉的sstable数量. 同时倒数第二层也有可能进行rewrite, 比如当最后一层对应的guard数据满了的时候. 当然论文中还包括不少细节, 比如如何选择guard的key range, 如何并行compact, 如何添加或者删除guard, 如何条件参数, 分析算法复杂度等等. 相信如果看完dostoevsky这篇论文之后, 会理解的更为透彻, 因为dostoevsky像是PebblesDB的加强版.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/09DD50AFD2564A6F86C7C87E97BBBEEA?ynotemdtimestamp=1546605107269)

## WiscKey: Separating Keys from Values in SSD-Conscious Storage
这篇论文可以说是提出了一个非常开脑洞的想法, 而且想法还非常容易理解. 那就是既然LSM-tree模型虽然实用, 但是存在非常严重的读写放大问题,  而产生读写放大的根源就是需要不停的compact. 那如果把value独立存储到一个叫做vLog的文件里, 使用lsm-tree只存储索引的话, 就可以极大的减少lsm-tree后台compact的开销了. 不过想法虽然简单, 却仍然有不少问题需要考虑, 这篇论文针对这些问题给出了相应的解法, 下面针对每个问题, 总结下对应的解决方法.
   - 并行读优化. key value分开之后, value数据无序了, 这样无疑会增加scan的开销, 把顺序IO转变成随机IO了. 针对SSD, 论文中对顺序读和随机读进行了测试, 发现当一次读的数据块大于一定程度(论文中测试的数据是64KB)时, 并发随机读和单线程顺序读的读吞吐基本上接近了. 由此很自然的想法就是用并发读(32线程)来优化, 并且针对scan迭代器的Next()和Prev()接口, 进行预读优化.
   - GC优化. value保存在独立的vLog里, 不能和LSM-tree一起GC了, 所以需要单独设计GC机制. 简单的方法就是首先扫描LSM-tree的index, 所有在vLog中但是不在LSM-tree中的数据视为无效, 但是这种方式显然太重了. WiscKey提供了一种轻量级的方法. 在vLog里保存完整的key value数据, 并且使用一组head和tail指针来记录有效数据的区间. 在GC过程中, 从tail开始读取一组数据, 然后和lsm-tree的index进行比较. 如果数据有效, 则append到head位置, 然后更新tail指针. tail和head指针非常重要, 也需要持久化到lsm-tree中. 需要注意的是, 当value被append到vLog之后, 需要调用fsync()同步数据. GC过程无疑会产生文件的空洞, 为此可以使用fallocate()函数来高效的删除空洞.
   - Crash Consistency. LSM-tree可以保证插入的key value是原子的, 并且及时crash也仍然能够恢复. 但是引入了vLog之后, 灾难恢复的过程就会变得复杂一些. WiscKey的灾难恢复过程使用了现代文件系统的一个特性, 那就是如果数据X在写入过程中crash了, 那么X以后的数据一定不会成功写到vLog文件里. 为此在数据恢复的过程中, 需要验证LSM-tree的index指向的数据和vLog里的数据是否匹配, 如果不匹配则认为数据丢失了, 从LSM-tree中删除相应的index.
   - LSM-tree log优化. LSM-tree中通常包含一个log文件来实现数据持久化, 考虑到vLog中已经包括了完整的key value对了, 因此LSM-tree的log理论上就可以优化掉. 一个简单的想法就是启动后扫描整个vLog肯定能恢复完整的数据. 但是这种方法显然效率太低了, 为此在LSM-tree中记录vLog的head指针, 这样启动的时候就只需要从head开始扫描了.

WiscKey这篇论文虽然思路不难, 但是却影响很大, 目前很多LSM-tree的存储引擎都在朝着这个方向优化. 但是仔细思考会发现GC过程实际需要考虑的问题比论文中还要多, 一个特别需要考虑的问题就是GC和真是写请求之间的并发控制问题. 假如GC和真实写请求在更新同一个key, 那就必须在GC的时候hold住真实写请求, 防止真实写请求的index被GC所覆盖. 但是这样block住写请求, 又会降低并发, 所以需要一种更好的解决方法了. 
