# 读后感 SuRF: Practical Range Query Filtering with Fast Succinct Tries

SuRF这篇论文是2018年SIGMOD唯一一篇best paper, 论文的核心思想是实现了一种叫做FST(Fast Succinct Trie)的数据结构, 既可以享受Succinct数据结构的高压缩特性, 还可以实现快速的point查询和range查询. FST本质上是一种高度优化之后的Trie树, 其实可以实现静态词典的数据结构. 论文中使用FST替换掉了rocksdb的bloomfilter, 在相同存储空间的情况下获得了查询性能的提升.

## Trie树
![image](http://note.youdao.com/yws/res/214/BE0B03F853D546FEB6C53356F1BD0C96)

上图是维基百科中介绍的Trie树的例子. Trie树又称前缀树或者字典树, 是一种可以保存静态kv数据的数据结构. Trie树包括以下几个特点:
   1. 一个节点的所有子孙节点具有相同的前缀
   2. 从根节点到叶子节点可以唯一表示一个健
   3. 可以实现基于前缀的模糊查询
   4. 根节点对应空字符串

## Level-Ordered Unary Degree Sequence(LOUDS)

![image](http://note.youdao.com/yws/res/227/15C129E9C4B44FFEA41C9715F3BC09E6)

对于一个树来说, 基于succinct的思路可以让树的存储空间接近信息论的下界. 上图将一个树的每个节点进行编码, 节点的编号按照层数生成. 编码规则就是对于一个节点来说, 将孩子节点标记为1, 最后标记为0. 比如对于节点3来说, 其编码就是1110. 按照节点编号的顺序, 生成一个bit序列从而完成整个树结构的编码(不包含value).

为了能够访问这棵树, 给定一个bit序列(起始位置是0), 定义四个基本操作:
* rank1(i): 返回[0, i)位置区间内, 1的个数
* rank0(i): 返回[0, i)位置区间内, 0的个数
* select1(i): 返回第i个1的位置(整个bit序列)
* select0(i): 返回第i个0的位置(整个bit序列)

为了计算方便, 在root节点之上, 增加一个新的root节点, 然后基于下面三个公式来访问整个树:

* first-child(i) = select0(rank1(i)) + 1
* parent(i) = select1(rank0(i))
* next-sibling(i) = i + 1

其中first-child(i), parent(i), next-sibling(i)都表示位置为i的节点对应的第一个子节点, 父节点和兄弟节点的位置. 大家可以使用上述公式计算下图中描述的树结构是否正确. 关于succinct tree的编码方式, 论文里写的比较简单, 论文的引文34给出了更详细的论述, 这篇论文是Jacobson在1989年发表的, 更详细的内容大家还是查阅论文, 里面有更多关于子节点的操作方法.

## Fast Succinct Trie
基于LOUDS编码方式, FST对LOUDS进行了进一步压缩, 下图介绍了基本的压缩方法:

![image](http://note.youdao.com/yws/res/289/5F2F376364BD4898A2A74DCFC322F0DB)

FST将LOUDS分成了两层, 上层节点数量少, 使用LOUDS-Dense编码方式, 下层节点数多, 使用LOUDS-Sparse编码方式. 

1.    LOUDS-Dense

我们先来看看LOUDS-Dense的编码方式. 假设每个节点最多有256个子节点, 那么在LOUDS-Dense编码方式中, 每个节点使用3个256个bit的bitmap来保存信息. 这3个bitmap分别是:
* D-Labels: 将子节点的label变化置位
* D-HasChild: 标记对应的子节点是否是叶子节点还是中间节点
* D-IsPrefixKey: 标记当前前缀是否是有效的key
我们仍然可以使用select&rank操作来访问对应的tree节点.

2.    LOUDS-Sparse

LOUDS-Sparse使用3个bit序列来对trie树进行编码, 在整个bit序列中, 每个节点的长度相同, 这三个bit序列分别是:
* S-Labels: 记录每个节点的label编号, key节点用0xFF标记, 按照树的层数按顺序记录(如果最多有256个子节点, 则每个节点占用4个byte)
* S-HasChild: 记录每个节点是否含有子节点, 有的话标记为1, 每个节点使用一个bit
* S-LOUDS: 记录每个节点是否是第一个节点, 每个节点使用一个bit
仍然可以使用rank&select操作来访问整个trie树.

trie树经过LOUDS-DS编码之后, 可以高效支持下面3个操作:
* ExtractKeySearch(key): 如果key存在, 返回value
* LowerBound(key): 返回一个迭代器, 迭代器指向第一个大于等于key的位置
* MoveToNext(iter): 移动迭代器指向下一个key-value
* 

3.    LOUDS-DS的空间复杂度分析

给定一个含有n个节点的trie树, S-labes需要使用8n个bits, S-HasChild和S-LOUDS一共使用2n个bits, 所以LOUDS-Sparse使用10n个bits. LOUDS-Dense的空间与Sparse和Dense的分界线有关, 通常情况下, Dense占用的空间要远远小于Sparse部分. 这样整个LOUDS-DS编码的Trie树接近10n个bits, 理论证明最少的编码数量大约是9.44n个bits, 接近理论的下限了.

## Succinct Range Filters

虽然FST已经尽可能的使用最少的存储空间了, 但是我们仍然希望减少存储空间的占用, 进而让整个索引全部放在内存里, 为此引入了4种不同的Trie树的裁剪方式.

![image](http://note.youdao.com/yws/res/403/3C7E9E50369849D3A73610E2133C6B9F)

1.    Basic SuRF

FST是一个完整的索引结构, 可以存储全部的索引数据, 这种情况下是100%精确的. Basic SuRF的思想就是只存储key的前缀, 实际上就是砍掉树的部分叶子节点. 我们使用FPR(false positive rate)来衡量效果, 具体的FPR与key的分布有关, 论文中给出了Basic SuRF的FPR的上限.

2.    SuRF with Hashed Key Suffixes

为了降低FPR, 在Basic SuRF的基础上, 对key进行hash计算之后, 将hash值的n个bits存储到value中, 查询的时候还原回来完整的key. 这种方法可以降低FPR, 论文中有计算公式, 但是这种方法对range query没什么帮助.

3.    SuRF with Real Key Suffixes

和SuRF with Hashed Key Suffixes不同, SuRF-Real存储n个bits的真实key, 这样point查询和range查询都可以获益, 但是在point查询下, FPR比SuRF-Hash要高.

4.    SuRF with Mixed Key Suffixes

为了享受Hash和Real两种方式的优点, Mix模式就是将两种方式混合使用, 混合的比例可以根据数据分布进行调节来获得最好的效果.

## 性能测试

![image](http://note.youdao.com/yws/res/472/1E68760A7EBE40B0A198F2547F831121)

论文中使用了两组key的数据进行性能对比测试. 一组是由YCSB输出的64bit的整数, 另一组是由字符串组成的电子邮件地址, 其中整数的key有50M个, 电子邮件地址组成的key有25M个. 然后使用FST分别和B+tree, ART(Adaptive Radix Tree), C-ART进行比较, 因为latency和memory实际上是两个trade-off, 所以上面的对比图中定一个了一个关于latency和memory的代价函数, 图中对比的是代价函数.

![image](http://note.youdao.com/yws/res/476/6DC019FD8F9644139A713A3ACE48F583)

第二组实验对FST和其他几种succinct数据结构进行了对比, 可以看出来无论是memory使用还是latency都是最优的.

![image](http://note.youdao.com/yws/res/481/F4B663C0E19846A28EF389B6F7E1F147)

这幅图对比了SuRF不同模式和bloomfilter的FPR对比, 一般情况下, 在pointquery下, SuRF比bloomfilter还是要差一些. 对于email这组测试数据, range query的FPR比较高(20%~30%之间了).

![image](http://note.youdao.com/yws/res/488/56DD3D8B5EED476E9D659C0511D38832)

这幅图对比了SuRF和bloomfilter的吞吐, 吞吐实际上指的是查询速度, 大家可以从这里大概评估出SuRF的吞吐数量级.

## 应用场景
试想如果我们把rocksdb的所有key都复制一份存储在SuRF中的话(不存储value), 那么SuRF起的作用不就和bloomfilter一样了么, 同时还可以支持range query了. 为此论文将SuRF应用在了Rocsdb中, 替换了bloomfilter, 并且进行了对比测试(占用的空间和bloomfiler相同). 测试程序运行在普通的SSD上, 下图是性能对比数据:

![image](http://note.youdao.com/yws/res/505/BD0D0D24ED964972821A72E3AD0AA599)

从性能数据上看, 对于point query, SuRF的效果比bloomfilter相比还是差一些, 但是在range query下, 效果比bloomfilter要好很多了, IO减少的次数还是非常明显的.

## 代码
SuRF的代码和rocksdb的集成代码已经在github上[开源](https://github.com/efficient/SuRF), 大家可以进一步了解代码. 作者代码封装的也比较工整, 读起来也比较顺畅.

## 总结
为了便于理解SuRF, 作者设计了一个[demo website](https://www.rangefilter.io/), 配合demo会更容易理解. 读完这篇论文之后, 最大的感受是之前的数据结构白学了! 在1989年就提出的LOUDS编码方法, 竟然完全不知道, 事实上, LOUDS已经在MIT的高级数据结构课程里了([youtube上有公开课视频](https://www.youtube.com/watch?reload=9&v=3Y2weLDiUWw/)). 作者在LOUDS的基础上设计了FST, 并且进行了相应的工程优化最终形成了SuRF, 无论是思路上还是效果上都非常出众, 能获得best paper还是很有道理的. 论文中的测试数据表明, 在和bloomfilter存储空间相同的条件下, point query的性能还是有所下降, 不过bloom filter本身占用的空间不大, 在我们的生产环境中, bloomfilter都是常驻内存的, 所以我觉得可以适当提升SuRF的空间占用来弥补point query的性能下降.
