# 从Anna看Berkeley RISELab在无协调分布式系统的研究历程

前些日子Berkeley RISE实验室发布的Anna论文引起了业界的热烈讨论, 其卓越的性能让大家惊叹不已. 不过论文完全读不懂, 原来Anna是RISE实验室近10年在无协调分布式系统领域研究成果的集大成者, 很多背景论文在国内讨论的并不多, 而且中文资料也非常少, 所以不得不顺藤摸瓜粗读了十来篇相关论文, 整理了一篇关于无协调分布式一致性研究成果的文章. 有些内容我也理解不深刻, 如有错误, 欢迎同行批评指正.

## 适合分布式系统的声明式编程语言
### Prolog和Datalog
Prolog是一种声明式逻辑编程语言, 是Prolog的子集. Prolog是一门历史非常悠久的语言了, 在SQL流行之前, Prolog和SQL是并列的地位, 后来SQL逐步取得了垄断性地位. 因为Datalog是Prolog的子集, 所以Prolog更不容易理解一些. 咱们先看容易理解的Datalog, 我们看一个维基百科的例子.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/3A8C6E60DD4E4B8987A298197218D82D?ynotemdtimestamp=1539347537645)

上面两行代码说明了两个事实, bill是mary的父母, mary是john的父母.

![iamge](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/9BB1DE3447E945A7B4194B34686D6EC2?ynotemdtimestamp=1539347537645)

上面两行定义了两个规则:
   - 如果X是Y的父母, 则X是Y的祖先
   - 如果X是Z的父母, Z是Y的祖先, 则X是Y的祖先
其中:-符号表示蕴含, 也就是说:-符号右边如果都为真, 那么:-左边就为真. 那这个东西怎么和数据库挂上边的呢? 我们看下面的查询语句:

![iamge](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/576531B418D54A24899BB7A4ABB87E4D?ynotemdtimestamp=1539347537645)

这行代码表示返回所有的X, 满足bill是X祖先的条件, 其实就是返回bill所有的子孙, 这样其实就表达了一种查询条件了. 事实上, 支持Datalog的数据库还不少, 大家可以看维基百科中的列表. 

Datalog的语法就是有一组类似上述的语句构成的, 其中:-叫做蕴含符号, 蕴含符号左边叫做规则头, 蕴含符号右边叫做规则体. 规则体由若干个子句组成, 各个子句之间是and关系. 子句又成为一阶谓词逻辑, 由谓词(其实就是关系), 个体和量词组成. 比如parent(X, Y)中parent是量词, X和Y是个体. 以大写字母开头的标识符叫做变量, 以小写字母开头的标识符叫做常量. 如果一个子句不包括蕴含符号, 那么规则就变成了事实. 比如parent(bill, mary)就说明了bill是mary父母的事实. 

那么这个与数据库查询有什么关系呢? 我们把上述场景替换成数据库的语义看看效果. parent这个谓词看做数据库中一张名字叫parent的表, (bill, mary)和(mary, john)看做两个tuple, 并且是parent表中的两行, 那么每个规则就可以表示一个查询条件了, 其中逗号用来表达join. 在Designing Data-Intensive Application一书的第二章, 讲述了Datalog的基本概念和例子, 可以帮助大家理解.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/5420ABEB4A594323B387BCC76592CD98?ynotemdtimestamp=1539347537645)

上面这幅图显示了datalog和SQL的转换关系, 这样对比着看就更容易理解了.

### Overlog
Overlog语言是Datalog的扩展, 最初设计出来是为了简化基于overlay网络的编程, 文章(Implementing Declarative Overlays)发表于2005年. Overlog不是一个纯粹的声明式语言, 为了方便表达消息的存储和传递, 在Datalog的基础上, 引入了新的语法元素. Overlog引入了位置表达式的概念, 用@X表示节点X, 从而就可以表达网络交互了, 如下图的语句就表示从A节点移动payload到B节点:

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/0C091B05EED54158BA468FA07E7CAE45?ynotemdtimestamp=1539347537645)

铺垫了这么晦涩的背景知识, 到底有什么用呢? 和分布式系统有啥关系呢? 在论文I Do Declare: Consensus in a Logic Language中, 作者给出了基于Overlog语言编写的2PC过程和Paxos过程, 终于和分布式系统沾上边了. 下图就是论文里说明的2PC协议的核心代码, 大家可以体会下是不是比用C++/Java这类编程语言实现要简单的多呢. 完整的paxos实现在论文中都有详细的论述, 大家感兴趣可以读论文, 我觉得我们理解了使用声明式语言来实现分布式系统编程这一核心思想就达到目的了, 毕竟这样小众的语言离我们日常工作还是有点遥远, 太学术化了.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/E4E8AF0F3C924D528FEF779AEC6BA3A7?ynotemdtimestamp=1539347537645)

### BOOM项目
BOOM是Berkeley Orders Of Magnitude的缩写, BOOM代表一系列研究项目, 目标是利用声明式编程语言来简化分布式系统的开发. 在BOOM Analytics: Exploring Data-Centric, Declarative Programming for the Cloud论文中, 作者使用Overlog构建了一个接口兼容的HDFS系统, 性能和原生的HDFS可比, 然后在高可用, 可扩展等方面, 进行了扩展, 从而证明BOOM项目的价值.

## Bloom编程语言和dedelus

Overlog虽然可以表达消息存储和传递, 但是无法表达时间和空间概念, 为此Dedalus: Datalog in Time and Space里提出了Overlog的加强版dedalus. 下面这条语句就是dedalus的典型.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/AF4B0F69D7BC40D890213794AB0609A3?ynotemdtimestamp=1539347537645)

BOOM项目的成功继续激励Berkeley的研究人员向声明式编程语言方向前进. 虽然BOOM可以实现复杂程度如HDFS的分布式系统, 但是没有考虑分布式一致性的问题. 我们都知道, 在一个大型分布式系统中, 因为网络消息乱序或者网络延迟等问题, 完美的数据一致性是非常昂贵的, 最终一致性往往就可以满足应用场景的需求了. 那怎么保证程序是最终一致性呢? 于是研究人员提出了CALM: consistency as logical monotonicity原则, 简单点说, 就是如果程序满足单调性, 那么就不需要任何协调机制来实现最终一致性了, 对于非单调程序段, 在协调机制保护条件下, 也仍然能实现最终一致性了. 于是在这个思路下, 研究人员设计了Bloom声明式语言, 并且可以自动检测非单调的代码段, 提示开发人员引入协调机制.

那么什么是单调程序呢? 简单理解就是增加程序的input只能增加程序的output, 程序最终的输出不会因为输入乱序而改变, 比如SQL里的查询语句. 如果是聚合操作或者否定操作, 那么就不满足单调的特性了, 必须等待所有输入按照逻辑顺序接收到之后, 才能计算并输出结果. 单调程序是非常容易分布式化的, 他们可以通过流式算法来实现, 并且可以容忍消息乱序和delay. 相反, 对于非单调程序, 就拿最简单的累加操作来说, 则必须等待所有input接收到之后才能输出结果. 而等待的学术化解释其实就是协调, 协调可以包括序列号, 计数器, paxos算法等. 在分布式系统设计中, 如果我们能控制需要协调的代码段最小, 那往往就意味着高性能了, 这就是Bloom出现的驱动力了.

Bloom和Overlog相比是一个纯粹的声明式编程语言, 在dedalus的基础上设计出来, 并且论文中给出了基于Ruby的原型实现Bud(Bloom Under Development). 有了datalog和overlog的基础, 理解Bloom也就不那么费劲了. 下图给出了Bloom的核心元素. 更详细的语法说明大家还是看论文吧, 我们重点还是说原理.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/C9922A3B43954CDDB75F81B355F57825?ynotemdtimestamp=1539347537645)

那么这和分布式存储好像还是没有扯上关系, 下面重点来了:

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/1306D5F58EAF458CB68405090E100254?ynotemdtimestamp=1539347537645)
![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/1306D5F58EAF458CB68405090E100254?ynotemdtimestamp=1539347537645)

这幅图里使用bloom定义了通用KV存储系统的协议或者说接口, 那么基于这个接口, 就可以实现更复杂的KV存储系统了. 论文里给出了单节点KV存储系统和多节点KV存储系统的代码. 下图是单节点KV存储系统的实现代码:

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/4377A549CE504779BA687BFD2C9FCD8F?ynotemdtimestamp=1539347537645)

通过bloom的分析软件我们进一步发现, 代码的9-11行打破了单调条件, 以上就是bloom的核心思想和应用场景. 论文中还分析了一个更复杂的购物车的例子, 并且bloom还提供了可视化功能, 方便可视化单调性分析过程. 经过这么多背景知识的学习, 总算是和分布式存储系统沾边了. 后来研究人员还设计了BloomUnit测试框架, 帮助对基于bloom实现的分布式系统进行测试, 让bloom可以更接地气了.

## Coordination Avoidance in Database Systems
如果我们把数据库看做一个整体, 数据库由多个版本的状态组成, 数据库状态的改变只能由写事务触发, 那么在保证应用层语义正确性(论文里叫invariants, 也可以理解为ACID的C)的前提下, 如果两个事务可以并行执行, 那么就认为这两个事务无须协调, 否则就认为两个事务需要协调. 协调的方法可以包括锁, 2PC, 阻塞等待等等方法. 不难想象, 如果两个事务需要协调, 那么性能必然受影响, 所以在保证invariants的情况下, 尽可能减少协调, 就意味着可以提升性能.

为此将上述过程建立数学模型. 我们将数据库看做一系列状态的改变(有点类似状态机), 事务运行在一些相互无关的状态快照上. 当写事务提交时, 写事务更改状态快照, 这些状态快照merge之后, 形成最新的状态. 论文 Coordination Avoidance in Database Systems 就根据上述模型, 给出了最优化的方法.

## Consistency Analysis in Bloom: a CALM and Collected Approach
我们知道协调往往是导致分布式数据库性能下降的原因, 而分布式数据库之所以需要协调本质上是为了满足一致性要求. Highly Available Transactions: Virtues and Limitations论文里对数据一致性和事务一致性进行了严格的定义和分类, 同时指出很多应用其实对严格的一致性并没有那么强的需求. 这就给我们提供了优化的空间了. Consistency Analysis in Bloom: a CALM and Collected Approach 这篇论文给出了一致性和逻辑单调性之间的关系, 只要代码满足逻辑单调性, 那么就一定能够满足最终一致性. 所以基于Bloom形式化语言, 并且开发了分析程序, 通过检测违反单调性的代码来指出违反最终一致性的地方, 指导开发人员增加协调机制的代码来保证一致性.

## Conflict-free Replicated Data Types(CRDT)
为了解决一致性和协调性之间的矛盾, CRDT采用了和CALM看上去出发点不同, 但是最终却非常类似的思路, 所以我们先说说CRDT. CRDT首先对数据复制过程进行建模, 论文里给出了两种建模方法, 并且证明两种方法的效果一致.这里简单介绍其中的一种, 叫做State-based object. 每个object表示成一个元组(S, s0, q, u, m). S代表object的状态集合, 进程pi中的数据副本状态, 记作si, 并且si属于集合S. object的初始状态记作s0. q, u, m分别代表3个函数, q代表query, u代表update, m代表merge. q和u比较好理解, 当数据副本接收到其他副本发过来的更新请求时, 调用m函数. 进程之间可以保证所有更新请求, 最终都会发送给所有副本. 在某些副本的操作序列会被编号, 从1开始, 编号逐步递增. 如果对一个object的所有查询请求都返回相同的结果, 则说明两个状态等价. 如果数据副本都按照相同的顺序接收请求并update本地副本, 那么所有副本一定是最终一致的. 但是这就要求请求的复制必须有序, 不得不引入协调机制来实现. 如果每个副本都可以提供读写请求(减少协调机制的引入), 并且最终所有副本都能够达成一致(假设所有消息都可以发送给所有副本, 允许乱序和不定期的时延), 那就是最理想的效果了.

能满足上述理想条件的数据结构或者说数据对象, 就叫做CRDT. 那么有没有这样的对象呢? 最直观的例子就是计数器, 显然满足上述理想效果. 论文中进行了大量的数学证明, 如果merge和update的组合能够满足交换律, 结合律和幂等率, 则满足CRDT的条件. 如果update无法满足三率, 则可以考虑让update附带元信息, 比如每次update操作都带有时间戳, 在merge时对本地副本的时间戳和远程副本的时间戳进行比较, 取最新结果, 就可以满足CRDT条件. 特殊情况, 如果update就可以满足三率, 那么merge只需要回放update即可. 让元信息满足条件的方式是update操作满足单调性, 也称为偏序关系. 论文里给出了一个相对通用的满足三律的merge函数, 叫做最小上界(Lease Upper Bound, 简称LUB). 给定一个偏序集合, 如果对于任意两个元素都存在LUB, 那么数学里叫做semilattice, 如果既存在最小上界, 也存在最大下届, 那么就叫做lattice. 显然通过LUB定义的偏序集合自然就是一个semilattice了. 论文中给出了满足CRDT的例子, 包括向量, 集合, 图等. 目前采用CRDT的系统主要是Riak KV. CRDT的论文公式比较多, 理解起来比较费劲, [CRDT——解决最终一致问题的利器](https://yq.aliyun.com/articles/635629) 这篇文章更通俗易懂些. 另外需要说明的是, CRDT不仅仅适用于多个进程, 也适用于多线程, 可以更充分的利用多核.

## Logic and Lattices for Distributed Programming
CRDT比较好的满足了最终一致性并且无协调性, 但是设计一个复杂的满足CRDT的数据结构还是比较复杂的(特别是有删除操作的时候或者需要操作多个数据项的时候), 并且不容易证明和测试其正确性. 所以在CALM和Bloom的基础上, 设计了BloomL, 对Bloom进行了扩展, 除了bloom中的集合类型之外(集合类型因为只增加不减少, 所以天然满足单调性, 这就是为什么bloom只定义了集合类型的原因), 定义了叫做lattice的类型, 并且除了内嵌的merge函数之外, 还可以支持自定义的merge函数. 因为lattice存在merge函数, 所以只要merge函数满足三率, 那么就可以像集合类型一样, 仍然满足单调性. 有了BloomL对lattice的支持, 论文中基于BloomL和vector clock构建了一个通用的分布式key-value存储系统, 并且满足最终一致性, 代码非常简洁, 足以证明BloomL的强大威力.

## Anna: A KVS For Any Scale
介绍了这么多铺垫内容, 我们终于可以开始看看Anna的设计思想了. 分布式系统的关键在于可扩展, 不仅仅是机器可扩展, 随着cpu核数的不断增加, 也包括在多核之间可扩展. 所以Anna的核心设计目标就是可以扩展到任意规模: 不仅仅在单机多核上性能出众, 也可以扩展到大规模集群上. 同时考虑到不同应用对一致性的需求不同, Anna还可以支持多种一致性级别.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/14A35EB86FCE4F548A0065EE8330A69A?ynotemdtimestamp=1539347537645)

上图是Anna的系统架构, 每个线程都有自己的私有内存来存储数据副本, 线程数不超过CPU核数, 数据副本之间通过消息队列异步的同步消息. 数据的分布采用一致性hash技术, 由client proxy维护数据以及数据副本的分布. 数据的副本数可以最小到key的粒度, 这样可以非常容易应对局部热点情况. 为了保证最终一致性, 每个线程私有的内存数据结构满足lattice的性质即可. 从架构上来看, Anna坚决的避免了由于共享内存而引入的额外协调机制, 同时因为每个线程都是私有内存, 所以对CPU cache的利用也更友好. 还有一点好处是, 这种架构让Anna可以非常容易的扩展到多机架构上, 每个线程的处理逻辑可以说完全相同. 当然为了减少同一台机器情况下的消息传递开销, 在同一台机器上传递消息的时候并没有传递消息体, 而是把消息体放在共享内存里, 只传递消息在共享内存的索引. 论文里给出了2个典型的例子来介绍如何通过基本的内嵌lattice组合来实现不同的一致性级别, 一种方法是基于vector clock机制组合基础的lattice结构, 另一种方法是利用client proxy的内存缓存来解决不同一致性级别的数据可见性问题. 下图就是用lattice组合和vector clock机制来实现因果一致性的方法. 其他一致性级别的实现方法可以阅读原文.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/AC1A44E140734E2DBC76C38EFCE4E7C1?ynotemdtimestamp=1539347537645)

论文中和一些优秀的KV程序库以及Redis/Casandra进行了性能对比, 给出的实验数据也非常有冲击力, 数据可以参见下面的图, 详细测试数据的分析就不翻译了, 大家还是阅读原文吧.

![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/484209AFF3C54F03B08F14ECEFE7DEE4?ynotemdtimestamp=1539347537645)
![image](http://note.youdao.com/yws/public/resource/e772e5a10c18b98b33880d107b8b89e4/4ACEAEAD92CC48508485D925AFAC51B8?ynotemdtimestamp=1539347537645)

## 总结
经过数十篇论文之后, 终于基本理解了Anna的核心设计思想了. Anna利用具备lattice属性的数据结构来构建KVS的思想真得让我脑洞大开, 而且仅仅用2000行C++代码就完成了核心功能的实现, 让我们再次感受到了成熟理论的威力. 在工程方面, Anna没什么特别让人想不到的技术点, 架构也非常简单并且容易理解. 不过距离生成环境可用还需要大量的工程工作, 让我们期待Berkeley RISELab能否像孵化Spark一样, 产出另一个工业级产品吧.

## 参考文献
   - https://en.wikipedia.org/wiki/Datalog
   - Implementing Declarative Overlays: 论文中定义了Overlog语言, 并且基于overlog实现了overlay网络编程
   - Dedalus: Datalog in Time and Space: 提出了dedalus.
   - The Declarative Imperative: 针对dedalus进一步加强, 丰富了语言的表达能力
   - BOOM Analytics: Exploring Data-Centric, Declarative Programming for the Cloud
   - Consistency Analysis in Bloom: a CALM and Collected Approach: 明确了一致性和单调性的关系(CALM原则), 设计了bloom声明式编程语言, 可以分析不满足CALM原则的代码
   - BloomUnit: Declarative Testing for Distributed Programs: 针对bloom语言的测试框架
   - Coordination Avoidance in Database Systems: 介绍了分布式数据库的协调性的概念和优化方法
   - Highly Available Transactions: Virtues and Limitations: 总结了哪些隔离级别可以实现HAT以及实现HAT的方法
   - Conflict-free Replicated Data Types(CRDT): 介绍了CRDT的理论和实践
   - A comprehensive study of Convergent and Commutative Replicated Data Types: CRDT的techreport
   - Logic and Lattices for Distributed Programming: 介绍了格理论和偏序模型
