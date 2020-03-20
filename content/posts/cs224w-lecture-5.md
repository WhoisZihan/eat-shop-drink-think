---
title: "Cs224w Lecture 5 - Spectral Clustering"
date: 2020-03-16T21:30:04+08:00
tags: [graph, cs224w]
categories: ["stanford courses"]
draft: false
---

Lecture 5讲的是Spectral Clustering, 就是如何利用spectral的方法对图进行一定的处理, 以便进行分类, 并使得切割方案和最优解尽可能接近。

<!--more-->

# 1. Overview

Clustering简单说就是把图中的节点分为若干类, 且若干类之间的cut(割)接近最优割

作者说Spectral Clustering Algorithms有三个步骤

1. **Pre-processing**

- Construct a matrix representation of the graph

2. **Decomposition**

- Compute eigenvalues and eigenvectors of the matrix
- Map each point to a lower-dimensional representation based on one or more eigenvectors

3. **Grouping**

- Assign points to two or more clusters, based on the new representation



不过，这个定义看起来有点让人一下子不知道到底是什么，而且涉及一些数学知识，因此需要介绍一下背景，定义一下问题。

# 2. Background

## 2.1 Graph Partition

首先从*graph bi-partition*开始, 先考虑如何把一个图切割为两个cluster, 然后再考虑推广到多个cluster。但怎样定义切割的好不好？

一个好的切分满足下面两个原则

* Maximize the number of within-group connections
* Minimize the number of between-group connections



这样把一个图切割为两部分的这样一个方法，叫做图的一个`cut(中文叫割)`

$$ cut(A, B) = \sum_{i \in A, j \in B} w_{ij} $$

也即两个group之间所有的边的权重之和, 对于无权图这就是边的个数

### 2.1.1. Criteria of cut

**(1) Minimum-cut**

第一种是最直接的, 就是使得两个group之间的cut值最小
$$
arg\ min_{A,B} cut(A, B)
$$
![cs224w-lec5-1](/images/cs224w/cs224w-lec5-1.png)

但是这种方法只考虑到了cluster外部的一些边, 而没有考虑到cluster内部的一些连通的特性。一个极端的例子是所有的节点都在一个cluster内, 那自然cut最小, 因为根本就不用cut, 当然在实际情况中一般会对两边的cluster做个规定, 以避免出现这种情况。

**(2) Conductance**

这个直译过来叫传导性
$$
\phi (A,B) = \frac{cut(A,B)}{min(vol(A), vol(B))}
$$
其中\\( vol(A)\\)对于有权图是A中节点的权重之和

对于无权图\\( vol(A) = \sum_{i \in A} k_i \\) (number of edge end points in A)

这种标准相比于minimum-cut可以产生更均衡的划分结果, 然而它的一个**缺点是计算复杂度太高, 是个NP-Hard问题**。

## 2.2 Spectral Graph Partition

那么如果直接计算是NP-Hard问题, 可以采用spectral方法, 所谓spectral其实就是将图用矩阵表示, 然后对矩阵求特征值和特征向量, 然后利用特征向量来表示各个节点, 而不再是直观的图的度之类的。这块需要先补一点数学和图论知识。

### 2.2.1 d-Regular Graph

这里先以`d-Regular`图为例, 所谓的`d-Regular`, 意思就是图中每个节点的度都为`d`, 例如一个三角形, 每个节点的度都为2(但要注意**这样的图并不一定是所有节点都连通的**, 后面会提)

对于这样的一个图的邻接矩阵\\( A, A_{ij} = 1如果(i, j) \in E, 否则为0 \\), 其有如下性质

* \\( x = (1,1, \dots , 1) \\) 是其一个特征向量, 因为\\( Ax = (d,d,\dots , d) = dx \\)
* 上式也说明了\\( d \\)是其一个特征值
* \\( d\\)是A最大的特征值

上面前两条很容易推出来，第三条在slide中有一个证明，这里简单提下。大致步骤就是

1. 首先\\( x = c \cdot (1,1, \dots , 1) \\)确定是一个特征向量(其中`c`是个常数), 因此不管最大的特征值是多少, 最后和`x`乘出来的向量里面肯定每个元素都相等
2. 假设\\( S= node i with maximum value of x_i  \\) (如果是1中的情况S就包含所有的节点)
3. 但如果假设存在这样一个向量为\\( y \\)其并不是所有的元素都相等(也即不是\\( (1,1,\dots , 1) \\)的倍数), 那么就必然存在某些label是\\( y_i \\)的节点\\( i \\)不在\\( S \\)中
4. 如果3成立, 那么就必然存在至少一个节点\\( j \in S \\), 而其邻居是\\( i \notin S \\), 那么节点\\( j \\)对应的label的值就小于`d`, 但在第1步汇总以及说了, 不管最大的特征值是多少, 最后乘出来的向量每个元素都相等, 而这已经不相等了, 因此\\( y \\)不是特征向量, 而\\( d \\)也就是最大的特征值了。

比较绕, 反正看懂了就行, 实际大部分情况只是用到一个结论。



另外需要说明的是这个性质对于非连通图仍然适用, 原理也很简单, 具体可以看slide

**A bit of intuition**

这里作者提到了一个有意思的直觉, 就是假定我们把特征值从小到大排列\\( \lambda_1 \le \lambda_2 \le \cdots \lambda_n \\), 那么

* 如果一个`d-Regular`图有两块完全相同的子图, 且它们不连通, 那么\\( \lambda_n = \lambda_{n-1} \\), 也即第二大的特征值和最大的特征值相等
* 如果一个`d-Regular`图有两块完全相同的子图, 且它们连通但两个子图之间的边非常少, 那么\\( \lambda_n - \lambda_{n-1} \approx 0 \\), 也即第二大的特征值和最大的特征值不相等, 但非常相近

具体证明过程

![cs224w-lec5-2](/images/cs224w/cs224w-lec5-2.png)

其中几个要点，首先是不同的特征向量之间是互斥的, 所以其他的特征向量\\( x_k \cdot x_1 = 0 \\), 而\\( x_1 = (1,1, \cdots ,1) \\), 因此可以得出\\( \sum_i x_k[i] = 0\\), 也即其他的特征向量的每项之和都为0.

其次, 一般都会将特征向量单位化, 这样可以作为基向量, 因此\\( \sum_i  x_k^2 [i] = 1\\)。

所以这就使得\\( x_k \\)中一定有一些元素 > 0, 一些元素 < 0, 所以可以将0(当然也可以根据其他数字)作为一个分界线, 使得\\( x_k[i] \gt 0 \\)的节点和\\( x_k[i] \lt 0 \\)的节点分别划分到两个不通的group中, 也就是利用特征向量的正负号来完成分类。

## 2.3 图的拉普拉斯矩阵

拉普拉斯矩阵定义上十分简单
$$
L = D-A
$$
其中\\(D\\)和\\( A \\)分别是图的度矩阵(Degree Matrix)和邻接矩阵(Adjacency Matrix).

对于Degree Matrix, 其\\( D_{ij} = 1, if i = j, otherwise 0\\), 也即只有对角线的值是其度数, 其他的值为0.

对于Adjacency Matrix就很直观了, \\( A_{ij} = 1, if (i, j) \in E, otherwise 0 \\), 就是如果(i, j)是一条边, 那么对应的值就为1, 其他为0。

拉普拉斯矩阵就是这两个矩阵相减, 且**具半正定的**。拉普拉斯矩阵具有以下性质

* (a) 所有的特征值\\( \gt 0 \\)
* (b) 对于任何(纬度匹配的)向量\\( x \\), \\( x^T L x \ge 0 \\)
* (c) L可以被写成\\( L = N^T N\\)的形式

![cs224w-lec5-3](/images/cs224w/cs224w-lec5-3.png)

这三个条件可以互相证明, 但是仍然得证明一个, 一般我在网上查到的证明都是证明(c), 那么具体怎么构造这个矩阵呢?

### 2.3.1. 拉普拉斯矩阵半正定性证明

我搜到了大概三种不同的方法

* 首先这个[fuqiang liu的知乎回答][1]提供了思路和构造算子矩阵的方法, **但是感觉他把矩阵的两个维度说反了**,  因为如果\\( In(G) \\)的尺寸为\\(  V \times E\\), 那么最后得到的矩阵的尺寸就成了\\( E \times E \\), 但实际上应该反过来, 才能得到\\( V \times V\\)的矩阵。

* 其实个人觉得讲的更好的是[【其实贼简单】拉普拉斯算子和拉普拉斯矩阵][2], 这里面提出了构造\\( K_G\\), 从而使得拉普拉斯矩阵\\( L = K_G^{'} K_G^T\\)。值得注意的是, 他里面提出的方法适用于有权图, 而对于我们这里讨论的无权图属于更简化的情况, 其\\( K_G = K_G^{''} \\)。

* 这篇[拉普拉斯矩阵(Laplacian Matrix) 及半正定性证明][3]里面最后那张图也不错, 它构造的\\( \nabla_G \\)矩阵恰好就是和第一个知乎回答里面构造的反过来的, 是一个\\( E \times V \\)的矩阵, 这样相乘后才能得到一个\\( V \times V\\)的拉普拉斯矩阵，而且



另外还有一些标准化和random walk的相关内容, 这个课程就暂时没有深入了

* [图的拉普拉斯矩阵和拉普拉斯算子有什么关系？][4]
* [图卷积神经网络(GCN)详解:包括了数学基础(傅里叶，拉普拉斯)][5]



## 2.4 Rayleigh Theorem, 瑞利商

瑞利商也是一个硬核的代数知识, 简单来说, 它定义一个Hermite矩阵(拉普拉斯矩阵也满足Hermite矩阵性质) \\( M \\)的瑞利商为
$$
R(L, x) = \frac{x^TLx}{x^Tx}
$$
下面是对于瑞利商的说明和证明, 个人感觉第一个写的最好

* [瑞利商与极值计算][6], 这篇硬核数学证明, 线代学的浅了有些步骤其实暂时没看懂...

* [每週問題 November 23, 2015][7], 这位繁体大哥似乎直接是把瑞利定理当作现成的来用了

* [瑞利商（Rayleigh quotient）与广义瑞利商（genralized Rayleigh quotient） ][8], 这个也没有证明, 直接认为 \\( \lambda_{min} \le 瑞利商 \le \lambda_{max} \\)

* [拉普拉斯矩阵(Laplace Matrix)与瑞利商(Rayleigh quotient)][9], 提到了PCA是瑞利商理论的一个应用。



这里先假定已经理解了瑞利商, 对于我们的graph partition有什么帮助呢？实际上
$$
\lambda_2 = \min \limits_{x: x^Tw_1=0} \frac{x^TLx}{x^Tx}
$$

其中\\( w_1 \\)是\\( \lambda_1 \\)对应的特征向量。



这个式子说明如果把瑞利商当作目标函数求最小值, 得到的结果就是第二小的特征值\\( \lambda_2  \\) (这里假定特征值升序排列, 有的地方可能是按照降序排列的, 需要区分开)

![cs224w-lec5-4](/images/cs224w/cs224w-lec5-4.png)

![cs224w-lec5-5](/images/cs224w/cs224w-lec5-5.png)

![cs224w-lec5-6](/images/cs224w/cs224w-lec5-6.png)



根据各种等式变换可以将\\( \lambda_2 \\)表示成上图中的平方和形式,  `Fiedler`等人在73年的时候据此提出了一种找最优割的方法
$$
\min \limits_{
y_i \in \mathbb{R^n}, \sum_i y_i = 0, \sum_i y_i^2 = 1
} f(y) = \sum_{(i,j)\in E} (y_i - y_j)^2 = y^TLy
$$


y的取值范围是全体实数。当然, **这是假定以0为分割线的目标函数**, 如果分割线不同, 则目标函数也需要变化。对这个目标函数求解得到的最小值和\\( \lambda_2 \\)相等, 而使得\\( f(y) \\)取最小的\\( x \\)就是\\( \lambda_2 \\)对应的特征向量, 也称之为`Fielder vector`。然后我们就可以**利用\\( x_i \\)的正负号来决定各个节点i属于哪一个cluster, 这就和最开始的graph partition的目的结合起来了, 至少现在可以划分为两堆了**

![cs224w-lec5-7](/images/cs224w/cs224w-lec5-7.png)

至于这个划分和最优划分的差别, 假设\\( \beta = \frac{( \\# edges\ from\ A\ to\ B)}{|A|} \\)是最优值, 那么有
$$
\frac{\beta^2}{2k_{max}} \le \lambda_2 \le 2\beta
$$
其中\\( k_max \\)是图中节点最大的度, 证明也给出了一部分

![cs224w-lec5-8](/images/cs224w/cs224w-lec5-8.png)



# 3. Spectral Clustering Algorithm

讲完了一些基础, 接下来就该到了实际的算法部分了, 还是先以划分为两部分为例, 之前讲的三个步骤也就有了对应。其实算法本身很简单, 就是构造拉普拉斯矩阵, 然后取其第二个特征向量来对节点进行分类, 有价值的是背后的数学思想。

![cs224w-lec5-9](/images/cs224w/cs224w-lec5-9.png)

![cs224w-lec5-10](/images/cs224w/cs224w-lec5-10.png)



现在可以有效的对节点进行二分类了, 可clustering很多时候不会只分两类就结束了, 比如下面

![cs224w-lec5-11](/images/cs224w/cs224w-lec5-11.png)

![cs224w-lec5-12](/images/cs224w/cs224w-lec5-12.png)

如果想将其分为4类, \\( x_2 \\)以0为分割线只能分为两类, 所以现在**要么继续在\\( x_2 \\)的基础上进行层次化分类, 要么就想办法利用更多的特征向量, 比如如果使用特征向量\\( x_3 \\)来对节点进行分类的话, 效果就很好**。因此也就有了所谓的`k-Way Spectral Clustering`

## 3.1 k-Way Spectral Clustering

![cs224w-lec5-13](/images/cs224w/cs224w-lec5-13.png)

![cs224w-lec5-14](/images/cs224w/cs224w-lec5-14.png)

![cs224w-lec5-15](/images/cs224w/cs224w-lec5-15.png)

大致思路就是找到 \\( eigengap \delta_k \\)最大的一个\\( k \\)值, 然后使用\\( \lambda_1, \lambda_2, \dots , \lambda_k \\), 把它们当作基向量来表示图中的各个节点, 最后使用例如k-means等方法来对节点进行分类。



# 4. Motif-Based Spectral Clustering

这是扩展的一个章节, 上面提到的cut都是针对边的`edge-cut`, 但是有很多时候(例如在生态环境中各个生物及它们之间的捕食关系)存在一些模式, 针对motif也可以有cut的存在。

![cs224w-lec5-16](/images/cs224w/cs224w-lec5-16.png)

如果要使用spectral的方法来对motif进行分类的话, 就需要先对图做一些转换, 然后套用前面介绍的那一系列方法

![cs224w-lec5-17](/images/cs224w/cs224w-lec5-17.png)

第一步就是先将有向图转换为无向有权图, 转换后每条边\\( (i,j)\\)的权值就是转换前这条边所在的motif的个数。比如如果一条边同时属于3个motif, 那么在无向图中它的权值就为3。

接下来对转换后的图求得起拉普拉斯矩阵, 然后利用上面提到的这些spectral的方法就好, 具体例子可以在slide中找到

最后, 这个方法切分出来的motif的传导率和最优解的关系是
$$
\phi_M(S) \le 4 \sqrt{\phi_M^*}
$$
![cs224w-lec5-18](/images/cs224w/cs224w-lec5-18.png)



> 课程主页是[http://web.stanford.edu/class/cs224w/][1], 视频在b站可以找到[CS224W 2019 Fall][11], youtube也有



[1]: https://www.zhihu.com/question/38804190/answer/100286320
[2]: 【其实贼简单】拉普拉斯算子和拉普拉斯矩阵
[3]: https://www.cnblogs.com/shiyublog/p/9785342.html
[4]:  https://www.zhihu.com/question/281266319/answer/660956134
[5]: https://zhuanlan.zhihu.com/p/67522582?utm_source=wechat_session
[6]: https://seanwangjs.github.io/2017/11/27/rayleigh-quotient-maximum.html
[7]: https://ccjou.wordpress.com/2015/11/23/%e6%af%8f%e9%80%b1%e5%95%8f%e9%a1%8c-november-23-2015/
[8]: https://www.cnblogs.com/pinard/p/6244265.html
[9]: https://www.cnblogs.com/xingshansi/p/6702188.html

[10]: http://web.stanford.edu/class/cs224w/
[11]: https://www.bilibili.com/video/av90106649?p=5
