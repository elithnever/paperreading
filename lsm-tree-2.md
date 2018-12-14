# LSM-tree存储引擎的优化研究成果总结(2) -- 关于索引空间的优化

上一篇文章主要是从数学模型作为切入点, 对lsm-tree做了各种优化, 那么这篇文章则重点选择了几篇关于内存的优化文章. 我们都明白, 如果能够尽可能多的把索引放在内存中, 那么对于提升性能无疑是巨大的帮助, 所以这块也是lsm-tree存储引擎的研究热点.

## SlimDB: A SpaceEfficient Key-Value Storage Engine For Semi-Sorted Data
这篇论文来自2017年的VLDB会议. 我们知道, 传统的LSM-tree存储引擎, 存在写放大和空间放大问题, 因此LSM-tree的研究往往在读放大, 写放大和空间放大之间进行折中. 这篇论文观察到有很多workload, 他们的key存在所谓的semi-sorted特点, 具体的来说就是很多key存在共同的前缀, 比如推荐系统的特征存储, 文件系统的元数据存储, 图数据库存储等. 针对semi-sorted特点, 这篇论文提出了更优的索引和filter设计方法. 核心是提出了两个新技术: three-level block index和multi-level cuckoo filter. 在介绍他们之前, 先科普下cuckoo hash和cuckoo filter.

### cuckoo hash和cuckoo filter
bloom filter相信大家都比较熟悉了, cuckoo filter的作用和bloom filter类似, 但是相比于bloom filter可以提供更低的False Positive Rate. cuckoo filter的原理耗子哥写过一篇非常详细的中文材料, 大家可以从coolshell上找到. 我简单总结下核心思想. 首先说说cuckoo hash, cuckoo hash采用两个hash表存储数据, 设置两个hash函数, 对于每个key, 如果第一个hash表有空闲位置, 就插入到第一个hash表, 否则就插入到第二个hash表. 如果第二个hash表也被占用了, 那么需要把这个数据踢走, 反复上述过程, 直到踢的次数达到一个上限, 就认为hash表已经满了, 需要rehash了. cuckoo的名字也是由此而来, 因为cuckoo(布谷鸟)有一个恶习, 就是他自己不筑巢, 把蛋生在其他鸟类的鸟巢里, 但是他的蛋会先孵化出来, 从而挤占掉其他鸟. 虽然看上去cuckoo hash需要很多次数据踢出, 但是平均的插入开销却是O(1)的, 并且cuckoo hash的满载率可以达到80%以上, 所以是一个性价比很高的hash算法. 用cuckoo hash就可以直接起到filter的作用, 但是如果因为会涉及key的替换, 所以必须存储完整的key来计算hash值, 但是存储全部的key还是比较占用内存空间的. 在论文Cuckoo Filter: Practically Better Than Bloom中, 给出了partial-key cuckoo hashing算法, 为了节省存储空间, 只存储key的fingerprint值, 不存储原始的key来节省存储空间. 但是因为不存储完整的key, 那么就必须找到一种方法二次计算hash值. 为此partial-key cuckoo hash算法设计了一个很巧妙的hash函数, 下图是插入算法, 查找和删除算法参见论文. 论文中对cuckoo filter的存储空间和FRP之间的关系进行了数学建模, 每个bucket的slot数量越多, hash table的负载越高, fingerprint的bit越多, FPR越小. 当bucket size=4 的时候, 装载率可以到95%, 在相同FPR条件下, bloom filter的存储空间是cuckoo filter的1.44倍.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/73DD3FF483C4440ABDE56FAE54006414?ynotemdtimestamp=1544784896470)

### Entropy-Encoded Trie data structure
针对已经排序好的数据, 通常的做法是使用二分查找来查询具体的key. 进一步来说, 因为key有序, 所以可以用更高效的编码方式对key进行编码来构建一个索引, 加速查找. trie树, 也叫做前缀树, 可以作为这样的数据结构来使用, 如果key-value都是固定长度的, 那么可以针对key的最短unique前缀来构建索引, 而无须完整的key. 下图上半部分就是一个trie树的例子. 然而使用树形结构在持久化是非常消耗存储空间的, 因为有大量的指针, 所以需要对树形结构进行相应的编码. 编码的基本思路是递归: Repr(T) = |L| Repr(L) Repr(R), 其中|L|代表左子树叶子节点的个数详细方法参加论文: SILT: A Memory-Efficient, High-Performance Key-Value Store. 基本上编码完之后的效果就是下图下半部分所示. 论文中还给出了进一步的优化方法.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/F3D34CCC35B24124ADEBEB91FC1D6C15?ynotemdtimestamp=1544784896470)

但是ECT仅仅对固定长度的key和value有效, 如果key和value长度不固定, 那么需要进行一定的变种, 一种方法是存储把(key, value)替换成(offset, paitial of key value), 然后其余的部分, 存储在单独的地方, 但是这种方法需要多引入一次IO操作. 如果value足够小, 那么可以和key存储到一起, 减少二次IO的开销. 使用ECT编码, 论文中给出平均每个entry占用0.4 bytes, 但是没有找到证明过程.

### three-level block index
利用three-level block index技术, 替换掉rocksdb原生的block index之后, 可以优化index的存储空间到平均每个key 0.7bits. 下面来看看three-level block index的核心设计. 在leveldb里, 每个SSTable的最后都有一个index block, 存储每个data block的last key, 这样每次查找都需要基于index block做一次二分查找来定位到具体的数据. 在作者观测到的典型的workload中, 每个entry的大小不超过256 bytes, 如果block size是4KB的话, 每个block最多16个entry(4KB/256=16), index block里存储的是full key, 平均大小为16 bytes, 所以平均每个entry的index需要16B/16=8bits. 和leveldb不同, leveldb存储的key是全部有序的, 但是semi-sorted data只要求key的前缀有序, 所以这就让我们可以使用ECT编码来更高效的进行index的压缩, ECT平均每个entry使用0.4 bytes. 在semi-sorted data环境下, ECT可以达到平均每个key只存储2.5 bits的效果, 论文中进一步优化到了1.9 bits. 

但是ECT是为了index hash table设计的, 对于semi-sorted key来说, 单纯使用ECT不够高效, 所以这里设计了Three-level index. Three-level index的原理如下图所示:
   - 将每个block的first key和last key组成一个数组, 这个数组的前缀进行ECT编码, 重复的前缀会删除, 这就是第一个level
   - 第二个level存储第一个数组中的下标, 这样当给定一个key查询时, 就能筛选出一组可能存在这个key得SSTable
   - 为了进一步缩小查找范围, 每个block last key的后缀组成一个数组, 共享公共前缀的suffix仍然采用ECT编码, 这样就可以采用二分查找的方式定位具体的block了. 当然如果为了加速查找, 不使用ECT编码也可以.

因为ECT必须针对key-value定长的情况, 所以这里使用所有key的最长公共前缀进行ECT编码的, 但是不知道对于所有的key的后缀是否需要补齐长度, 论文里没有写, 我估计是需要的.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/62E4E141FC3B4AD58C25CE30CD9DDC58?ynotemdtimestamp=1544784896470)

简单分析下three-level block index的开销, 对于第一层, 每个block使用first和last两个key, 使用ECT之后, 平均每个key 2.5 bit, 所以每个block需要5 bit. 第二层存储的是个数, 可以使用差分编码, 平均每个block也是2.5 bit. 第三层仍然是ECT编码, 每个block一个key, 那么平均每个block占用2.5 bit. 所以平均每个block需要占用10 bit(5 + 2.5 +2.5). 如果每个block平均存储16个entry, 那么每个entry相当于10/16=0.7 bit, 远远小于leveldb的8 bit.

### multi-level cuckoo filter
因为cuckoo filter中存储的是key的fingerprint, 所以存在一定冲突的可能, 而冲突就会产生False Positive的读请求. 为了降低在冲突情况下的读延迟, 引入了main table和secondary table. main table存储fingerprint和level, 这样可以快速定位key所属于的level. secondary table中存储了main table中冲突的entry的full key和对应的level, 基本逻辑如下图所示.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/144FFF95721847A4A53CA3EDF58AFFF2?ynotemdtimestamp=1544784896470)

下面这幅图显示了SlimDB和原生levelDB的性能对比, 可以看出来SlimDB的内存节省还是挺明显的, 详细的测试数据大家看论文吧.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/12C49C541BCF4A51AE7EDF08A384F94B?ynotemdtimestamp=1544784896470)

## LSM-trie: An LSM-tree-based Ultra-Large Key-Value Store for Small Data
既然提到了trie树, 就一起来看看这篇论文吧. 这篇论文发表自2015年的ATC(USENIX Annual Technical Conferene), 核心出发点就是希望用trie树结构来精简索引的大小, 来降低大量小key-value数据而造成的内存膨胀问题. 但是为了使用trie树, 就必须采用类似sha-1的hash算法, 将原始的key计算出来的hash值来替换掉原始的key, 因为sha-1 hash之后, key是定长的而且公共前缀更多了, trie树的优势才能充分发挥出来. 下图所示的结构就是一种所谓SSTable-trie的结构, 和SSTable相比就是把hashkey存在levelDB里了. 但是这还不够高效, 毕竟还是存在索引的, 将原始key转换成hashkey之后, 就无法保证key有序的特性了, 所以论文干脆引入了HTable的思路, 这就是LSM-trie. 简单点说, 就是在给item分配block的时候, 使用hash而不是基于hashkey排序了, 不过使用hash就容易导致不同的block之间数据不均衡, 所以还需要对特别不均衡的block进行一定程度的修正, 这里面细节比较多. 同时还需要引入一种新的bloomfilter算法, 这块大家感兴趣就去看论文吧.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/36E95C08EA5240D9853DE78B79F6D22E?ynotemdtimestamp=1544784896470)

## Optimizing Space Amplification in RocksDB
这篇论文比较简单, 论文的核心内容是基于rocksdb研发了MySQL的存储引擎, 在facebook的workload下, 替换InnoDB之后大约降低了50%的存储空间. 论文中介绍了RocksDB使用的一些空间压缩技术, 主要包括:
   - 为不同的层设置不同的压缩算法. 比如最后一层包含90%的数据, 但是被读取的概率相比其他层要小一些, 所以最后一层设置更高压缩率的算法, 相对来说性价比更高, 而低层因为数据量少, 访问频率可能更高, 可以采用相对低压缩率的算法, 节省压缩和解压缩的CPU开销, 提高访问性能.
   - 动态条件每个level的size. 我们知道传统的leveldb实现中, 每个level的size是固定配置的, 但是在最坏情况下, 最后一层可能不是上一层的10倍, 因为最后一层可能数据不满, 这时候空间放大就不是1.1倍了. RocksDB引入了动态调整每个level size的技术, 仍然保证上下两层的size在固定的倍数, 比如10, 这样可以保证空间用于是1.1倍.
   - Key的前缀压缩, sequence ID的垃圾回收, 基于词典的压缩算法等.
   - 前缀bloomfilter. 对于scan来说, 传统的bloomfilter就起不到作用了, 然而很多scan请求都是基于共同前缀的, 比如如果key是(userid, timestamp), 那么userid很可能会成为前缀. rocksdb开发了prefix bloom filter, 由用户传入一个prefix extractor, 来实现prefix bloom filter, 在facebook的workload上可以降低64%的读放大.

论文的实验环节介绍了很多在facebook workload下的实验数据和参数, 来证明使用RocksDB替换innodb之后的效果, 详细数据可以参考论文原文.

## Accordion: Better Memory Organization for LSM Key-Value Stores
这篇论文发表于2018年的VLDB, 论文提出了优化内存分配和使用的方法. 这篇论文主要针对HBase做了内存方面的优化, 核心思路是一方面在memtable中提前做compact, 并且将immemtable的index进行flatten, 从而节省内存空间, 这一点和tera的优化思路类似, 另一方面用Java的堆外内存技术, 将内存组织成相对较大的数据块, 提高内存管理的效率, 降低GC的影响. 因为Java有复杂的内存管理和GC机制, 所以单纯的在内存里进行compact不一定能提升性能, 因为compact过程可能会free大量小对象, 对GC和内存管理会带来负担. 所以HBase 2.0提供了2个策略, 一个叫做basic策略, 在内存里不进行数据删除, 只对索引进行flatten, 另一个种策略叫做Eager, 非常激进的进行数据删除, 来应对数据频繁变化的workload场景. 这两种策略相对死板, 所以Accordion提出了一种adaptive的策略. Accordion设计了2是个参数, 一个是t, 随着内存的增加而增加, 一个是u, 记录unique的key的比例, 通过给t和u设置阈值, 来决定是否进行数据删除. 

整体上这篇论文比较偏工程化, 而且很大一部分是结合Java内存管理机制进行优化的, Java堆外内存优化这部分细节可以详见论文.
