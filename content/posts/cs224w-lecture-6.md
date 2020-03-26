---
title: "Cs224w Lecture 6 - Message Passing and Node Classification"
date: 2020-03-23T18:37:15+08:00
tags: [graph, cs224w]
categories: ["stanford courses"]
draft: false
---

Lecture 6主要讲的是如何利用一些机器学习和图网络的方法来实现对图中节点的分类

<!--more-->

# 1. Node Classification in Network

课件前半部分感觉一些介绍性的话比较多, 所以图比较少, 主要内容就是说网络中存在所谓的*correlation*, 主要有三种

* Homophily: 同质性, 同一个群体里面的若干个体具有同质性
* Influence: 个体的行文会受到群体的影响 

* Confounding: 环境会对个体和群体都有影响

基本都是概念的东西, 目前而言理解就好。



那么, 在网络中, 给定一些节点的类别, 如何确定剩下其他节点的类别呢？根据利用信息的多少, 大概分为三个层次

- Local Classifier: 基本就是传统的一些机器学习的方法, 因为只看节点本身的特征, 然后根据特征来分类。这种做法好处就是简单, 可以直接套用现有的机器学习的方法, 但是缺点是一般效果不好, 因为显然它忽略了节点在网络中的特性, 而只考虑了节点本身
- Relational Classifier: 相比于local的方法, 它利用了网络中的一些相互关系(correlation)
- Collective Inference: 会传播相互关系

个人觉得上面这几个定义其实都比较宽泛, 视频里Michele也说了, 这只是一个大概的归纳, 实际不同方法之间的区分没有这么严格。一般用的比较多的还是`collective inference`

所谓Inference分为严格的和近似的, **严格的推理(Exact Inference)是一个NP-hard**问题, 因此更主要关注的**近似推理 (Approximate Inference)**。那么Exact和Approximate之间的区别是什么呢?

> 假定我们把每个节点都表示为一个离散变量, 并用\\( p \\)来表示其联合质量函数(个人理解是联合分布函数的离散版)。
>
> Exact指的是一个节点的边缘分布是**所有其他节点**的和, 这样会导致计算复杂度急剧增大。
>
> Approximate相比于exact就是缩小了范围, 例如不需要所有节点, 只需要获得邻居的值即可(可以是一度也可以是二度)。

可以看到Exact就是会充分利用网络的结构信息, 而Approximate就是舍弃了部分节点(例如离当前节点距离太远的节点)等来降低计算复杂度。

Approximate Inference也可以分为三类

* Relational Classifiers
* Iterative classification (其实Relational也有多轮迭代, 这里的iterative感觉可能指的是还利用了节点的邻居的一些属性)
* Loopy Belief Propagation(这个名字暂且叫信任传播吧, 感觉翻译总是非常奇怪, 大概内容就是在把图以条件概率的形式表达出来了, 然后再利用相互关系多轮迭代)

开始看着一脸懵, 不如结合slide的例子来看这几种方法。

# 2. Probalistic Relational Classifier

基本思想是一个节点属于某个标签\\( Y_i \\)的概率是其所有邻居属于这个标签的概率的加权平均。最开始的时候, 可能只有一部分节点有标签, 剩下一些节点可能暂时没有获得标签, 对于那些暂时没有标签的节点, 将其初始化为均匀分布, 也即每个节点属于各个标签的概率相同。这样可以确保初始化的时候各个节点都有一个概率, 然后在每轮迭代的时候会更新概率。

对于一个节点\\( i \\)和一个标签\\( c \\)
$$
P(Y_i = c) = \frac{1}{\sum_{(i,j) \in E} W(i,j)} \sum \limits_{(i, j) \in E} W(i,j) P(Y_j = c)
$$
其中\\( W(i, j) \\)就是边(i, j)的权重

算法就是初始化节点, 然后每轮重新计算节点的各个概率并更新。直到某一轮的时候所有节点的概率值和上一轮相比没有发生变化, 或者到达一个设定的最大迭代轮数时, 算法才会停止。

![cs224w-lec6-1](/images/cs224w/cs224w-lec6-1.png)

![cs224w-lec6-2](/images/cs224w/cs224w-lec6-2.png)

![cs224w-lec6-3](/images/cs224w/cs224w-lec6-3.png)

![cs224w-lec6-4](/images/cs224w/cs224w-lec6-4.png)

![cs224w-lec6-5](/images/cs224w/cs224w-lec6-5.png)

![cs224w-lec6-6](/images/cs224w/cs224w-lec6-6.png)

![cs224w-lec6-7](/images/cs224w/cs224w-lec6-7.png)

![cs224w-lec6-8](/images/cs224w/cs224w-lec6-8.png)

![cs224w-lec6-9](/images/cs224w/cs224w-lec6-9.png)



这种方法的不足之处在于

1. 不一定能收敛, 有可能到了最大迭代次数仍然没收敛
2. 仅使用了邻居节点, 而忽略了节点特征(node feature information)



# 3. Iterative Classification

这个迭代式的方法感觉课件描述的有点奇怪, 我理解的是不再直接看节点的邻居, 而是分为了两个步骤

1) **BootStrap phase**, 将节点先转换为一个一维的向量\\( a_i \\), 然后利用local classifier(例如SVM, kNN之类)对节点的向量进行计算分类, 得到初始化的标签\\( Y_i \\)

2) **Iterative phase**, 每轮首先更新节点的向量\\( a_i \\), 然后根据向量更新节点的标签\\( Y_i = f(a_i) \\), 而更新标签是一个hard assignment(原文这么说的, 猜测意思是这一步比较费时)。然后不停迭代直到收敛或者达到最大迭代轮数的阈值

设置最大迭代轮数的原因是**这种方法不一定会收敛**

## 3.1 Example: Web page classification

给了一个例子, 对于网页分类, 如果仅仅根据网页内容来分类而不考虑网页之间的互相关联, 那么两个内容相似的网站(例如一个盗版网站抄袭一个正版网站的大部分内容)就很可能会被给予同样的标签。下图是例子

![cs224w-lec6-10](/images/cs224w/cs224w-lec6-10.png)



解决办法就是不仅仅利用节点的词向量\\( w = word\  vectors\ of\ current\ page \\), 还利用相邻节点的标签信息
$$
I = (I_A, I_B, O_A, O_B), I = in, O = out
$$
其中\\( I_A = 1\\)当前节点的所有入边中, 至少有一个邻居的标签为1, \\( I_B, O_A, O_B \\)的定义类似。

然后, 剩下可以分为三个步骤

### (1) Training set

首先最开始的时候, 我们是什么标签都没定的, 有的只有网页文本, 及提取的一些特征。仅利用网页文本的word vectors训练一个local classifier来对文本进行一个最初的分类, 把这一步训练出的模型保存至下一步使用。

### (2) Bootstrap

利用上一步中的模型对各个节点进行第一次分类, 获得初始化的标签, 然后就**可以利用这些初始化的标签对节点的邻居特征(上文中提到的\\( I_A, I_B, O_A, O_B, \dots \\)等等 ) 进行初始化**。

这一步完成后, 各个节点就相当于有了一个完整的初始化特征, 包括其自身的word vectors, 以及其各个邻居的特征关系。

### (3) Iterate

这一步的内容就是根据上一步得到的初始化特征运行多轮算法的迭代, 每轮

* 更新节点的相关特征
* 利用更新后的特征对节点进行重新分类

一直迭代到各个节点的标签不再发生变化, 或者达到了最大迭代轮数。注意**这里的收敛指的是节点的分类标签不再变化, 而不是节点的特征**

![cs224w-lec6-11](/images/cs224w/cs224w-lec6-11.png)

![cs224w-lec6-12](/images/cs224w/cs224w-lec6-12.png)

![cs224w-lec6-13](/images/cs224w/cs224w-lec6-13.png)

![cs224w-lec6-14](/images/cs224w/cs224w-lec6-14.png)

![cs224w-lec6-15](/images/cs224w/cs224w-lec6-15.png)

![cs224w-lec6-16](/images/cs224w/cs224w-lec6-16.png)

![cs224w-lec6-17](/images/cs224w/cs224w-lec6-17.png)



## 3.2 Application of iterative classification framework: fake reviewer/review detection

这是Kumar等人在ACM Web Search and Data Mining, 2018发表的论文[REV2: Fraudulent User Predictions in Rating Platforms][1]



Fake review(不知为何老是想到fake news...)的检测一般会利用个人的一些行为 + 语义内容检测等, 个人特征指的是地址位置、会话历史等, 而语言特征则是说话的常用词和结构之类。

然而, **个人的行文和评论的内容是很容易伪造的**, 比如大家经常能看到微博和一些网店下面都是控评/刷好评的。

相比之下, **图网络的结构是更难伪造的**, 比如这些僵尸号都给同一件物品刷了评论等特征。



基本思路就是把用户和商品分别看成一个二分图等两部分, 然后用户可以给一些商品打分, 一共维护三个评分

* 每个用户的fairness评分, \\( F(u) \in [0, 1] \\)
* 每个评分有一个reliability评分, \\( R(u, p) \in [0, 1] \\)
* 每个商品有一个goodness评分, \\( G(p) \in [-1, 1] \\)

在每一轮的更新公式为
$$
F(u) = \frac{\sum \limits_{(u, p) \in Out(u)} R(u, p)}{|Out(u)|}
$$

$$
G(p) = \frac{\sum \limits_{(u,p) \in In(p)} R(u,p) \cdot score(u,p)}{|In(p)|}
$$

$$
R(u,p)=\frac{1}{\gamma_1 + \gamma_2}(\gamma_1 \cdot F(u) + \gamma_2 \cdot (1-\frac{|score(u,p)-G(p)|}{2}))
$$

其中\\( \gamma_1, \gamma_2 \\)是超参数, 分别调节用户评分的fairness和商品的goodness

![cs224w-lec6-18](/images/cs224w/cs224w-lec6-18.png)

![cs224w-lec6-19](/images/cs224w/cs224w-lec6-19.png)

![cs224w-lec6-20](/images/cs224w/cs224w-lec6-20.png)

![cs224w-lec6-21](/images/cs224w/cs224w-lec6-21.png)



当然, 所有这些迭代公式都需要一个初始化的分数(不知道这是不是就是ml中所说的冷启动?), 默认都是1, 下面是个例子。

![cs224w-lec6-22](/images/cs224w/cs224w-lec6-22.png)

![cs224w-lec6-23](/images/cs224w/cs224w-lec6-23.png)

![cs224w-lec6-24](/images/cs224w/cs224w-lec6-24.png)

![cs224w-lec6-25](/images/cs224w/cs224w-lec6-25.png)

![cs224w-lec6-26](/images/cs224w/cs224w-lec6-26.png)



这种方法的好处在于

* 保证会收敛(暂时没看到证明)
* 在收敛之前需要迭代的轮数是有上限的, 也就是最多迭代一个固定轮数后就一定收敛
* 时间复杂度和图中边的个数线性正相关



最后把fairness得分低低用户当作虚假用户, 精确率 127/150, 而且top 80的结果都是100%准确的。

# 4. Collective Classification: Belief Propagation

更精确的说, 这节叫*Loopy belief propagation*, 名字有点奇怪, 原文是这样定义的

> - Belief Propagation is a dynamic programming approach to answering conditional probability queries in a graphical model
> - Iterative process in which neighbor variables “talk” to each other, passing messages

这个迭代过程会一直重复, 直到各个节点达成共识, 然后将最后这一步的belief当作结果输出。

感觉有点云里雾里, 其实这个名字和后面的方法感觉并不是严格对应, 看个例子

## 4.1 Message passing

假定

**任务**: 计算图中有多少节点

**条件**: 每个节点只能和其邻居进行通信

**解决办法**: 每个节点监听从邻居节点发送过来的消息, 更新, 然后再传递给新的邻居。

![cs224w-lec6-27](/images/cs224w/cs224w-lec6-27.png)

![cs224w-lec6-28](/images/cs224w/cs224w-lec6-28.png)

![cs224w-lec6-29](/images/cs224w/cs224w-lec6-29.png)

感觉其实就是和DFS有点像, 只不过换成了图的版本。下面这个例子会更清楚

![cs224w-lec6-30](/images/cs224w/cs224w-lec6-30.png)

![cs224w-lec6-31](/images/cs224w/cs224w-lec6-31.png)



不过要注意这种方法要求**图中不能有环**, 不然会无法停止, 不过个人觉得如果魔改下应该还是可以处理环的结构的。

## 4.2 Loopy BP Algorithm

这节是主要解决, 一个节点*i*发送给节点*j*的消息到底包括什么

> Each neighbor *k* passes a message to *i* its beliefs of the state to *i*

什么叫belief？其实就是你当前获得的其他节点的信息的表示。先看几个基本概念

- **Label-label potential matrxi \\( \psi \\)**: \\( \psi (Y_i, Y_j) \\)表示如果节点*j*的邻居*i*处于状态\\( Y_i \\), 那么节点*j*处于状态\\( Y_j \\)的概率

- **Prior belief \\( \phi \\)**:\\( \phi_i (Y_i) \\)代表节点*i*属于状态\\( Y_i \\)的概率

- \\( m_{i \rightarrow j } (Y_j) \\) : 从i发到j的消息, 代表节点i处于状态\\( Y_j \\)的概率, 实际中不会仅仅发送状态Yi, 而是包括了所有可能状态的概率

- **\\( \mathcal{L} \\)**: 代表所有状态的集合, 实际中比较少用, 一般是数学证明的时候用到

$$
m_{i \rightarrow j}(Y_j) = \alpha \sum \limits_{Y_i \in \mathcal{L}} \psi(Y_i, Y_j) \phi_i(Y_i) \prod \limits_{k \in \mathcal{N_i \backslash j}} m_{k \rightarrow i} (Y_i)
$$

$$
b_i(Y_i) = \alpha \phi_i(Y_i) \prod \limits_{j \in \mathcal{Ni}} m_{j \rightarrow i}(Y_i), \forall Y_i \in \mathcal{L}
$$

![cs224w-lec6-32](/images/cs224w/cs224w-lec6-32.png)

![cs224w-lec6-33](/images/cs224w/cs224w-lec6-33.png)

那么, 如果图里面有环呢? 会导致**来自不同子图的消息之间不再相互独立, 但是我们仍然可以运行BP算法**, 因为它是一个local算法, 因此它不会收到环的影响。至于环会不会对最后的结果准确度有影响, 这个可能就要看实际的场景了。

下面是一个环可能导致出错的例子

![cs224w-lec6-34](/images/cs224w/cs224w-lec6-34.png)

上图中, belief在第一圈传播的时候还是对的, 但是到第二圈的时候经过求和T的值会加倍, 因此循环的次数越多, 标签为\\( T \\)的可能性就越大, 而这和原始的标签概率已经不同了。当然这是一个极端的例子, 大部分情况下环的影响是比较弱的。



**信任传播的优点**

* 易于编程, 也容易并行化
* 较为通用, 可以被稍作修改后套用到任何图形模型中

**缺点**

* 不一定会收敛, 尤其是如果图中有过多循环的时候



对于上面公式中一些潜在的参数, 需要经过训练来确定最优值, 利用梯度下降来训练, 但训练时也会有收敛性方面的问题(什么问题? 这里我没太看懂)

## 4.3 Application of Belief Propagation: Online Auction Fraud

Pandit等人发表在World Wide Web conference 2007上的[Netprobe: A Fast and Scalable System for Fraud Detection in Online Auction Networks][2]

在购物网站上, 用户和卖家分别可以给对方打分, 但总是免不了会有一些不守规则的人, 比如卖家发假货, 买家恶意给差评等。

那些不守法的卖家或买家之间会不会互相勾结, 进行刷分呢? 大部分不会, 因为如果这样做, 一个人被抓了, 所有人都暴露了

可以将整个网络看成一个类似二分图的形式, 总共有三种角色

* 正常买家/卖家: 诚实的人 :)

* 同谋: 大部分情况和正常买家/卖家进行交易, 表面上看起来似乎是正常的
* 欺诈者: 和同谋进行大量正常交易来尽量比避免被风控, 但是遇到正常买家/卖家的时候就会有欺诈行为

![cs224w-lec6-35](/images/cs224w/cs224w-lec6-35.png)

![cs224w-lec6-36](/images/cs224w/cs224w-lec6-36.png)

上面的表格中, 不同的欺诈者之间相互沟通的概率\\( \epsilon_p \\)一般会比较低, 而和accomplice进行交易的概率比较高.

下面是一个例子

![cs224w-lec6-37](/images/cs224w/cs224w-lec6-37.png)

其中无色的是诚实的用户, 黄色的是同谋accomplice, 红色的节点是欺诈者。每个节点的标签就是其各个belief中最高值对应的标签

# 5. Conclusion

![cs224w-lec6-38](/images/cs224w/cs224w-lec6-38.png)



[1]: https://cs.stanford.edu/~srijan/pubs/rev2-wsdm18.pdf
[2]: cs.cmu.edu/~christos/PUBLICATIONS/netprobe-www07.pdf

