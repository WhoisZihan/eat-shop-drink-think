---
title: "CS224W Lecture 4 - Community Structure in Networks"
date: 2020-03-14T19:55:25+08:00
tags: [graph, cs224w]
categories: ["stanford courses"]
draft: false
---

> 课程主页是[http://web.stanford.edu/class/cs224w/][1], 视频在b站可以找到[CS224W 2019 Fall][2], youtube也有

> 今天看了cs224w的lecture 4, 讲的是*Community Structure in Networks*，大致就是如何检测图中的community、overlapping community，以及评价方法

# 1. What is Communities

上个lecture中讲了如何检测网络中各个节点的角色(role), 这个lecture讲的就是如何检测每个节点属于哪个团体(community)。顾名思义, 就是把网络分为不同的部分, 每个部分内部的节点可以认为具有更高的相似性, 一个典型就是社交网络, 在同一个专业里面的人所学的课程都是类似的，同一个班级里的人比不同班级的人之间更容易互相认识。

当然, 对于一个实现没有标注的网络来说, 就会变得跟聚类类似, 得首先想好要把网络分为多少个community, 然后再用无监督的方法去分。

这个东西的前身可以认为是这个Granovetter对社会关系做的一个抽象分析

![cs224w-lec4-1](/images/cs224w/cs224w-lec4-1.png)

他认为同属于一个community里面的人都有strong connection, 互相基本都认识, 因此边也较多, 可以理解为clustering coefficiency较高。而不同团体之间互相认识的人少, 属于weak connection, 边也就少。

# 2. The measurement of edges

如何分析一条边的属性呢, 定义了一个叫`Edge Overlap`的性质
$$O_{ij} = \frac{|N(i) \cap N(j) \backslash \{i,j\}|}{|N(i) \cup N(j) \backslash \{i,j\}|}$$

![cs224w-lec4-2](/images/cs224w/cs224w-lec4-2.png)

可以看到, edge overlap越高, 说明这两个点周围点的重合程度也就越高, 而当edge overlap为0时, 这就是一条`bridge edge`, 也就是如果原来图是连通的, 然后这条边没了, 那么图就不再连通了。

Edge overlap有时候比单纯的边的权重要效果好, 详细可以看课件, 里面提到如果把两个人之间的通话次数当作边的权重, 其在一个edge removal的实验中效果没有edge overlap明显。

不过这些都是讲了一些背景和edge的知识，和下面介绍的community的算法关系不太大, 了解就好。

# 3. Properties of community

Community的性质可以用一个叫`Modularity`的属性来描述, 它代表网络切分成community的效果到底好不好。需要注意的是modularity是**对于整个网络所有的community整体来计算, 而不是仅仅针对某一个或几个community**。

整个图的**Modularity Q**等于\\( \sum_{s \in S} [(\\# edges\ within\ group\ s) - (expected\ \\#edges\ within\ group\ s)] \\)

这里那个*expected #edges within group s*是需要构建一个`null model`, 也叫`configuration model`来对比的, 这个模型其实就是对图引入了一些随机化, 使得**生成的图和原图有相同的度分布, 但边的位置不同**。

在Null Model生成的`multigraph(两个点可以有多条边)`中, **图中的两点之间边数量的期望**(如果是有向图或者无向有权图, 这里也会相应变成平均入度出度或者平均权重)

假设节点i和j的度分别为\\( k_i, k_j \\) (这里应该指的是在原始图中的度?), 那么\\( e_{ij} = k_i \cdot \frac{k_j}{2m} \\), 其中`m`是图中边的数量, 那么整个图边数量的期望的就是
$$
\begin{aligned}
E &= \frac{1}{2} \sum_{i \in N} \sum_{j \in N} \frac{k_i k_j}{2m} \\ &= \frac{1}{2} \cdot \frac{1}{2m} \sum_{i \in N} k_i(\sum_{j \in N} k_j) \\ &= \frac{1}{4m} 2m \cdot 2m = m
\end{aligned}
$$

其中\\( \sum_{u \in N} k_u = 2m \\), 乘以\\( \frac{1}{2} \\)的意义在于求和的时候每条边算了两遍。可以看到, 这个公式和图里面实际的边的数量是一致的, 所以可以看出这个图在期望上和原始图具有相同的边的数量, 至于度的分布, 这个暂时没有看到证明。

这样一来整个modularity就可以用公式定量表示了
![cs224w-lec4-3](/images/cs224w/cs224w-lec4-3.png)

这里应该代表两点间的实际的边的总权重(如果是无权图就是边的数量), `Q`越大, 说明某些点对的边的数量远大于随机图中的期望数量, 也就代表着这个图里面的community成分更多。

因此, 如果能有一个方法找最大化modularity, 就可以找到对应的community。

# 4. Louvain Algorithm
Louvain算法是一个**贪心算法, 用来检测图中的commmunity, 时间复杂度是\\( O(nlogn) \\)**, 其采用了一种层次化的思想来找。

算法需要多轮迭代, 每轮步骤如下 (为了准确, 这里先用英文slide原话说下)
* **Phase 1:** Modularity is optimized by allowing only local changes to node-communities memberships
* **Phase 2:** The identified communities are **aggregated** into super-nodes to build a new network.
* **Goto Phase 1**

这里需要说明一下的是phase 1是什么意思, 其实就是不停的选node, 然后挨个试其他的communities, 看看如果把这个node从当前community移动到另外的community, modularity会增加多少, 然后选择增量最大的node和community作为第一步的结果。**如果Phase 1中发现不论怎么选都无法使得整个图的modularity增加, 那么算法结束, 输出检测到的community结果。**

第二步就是把同属于一个community的所有点都看作一个巨点, 然后这样可以抽象得到一个新的图, 再在这个图上开始下一轮的算法迭代

## 4.1 Louvain: 1st phase (Partitioning)
每次操作的modularity变化\\( \Delta Q=\Delta Q(i \rightarrow C) + \Delta Q(D \rightarrow i) \\)
分别代表把i放进C和把i移出D造成的modularity变化。
![cs224w-lec4-4](/images/cs224w/cs224w-lec4-4.png)

给出了\\( \Delta Q(i \rightarrow C) \\)的推导, 但是没有给出移出D的推导

![cs224w-lec4-5](/images/cs224w/cs224w-lec4-5.png)

左边的看懂了, 右边的过程倒是看懂了, 但是右上角那个“Modularity contribution before merging node i”这个没太看懂什么意思, 为什么还要把i的modularity减掉? 不是直接算C的modularity就可以了吗？如果有懂的大佬指教下。

Phase 1停止时, 代表找到了这个样一个使得modularity增益最大的node和community的组合操作， 进入Phase 2.

## 4.2 Phase 2

第二步主要是形成super-nodes, 就是把第一步中得到的各个community中节点的权重进行相加, 当作新的节点的权重。
步骤如下
- The communities obtained in the first phase are contracted into super-nodes, and the network is created accordingly:
  - Super-nodes are connected if there is at least one edge between the nodes of the corresponding communities
  - The weight of the edge between the two super-nodes is the sum of the weights from all edges between their corresponding communities
- Phase 1 is then run on the super-node network

![cs224w-lec4-6](/images/cs224w/cs224w-lec4-6.png)

但其实上面这张图最后一步形成的这个边的权重有点没看懂, 难道不是左边22, 右21边吗?

算法的伪代码
![cs224w-lec4-7](/images/cs224w/cs224w-lec4-7.png)

看起来就比较复杂，不过大意就是和上面描述的一致, Phase 1就是负责找modularity增加最大的点和community, Phase 2就更新V, E及其相应的权重。

# 5. Detecting Overlapping Communities: BigCLAM

主要思想是极大似然估计, 比如给定模型参数F, 我们可以得到生成当前图的概率\\( P(G|F) \\), 那么给定了图G, 如何估计参数F呢?
BigCLAM就给出了这样一个方法。
![cs224w-lec4-8](/images/cs224w/cs224w-lec4-8.png)

这个\\( F_{uA} \\)就代表节点u和community A之间的membership strength(英文感觉翻译过来总不准确, 就直接英文了)
![cs224w-lec4-9](/images/cs224w/cs224w-lec4-9.png)

有了这个参数后就可以计算u和v之间有边的概率, 然后就可以套用上面提到的那个条件概率公式来求极大似然估计, 进而找出能相应的F参数组合.

极大似然估计可以看csdn的这篇[极大似然估计详解][3], 或者英文的这篇[A Gentle Introduction to Maximum Likelihood Estimation][4], 英文的这篇有代码, 看起来可以加深理解, 但是我没有跑代码。

![cs224w-lec4-10](/images/cs224w/cs224w-lec4-10.png)

最后还提了一下求导加速的方法, 作为优化可以看看slides.

[1]: http://web.stanford.edu/class/cs224w/
[2]: https://www.bilibili.com/video/av90106649?p=4
[3]: https://blog.csdn.net/qq_39355550/article/details/81809467
[4]: https://towardsdatascience.com/a-gentle-introduction-to-maximum-likelihood-estimation-9fbff27ea12f
