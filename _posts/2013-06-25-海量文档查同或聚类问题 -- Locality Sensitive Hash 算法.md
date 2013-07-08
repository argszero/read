---
layout: default
title: 海量文档查同或聚类问题 -- Locality Sensitive Hash 算法
---
# 海量文档查同或聚类问题 -- Locality Sensitive Hash 算法

原文：http://blog.csdn.net/fxjtoday/article/details/6200257


考虑一下这个场景 , 使用网络爬虫高速爬取大量的网页内容 , 如果想把这些网页进行实时聚类 , 并从中提取每个网页聚类的主题 . 我们应该怎么样去做

对于普通或常见的聚类算法 , 比如 K-means, 或 Hierarchical 聚类 , 无法适用于这个常见 , 对于这些聚类算法无法进行 incremental 聚类 , 即在聚类开始前必须知道整个数据集 , 而这个场景中的数据集是随着爬虫不断增多的 . 而且这些聚类算法的 performance 不够高 , 比如对于 K-means 需要不断的 partition 以达到比较好的聚类效果 . 所以向来聚类算法在我的印象中是低效的 , 而面对这样一个需要实时数据递增处理的场景 , 我们需要一种 one-shot 的高效算法 , 接收到网页内容 , 迅速判断其类别 , 而不用后面不断地 revisit 或 recluster.

首先介绍下面这个聚类方法 
Leader-Follower Clustering (LFC) 
The algorithm can be described as follows:
If distance between input and the nearest cluster above threshold, then create new cluster for the input.
Or else, add input to the cluster and update cluster center. 
其实这个聚类方法再简单不过了 , 我是先想到这个方法 , 然后才发现这个方法有这么个看上去蛮牛比的名字 . 
有了这个方法 , 当新网页来的时候 , 和所有老的网页形成的聚类算下相似度 , 相似就归到这类 , 不相似就创建新类

这个过程当中有个经典问题 , KNN 问题 (K-Nearest Neighbor) . 
面 对海量数据 , 而且是高维数据 ( 对于文本 feature 一般是选取文本中的 keywords, 文本中的 keywords 一般是很多的 ), KNN 问题很难达到线性 search 的 , 即一般是比较低效的 . 这样也没办法达到我们的要求 , 我们需要新的方法来解决这个 KNN 问题

当然该牛人出场了 , 他提出了一种算法 
Locality Sensitive Hash(LSH) 
这个算法的效果是 , 你可以把高维向量 hash 成一串 n-bit 的数字 , 当两个向量 cosin 夹角越小的时候 ( 即他们越相似 ), 那么他们 hash 成的这两串数字就越相近 . 
比较常用的 LSH 算法是下面这个 
Charikar's simhash 
Moses S. Charikar. 2002. Similarity estimation techniques from rounding algorithms. In STOC ’02: Proceedings of the thiry-fourth annual ACM symposium on Theory of computing, pages 380–388, New York, NY, USA. ACM.

用 LSH 算法怎么样来解决高维数据的 KNN 问题了 , 我们可以参考 Google 在 WWW2007 发表的一篇论文 “Detecting near-duplicates for web crawling”, 这篇文章中是要找到 duplicate 的网页 , 和我们的问题其实是同一个问题 , 都是怎样使用 LSH 解决 KNN 问题

分两步 , 
第一步 , 象我们上面说的那样 , 将文档这样的高维数据通过 Charikar's simhash 算法转化为一串比特位 . 对于 Google 的问题 , 
We experimentally validate that for a repository of 8 billion webpages, 64-bit simhash fingerprints and k = 3 are reasonable. 
就是对于 80 亿的文档 , 我们把每个文档转化为 64-bit 的 simhash fingerprints, 当两个 fingerprints 有 k = 3 位不同时 , 我们就认为这两个文档不相同 .

Charikar's simhash is a dimensionality reduction technique . It maps high-dimensional vectors to small-sized fingerprints. 
其实 LSH 算法的基本原理就是 , 把一个多维空间上的点投影到一个平面上 , 当多维空间中的两个点在平面上的投影之间距离很近的时候 , 我们可以认为这两个在多维空间中的点之间的实际距离也很近 . 但是 , 你想象一下 , 你把一个三维球体中的两个点投影到一个随机平面上 , 当投影很靠近的时候 , 其实那两个点不一定很靠近 , 也有可能离的很远 . 所以这儿可以把两个点投影到多个随机平面上 , 如果在多个随机平面上的投影都很靠近的话 , 我们就可以说这两个多维空间点之间实际距离很近的概率很大 . 这样就可以达到降维 , 大大的减少了计算量 .

算法过程如下 , 其实挺好理解的 
Computation: 
Given a set of features extracted from a document and their corresponding weights, we use simhash to generate an f-bit fingerprint as follows. 
We maintain an f-dimensional vector V, each of whose dimensions is initialized to zero. 
A feature is hashed into an f-bit hash value. 
These f bits (unique to the feature) increment/decrement the f components of the vector by the weight of that feature as follows: 
ü   if the i-th bit of the hash value is 1, the i-th component of V is incremented by the weight of that feature; 
ü   if the i-th bit of the hash value is 0, the i-th component of V is decremented by the weight of that feature. 
When all features have been processed, some components of V are positive while others are negative. The signs of components determine the corresponding bits of the final fingerprint. 
For our system, we used the original C++ implementation of simhash, done by Moses Charikar himself.

第二步 , HAMMING DISTANCE PROBLEM

第一步把所有文档都变成 64-bit 的 fingerprints, 那么面对几十亿的 fingerprints, 怎么样能快速找到和目标 fingerprint 相差 k 位的所有 fingerprint 了 . 
其实这就是个对于 hamming distance 的 KNN 问题 , 
Definition: Given a collection of f-bit fingerprints and a query fingerprint F, identify whether an existing fingerprint differs from F in at most k bits.

汉明距离 (hamming distance) 
在信息论中，两个等长字符串之间的汉明距离是两个字符串对应位置的不同字符的个数。换句话说，它就是将一个字符串变换成另外一个字符串所需要替换的字符个数 
可见 , 对于 hamming 距离 , 不是简单的通过排序索引就可以解决的

说两个简单的方法 , 虽然不可行 , 但也是一种思路 
耗费时间的方法 
Build a sorted table of all existing fingerprints 
对于给定的 F, 找出所有 Hamming distance from F is at most k 的 fingerprint 然后去 table 里面搜索 , 看有没有 
For 64-bit \_ngerprints and k = 3, we need C64 3 = 41664 probes. 这样查找时间太长了 .

耗费空间的方法 
还有个办法就是空间换时间 , 对现有的每个 fingerprints, 先事先算出所有和它 Hamming distance 小于 3 的情况 , 但这种方法预先计算量也太大了 , 如果现有 n 个 fingerprint, 就需要算 41664\*n. 
可见用传统的方法是很难高效的解决这个问题的 .

那么怎么办 , 有什么办法能够在海量的 F bit 的向量中 , 迅速找到和查询向量 F ′ 只差 k bit 的向量集合了 
We now develop a practical algorithm that lies in between the two approaches outlined above: it is possible to solve the problem with a small number of probes and by duplicating the table of fingerprints by a small factor. 
我们需要一种介于上面两种比较极端的情况的方法 , 耗些时间 , 也耗些空间 , 但都不要太多 ......

    设想一下对于 F bit, 可以表示 2F 个数值 , 如果这儿我们完全随机产生 2d 个 F bit 的数 , 当 d << F 时 , 这些随机数值的高 d 位重复的应该不多 , 为什么 , 这些数值是完全随机产生的 , 所以应该相对均匀的分布在 2F 大小的空间里 , 如果完全平均生成 2d 个数 , 那么每个数的高 d 位都是不同 . 但是这儿是随机产生 , 所以会有些数的高 d 位是相同的 , 不过数量不会多 . 所以这边就可以把高 d 位作为计数器 , 或索引 . 这个假设是这个方法的核心 , 有了这个假设 , 不难想到下面怎么做 ...

首先对现有的所有 fingerprints 进行排序 , 生成有序的 fingerprints 表 
选择一个 d ′, 使得 |d ′-d| 的值很小 ( 就是说你选择的这个 d’ 和 d 只要差的不多 , 都可以 ), 因为表是有序的 , 一次检测就能够找出所有和 F ′ 在最高的 d ′ 位相同的指纹 , 因为 |d ′-d| 的值很小 , 所有符合要求的指纹数目也比较小 , 对于其中的每一个符合要求的指纹 , 我们可以轻易的判断出它是否和 F 最多有 K 位不同 ( 这些不同很自然的限定在低 f-d ′ 位 ) 。

上面介绍的方法帮我们定位和 F 有 K 位不同的指纹 , 不过不同的位被限定在低 f-d ′ 位中。这对大部分情况来说是合适的 , 但你不能保证没有 k 位不同出现在高 d 位的情况 . 为了覆盖所有的情况 , 采用的方法就是使用一种排序算法 π, 把当前的 F bit 随机打乱 , 这样做的目的是使当前的高位 bit, 在打乱后出现在低位 bit, 然后我们再对打乱后的表排序 , 并把 F ′ 用相同的排序算法 π 打乱 
再重复我们上面的过程 , 来查找低 f-d ′ 位上 k 位不同的情况 
这样当我们多使用几种排序算法 π, 重复多次上面的过程 , 那么漏掉 ’k 位不同出现在高 d 位 ’ 的情况的概率就会相当的小 , 从而达到覆盖到所有情况

还有个问题 , 这儿的假设是 , 2d 个数是随机产生的 . 那么我们这儿的 fingerprints 是基于 hash 算法产生的 , 本身具有很大的随机性 , 所以是符合这个假设的 . 这点原文 4.2 Distribution of Fingerprints 有相应的实验数据 .

假设 f=64,k=3, 那么近似网页的指纹最多有 3 位不同。假设我们有 8B=234 的已有指纹 , 即 d=34 。 
我们可以生成 20 个有序排列表 ( 即使用 20 种不同的排列算法打乱原 fingerprint, 并生成有序表 ), 方法如下 ,  
把 64 位分成 6 块 , 分别是 11,11,11,11,10 和 10 位。共有 C(6,3)=20 种方法从 6 块中选择 3 块。对于每种选择 , 排列 π 使得选出的块中的位成为最高位 . d ′ 的值就是选出的块中的位数的总和。因此 d ′=31,32, 或者 33 ( 和 d 差的不多 ). 平均每次检测返回最多 234~31 个排列后的指纹。实际应该不会很多

你也可以用 16 个表 , 或更少 , 但使用的表越少 , 必须 d 的取值也越少 , 这样最后需要验证的 fingerprint 就越多 , 这儿就有个时空的平衡 , 时间和空间不可兼得 .

说到这儿大家是不是已经被这个复杂的方法给搞晕 , Google 的这个方法是为了在几十亿篇文章中发现相同的文章 , 相对的精确性要求比较的高 , 如果为了我们的初衷 , 进行文本聚类的话 , 我们不需要用 64-bit 来进行 hash, 也许可以用 16bit, 这个可以通过实验来选择 , 为了避免复杂的汉明距离问题 , 只当两个文章的 fingerprint 完全一致时才认为他们属于一类 , 随着用更少的位数来进行 hash, 这个应该是可行的 , 不过需要具体的实验证明

