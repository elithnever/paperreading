# LSM-tree存储引擎的优化研究成果总结(1)
基于LSM-tree的存储引擎自从Google发表了bigtable论文之后, 就变得异常火热起来, 工业界和学术界先后做了大量的研究和优化. 为此, 我计划把比较知名的论文整理下, 整体学习和思考下业界的研究思路和成果. 

## LSM-tree存储引擎建模

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/A2FC734594864994AAB0822F6E8C3625?ynotemdtimestamp=1541669980860)

本文假设读者对LSM-tree存储模型有一个基本的理解. 为了分析LSM-tree存储引擎的性能开销, 需要先进行数学建模. 在LSM-tree中, 内存有M(buffer), M(filter)和M(pointers)组成, 磁盘上分成多个Level, 每个Level有若干个sorted runs组成, Level与Level之间, 容量相差T倍. LSM-tree模型存在两种典型的merge策略, 叫做tiered和leveled. 二者的区别如下图所示:

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/E6ECC0E47D684464B4F56FDE356B662F?ynotemdtimestamp=1541669980860)

tiered允许每个level最多有T个runs, 每次merge的时候, 都是从选择L层所有的runs, 然后merge成一个大的runs放在L+1层. 而leveled则每次从L层和L+1层选择runs的一部分, 生成L+1层runs的子集, 也就是说, leveled每个Level只允许最多一个run存在(L0层除外). 这样对于一个LSM-tree来说, 包含的参数就包括M(buffer), M(filter), T, 总的entry数量N, entry的平均大小E和Block的大小B以及merge策略. 针对这些输入参数, 需要优化的目标就是update cost, point lookup cost, range lookup cost了. 那么如何衡量呢? 很明显的一个方法就是用IO次数, 因为不管存储介质是机械硬盘还是固态硬盘, LSM-tree的存储模型瓶颈往往在于IO次数. 那么建模的目标就变成了在不同参数条件下如何计算各种条件的IO次数了. 但是对于查询来说, 需要考虑bloom filter的影响, 而bloom filter是一个概率模型, 分析概率模型的方法一般是取最坏/最好/平均情况, 这里我们选择最坏情况, 因为在false positive情况下, 会发生无效IO, 无效IO对于zero-result lookup还是很伤的. 有了目标, 计算各种cost相对来说就比较容易了, 下图给出了各种情况下的计算结果.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/694E37C78D7E401FBF5992458DF9D748?ynotemdtimestamp=1541669980860)

论文里给出了解释, 其中e^(-M(filters)/N)表示bloom filter的FPR. 对于tiered来说, 因为每层最多T个runs, 所以point lookup cost就是O(L * T * FPR). 而对于leveled来说, 每层最多一个run, 所以point lookup cost就是O(L). update的开销主要来自于同一个entry的merge次数. 对于tiered来说, 每个entry在每个level只会merge一次, 同时每次merge最小的粒度都是block, 所以平摊到每个entry的update cost就是O(L/B). 而对于leveled来说, 每个entry在每个level平均拷贝O(T)次, 所以update cost就是O(L * T / B). 注意T的区间是 2 <= T < Tlim, 其中Tlim=N * E / M(buffer). 不难想象, 当T=2时, 对于tiered来说, 模型退化成log. 当T趋于Tlim时, 对于leveled来说, 模型退化成sorted array.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/73F56079BFB44FF58975EC3E3BBEDD2D?ynotemdtimestamp=1541669980860)

当把模型的参数画在坐标系中的时候, 如上图所示, 就构成了整个LSM-tree模型的设计空间了. 再次感受到了数学的魅力, 有了这个数学模型, 那么就有了明确的优化空间了, 而且各种优化手段也不会超越这个设计范畴了. 更详细的数学建模过程大家可以参考monkey和dostoevsky的论文, 二者对比着看会更清楚些.

## Monkey: Optimal Navigable Key-Value Store
上面介绍的数学模型为优化LSM-tree提供了设计的空间, 主要从三个方面考虑:
1. 如何调整M(filter)的大小来减少FPR
2. 如何分配M(buffer)和M(filter)的大小
3. 如何调整T和选择merge策略, 这往往和workload有密切关系

目前现有的LSM-tree的存储引擎针对上述设置往往是静态的, 为此作者提出了Monkey的设计思路, 希望可以快速准确的在上面的设计参数中进行选择和调节, 从而实现在给定memory和workload的情况下, 让lookup cost和update cost达到最佳平衡点. 核心思路上, Monkey主要的贡献在于: 设计可调节的开关, 通过调整M(filter)最小化lookup cost, 性能预测, 根据历史数据自动化参数调整, 下面重点总结下Monkey的核心思路.

### 1. 最小化查询开销
最坏情况下的查询开销取决于FPR, 所以平均最坏情况下的查询开销定义参见公式3. 对于leveled来说, 每个level只有一个run, 所以R等于每个level的FPR加和. 对于tiering来说, 每个level最多T-1个run, 所以要乘以T-1. 根据FPR的定义, 不难计算出总的M(filter)的大小, 参见公式4.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/B0416B9B475741EBB7805FDCCCF307DB?ynotemdtimestamp=1541669980860)

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/7D899DC566C04354927D9005FCDEBF73?ynotemdtimestamp=1541669980860)

有了公式3和公式4, 现在的问题变成了在R一定的情况下, 如何让M(filter)最少呢? 在论文的附录B中, 作者给出了数学证明, 我们直接看论文中的结论, 参见公式5和公式6. 简单点说就是在R给定的情况下, 如果按照指数递增的方式逐步提升FPR, 可以实现M(filter)最少. 直觉上来说, level越大, 包含的entry数量越多, 分配的bits越少, 才能让filter占用的总内存尽可能少. 以leveldb默认的pi=1%, L=7来说, 代入公式5计算可以节省大约60%的内存, 相当于层次越高, bits越少.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/BDCA6CB73A2A49A596F6AFF22F6A3D33?ynotemdtimestamp=1541669980860)

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/FEDC648B154A4035972AA9B32C26F78A?ynotemdtimestamp=1541669980860)

### 2. 预测查询开销
对于zero-result lookup cost R来说, 结合公式4, 5和6, 可以推导出公式7和8. 公式8给出了M(filter)的变化对R的影响, 一般情况下, 在内存允许的条件下, M(filter)至少需要M(threshold)大小. 不过这只是在查询不到数据的情况, 对于最坏能查询到数据的情况也比较简单, 肯定是数据处于最后一层的情况, 相应的查询开销V=R-P(L)+1.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/B1B4E8636AEA46129AEDEE1C8ACCB924?ynotemdtimestamp=1541669980860)

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/9B41F36D55C04B82B9EB6BAE92F00316?ynotemdtimestamp=1541669980860)

### 3. 写入开销和区间查询开销建模
思路仍然和上面类似, 构造一个最坏的写入模式, 也就是每个entry经过N个写请求之后, 最多更新一次, 这样这个entry一定会下沉到最后一层, 这中间的merge操作不会删除这个entry. 这样每个level的平均merge次数分别是(T-1)/T和(T-1)/2, 对应tiering和levelding. 由于每个entry一定会从L1下沉到最后一层, 所以总的次数乘以L, 又由于每次merge最小移动B个entry, IO次数还需要除以B, 所以得到公式10. 另外由于读和写操作的开销不一样, 所以增加一个因子代表读和写的开销比例.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/084D6354A85E4D6FA23999C1BFAE2676?ynotemdtimestamp=1541669980860)

对于区间扫描来说, 最坏情况下, 每个level都存在数据, 所以每个level的每个run都需要一次IO, 那么leveling模式就是L次, tiering模式就是L*(T-1)次. 假设s代表每个run的平均扫描页面个数, 那么写入区间查询的开销就是公式11.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/4BA3E56792754AB2A19D58784AF8EAF0?ynotemdtimestamp=1541669980860)

### 4. 可扩展性和可调节性

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/57CFE3FCE4F84CF18AC7506F55033039?ynotemdtimestamp=1541669980860)

根据上面介绍的公式7, 公式8和公式10, 在上面的表格中显示了monkey和主流lsm-tree存储系统的各纬度开销差异. 论文中对上面表格中的性能差异进行了分析, 主要观点有:
   - M(filter) > M(threshod), 查询的复杂度是O(Rfiltered), 否则复杂度是O(Runfiltered). 
   - 当T=2的时候, 每个entry平均分配的bloomfilter的bits数大概是1.44, 通常leveldb配置的是10bits, 远远大于1.44. 所以主流系统的主要查询复杂度是O(Rfiltered)
   - R是一个与层数L无关的函数, 评估开销的时候不需要考虑L的影响.
   - R与entry size无关, 因为bloomfilter只与entry的数量有关, 与entry的大小无关. 而且与整体的buffer size也无关, 这样也无须平衡总的内存和bloomfilter的内存占用了. 当然这里是最坏情况下的查询开销(也就是数据不存在的情况), 对于数据存在的情况, 如果数据能缓存在内存里, 肯定能提升查询性能了.

论文还给出了调节吞吐, 参数T, merge策略, 内存的方法以及实验数据, 详细内容请大家阅读论文, 论文附录中还有公式的详细推导, 不需要很复杂的数学知识, 大家看的时候也不用有畏惧的心态. 论文的实验中全部关闭了block cache, 在附录里补充了开启block cache的测试结果, 仍然符合上述规律. Monkey这篇论文创造性的用数学知识推导出了一个非常容易实现的做法, 就达到了内存, 查询延迟, 吞吐的提升, 真的是非常神奇的工作, 向作者致敬.

## dostoevsky: Better Space-Time Trade-Offs for LSM-Tree Based Key-Value Stores
dostoevsky在monkey建立的数学模型基础上, 进行了进一步的优化. dostoevsky主要的启发来源于最后一层entry数量最多, 也就贡献了最多的lookup cost和update cost, 所以如果能有效减少最后一层的cost就能降低整体的cost. 

### 1. 空间放大
Monkey论文没有对空间放大进行建模, 空间放大建模比较容易. 我们知道, 1到L-1层包含了大约N/T个entry, L层包含N(T-1)/T个entry. 对于leveling来说, 最坏情况就是1到L-1层的entry全部被更新了, 那么L层就会包含N/T个无效的entry, 空间放大就是O(1/T), 如果T=10, 那么空间放大大约是10%. 对于tiering来说, 最坏情况是1到L-1层的entry全部更新, 所以L层的每个run都是无效数据, 最坏情况下空间放大就是O(T).

### 2. range lookup cost
dostoevsky对long range lookup进行了定义, 如果扫描的block数量大于2 * Lmax, 就认为是long, 否则就是short. range lookup是无法使用bloomfilter的, 所以可以认为, shot range lookup每个run都需要一次IO, 则leveling就是O(L), tiering就是O(L * T). 对于long range lookup来说, 开销和空间放大成正比了, 对于leveling来说就是O(s/B), tiering则是O(T * s / B).

### 3. Lazy Leveling

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/A9C0C2F9C6DF4A519A7D34EFB3D0D901?ynotemdtimestamp=1541669980860)

从上面的性能分析对比上来看, point lookup cost和long range lookup cost的大头来自于最后一层, 因为最后一层包含的entry数量最多. 而update cost来说, 每个level的cost是一样的, 所以dostoevsky的核心思想就是尽量优化1到L-1层的merge操作, 减少这些层的update cost, 理论上对于lookup cost的影响也非常小, 可以保持持平. 所以lazy leveling的做法就是1到L-1层采用tiering模式, L层采用leveling模式. 这样L层最多包括1个run, 1到L-1层每层最多包括T-1个run. 和monkey论文的方法类似, dostoevsky也同样对lazy leveling进行数学建模, 然后分析相应的开销, 分析结果如下图所示. 从图中可以看到, lazy leveling模式很好的在leveling和tiering中进行了性能折中. 需要注意一点的是, 每层的FPR设置方法和monkey类似, 也是等比变化的.

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/1C91C71A50044D109F1547F7B9D7B256?ynotemdtimestamp=1541669980860)

### 4. Fluid LSM-tree
有了lazy leveling思路之后, 将上述模式通用化, 就引出了Fluid LSM-tree模式. L层最多Z个runs, 其他层最多K个runs, 那么可以发现:
   - Z=1, K=1, 就是leveling模式
   - Z=T-1, K=T-1, 就是tiering模式
   - Z=1, K=T-1就是lazy leveling模式

![image](http://note.youdao.com/yws/public/resource/79570ffa03dcce127a39aaaa7419c180/3FC85105E95F47DEACBE696B992697A8?ynotemdtimestamp=1541669980860)

不过至于如何设置K和Z, 那就取决于workload了, 没什么更好的办法, 不停的实验和调节吧.
