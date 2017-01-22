---
title: 近似推断
layout: post
share: false
---
<!-- % 623 -->


许多概率模型是很难训练的，其原因是很难进行推断。
在深度学习中，我们通常有一个可见变量$\Vv$的集合和一个潜变量 $\Vh$的集合。
推断的挑战往往在于计算$p(\Vh\mid\Vv)$或者计算在分布$p(\Vh\mid\Vv)$下期望的困难性。
而这样的操作在一些任务比如最大似然学习中往往又是必需的。
<!-- % 623 -->

许多诸如受限玻尔兹曼机和概率PCA这样的仅仅含有一层隐藏层的简单图模型的定义，往往使得推断操作如计算$p(\Vh\mid\Vv)$或者计算分布$p(\Vh\mid\Vv)$下的期望是非常容易的。
不幸的是，大多数的具有多层潜变量的图模型的后验分布都很难处理。
对于这些模型精确的推断算法需要指数量级的运行时间。
即使一些只有单层的模型，如稀疏编码，也存在着这样的问题。
<!-- % 623 -->


在本章中，我们介绍了几个基本的技巧，用来解决难以处理的推断问题。
稍后，在\chap?中，我们还将描述如何将这些技巧应用到训练其他方法难以奏效的概率模型中，如深度信念网络，深度玻尔兹曼机。
<!-- % 623 -->


在深度学习中难以处理的推断问题通常源于结构化图模型中潜变量之间的相互作用。
详见\fig?的几个例子。
这些相互作用可能是无向模型的直接作用，也可能是有向模型中同一个可见变量的共同祖先之间的explaining away作用。
<!-- % 623 end -->


\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
	\centerline{\includegraphics[width=0.8\textwidth]{Chapter19/figures/intractable_graphs}}
\fi
	\caption{深度学习中难以处理的推断问题通常是由于结构化图模型中潜变量的相互作用。
	这些相互作用产生于当V-结构的子节点是可观察的时候一个潜变量与另一个潜变量或者更长的激活路径相连。
	（左）一个隐藏单元相互连接的半受限波尔兹曼机{cite?}。由于存在大量的潜变量的团，潜变量的直接连接使得后验分布难以处理。
	（中）一个深度玻尔兹曼机，被分层从而使得不存在层内连接，由于层之间的连接其后验分布仍然难以处理。
	（右）当可见变量是可以观察时这个有向模型的潜变量之间存在相互作用，因为每两个潜变量都是coparent。
	即使拥有上图中的某一种结构，一些概率模型依然能够获得易于处理的后验分布。
	如果我们选择条件概率分布来引入相对于图结构描述的额外的独立性这种情况也是可能出现的。
	举个例子，概率PCA的图结构如右图所示，然而由于其条件分布的特殊性质（带有相互正交的基向量的线性高斯条件分布）依然能够进行简单的推断。}
\end{figure}



# 推断是一个优化问题

<!-- % 624 -->

许多难以利用观察值进行精确推断的问题往往可以描述为一个优化问题。
通过近似这样一个潜在的优化问题，我们往往可以推导出近似推断算法。
<!-- % 624 -->


为了构造这样一个优化问题，假设我们有一个包含可见变量$\Vv$和潜变量 $\Vh$的概率模型。
我们希望计算观察数据的概率对数$\log p(\Vv;\Vtheta)$。
有时候如果边缘化消去$\Vh$的操作很费时的话，我们通常很难计算$\log p(\Vv;\Vtheta)$。
作为替代，我们可以计算一个$\log p(\Vv;\Vtheta)$的下界$\CalL(\Vv,{\Vtheta},q)$。
这个下界叫做证据下界。
这个下界的另一个常用名字是负变分自由能。
具体地，这个证据下界是这样定义的：

\begin{align}
\CalL(\Vv,{\Vtheta},q) = \log p(\Vv;{\Vtheta}) - D_{\text{KL}}(q(\Vh\mid\Vv) \Vert p(\Vh\mid\Vv;{\Vtheta})).
\end{align}
在这里 $q$是关于$\Vh$的一个任意概率分布。
<!-- % 625 -->


因为$\log p(\Vv)$和$\CalL(\Vv,{\Vtheta},q)$之间的距离是由KL散度来衡量的。
因为KL散度总是非负的，我们可以发现$\CalL$小于等于所求的概率的对数。
当且仅当$q$完全相等于$p(\Vh\mid\Vv)$时取到等号。
<!-- % 625 -->


令人吃惊的是，对某些分布$q$，$\CalL$可以被化得更简单。
通过简单的代数运算我们可以把$\CalL$重写成一个更加简单的形式：

\begin{align}
\CalL(\Vv,{\Vtheta},q) = & \log p(\Vv;{\Vtheta})- D_{\text{KL}}(q(\Vh\mid\Vv)\Vert p(\Vh\mid\Vv;{\Vtheta})) \\
= & \log p(\Vv;{\Vtheta}) - \SetE_{\RVh\sim q}\log \frac{q(\Vh\mid\Vv)}{p(\Vh\mid\Vv)} \\
= & \log p(\Vv;{\Vtheta}) -  \SetE_{\RVh\sim q} \log \frac{q(\Vh\mid\Vv) }{ \frac{p(\Vh,\Vv;{\Vtheta})}{p(\Vv; {\Vtheta})} } \\
= & \log p(\Vv; {\Vtheta}) -  \SetE_{\RVh\sim q} [\log q(\Vh\mid\Vv) - \log {p(\Vh,\Vv; {\Vtheta})} + \log {p(\Vv; {\Vtheta})} ]\\
= & - \SetE_{\RVh\sim q}[\log q(\Vh\mid\Vv) - \log {p(\Vh,\Vv;{\Vtheta})}].
\end{align}
<!-- % 625 -->


这也给出了证据下界的标准定义：
\begin{align}
\CalL(\Vv,{\Vtheta},q) = \SetE_{\RVh\sim q}[\log p(\Vh , \Vv)] + H(q).
\end{align}


对于一个较好的选择$q$来说，$\CalL$是容易计算的。
对任意选择$q$来说，$\CalL$提供了一个似然函数的下界。
越好的近似$q$的分布$q(\Vh\mid\Vv)$得到的下界就越紧，换句话说，就是与$\log p(\Vv)$更加接近。
当$q(\Vh\mid\Vv) = p(\Vh\mid\Vv)$时，这个近似是完美的，也意味着$\CalL(\Vv,{\Vtheta},q) = \log {p(\Vv;{\Vtheta})} $。
<!-- % 625 -->


因此我们可以将推断问题看做是找一个分布$q$使得$\CalL$最大的过程。
精确的推断能够通过搜索一组包含分布$p(\Vh\mid\Vv)$的$q$函数族，完美地最大化$\CalL$。
在本章中，我们将会讲到如何通过近似优化来找$q$的方法来推导出不同形式的的近似推断。
我们可以通过限定分布$q$的形式或者使用并不彻底的优化方法来使得优化的过程更加高效，但是优化的结果是不完美的，因为只能显著地提升$\CalL$而无法彻底地最大化$\CalL$。
<!-- % 625  end -->


无论我们选择什么样的$q$，$\CalL$始终是一个下界。
我们可以通过选择一个更简单抑或更复杂的计算过程来得到对应的更松或者更紧的下界。
通过一个不彻底的优化过程或者将$q$分布做很强的限定（并且使用一个彻底的优化过程）我们可以获得一个很差的$q$，尽管计算开销是极小的。
<!-- % 626 head -->



# 期望最大化

<!-- % 626 -->

我们介绍的第一个最大化下界$\CalL$的算法是期望最大化算法。
在潜变量模型中，这是一个非常热门的训练算法。
在这里我们描述{emview}所提出的EM算法。
不像大多数我们在本章中介绍的其他算法一样，EM并不是一个近似推断算法，但是是一种能够学到近似后验的算法。
<!-- % 626 -->


EM算法包含了交替进行两步运算直到收敛的过程：

+ E步: 令${\Vtheta^{(0)}}$表示在这一步开始时参数的初始值。
	令$q(\Vh^{(i)}\mid \Vv) = p(\Vh^{(i)}\mid\Vv^{(i)};\Vtheta^{(0)})$对任何我们想要训练的(对所有的或者minibatch数据均成立)索引为$i$的训练样本$\Vv^{(i)}$。
	通过这个定义，我们认为$q$在当前的参数$\Vtheta^{(0)}$下定义。
	如果我们改变$\Vtheta$，那么$p(\Vh\mid\Vv;\Vtheta)$将会相应的变化，但是$q(\Vh\mid\Vv)$还是不变并且等于$p(\Vh\mid\Vv;\Vtheta^{(0)})$。
+ M步：使用选择的优化算法完全地或者部分地最大化关于$\Vtheta$的
	\begin{align}
	\sum_i \CalL(\Vv^{(i)},\Vtheta,q).
	\end{align}

<!-- % 626   -->


这可以被看做通过坐标上升算法来最大化$\CalL$。
在第一步中，我们更新$q$来最大化$\CalL$，而另一步中，我们更新$\Vtheta$来最大化$\CalL$。
<!-- % 626 -->


基于潜变量模型的随机梯度上升可以被看做是一个EM算法的特例，其中M步包括了单次梯度的操作。
EM算法的其他变种可以实现多次梯度操作。
对一些模型族来说，M步甚至可以通过推出理论解直接完成，不同于其他方法，在给定当前$q$的情况下直接求出最优解。
<!-- % 626 end -->


即使E步采用的是精确推断，我们仍然可以将EM算法视作是某种程度上的近似推断。
具体地说，M步假设了一个$q$分布可以被所有的$\Vtheta$分享。
当M步越来越远离E步中的$\Vtheta^{(0)}$的时候，这将会导致$\CalL$和真实的$\log p(\Vv)$的差距。
幸运的事，当下一个循环的时候，E步把这种差距又降到了$0$。
<!-- % 627 head -->



EM算法包含了一些不同的解释。
首先，学习过程的一个基本思路就是，我们通过更新模型参数来提高整个数据集的似然，其中缺失变量的值是通过后验分布来估计的。
这种特定的性质并不仅仅适用于EM算法。
比如说，使用梯度下降来最大化似然函数的对数这种方法也利用了相同的性质。
计算对数似然函数的梯度需要对隐藏单元的后验分布来求期望。 
EM算法另一个关键的性质是当我们移动到另一个$\Vtheta$时候，我们仍然可以使用旧的$q$。
在传统机器学习中，这种特有的性质在推导M步更新时候得到了广泛的应用。
在深度学习中，大多数模型太过于复杂以致于在M步中很难得到一个最优解。
所以EM算法的第二个特质较少被使用。
<!-- % 627 -->



# 最大后验推断和稀疏编码

<!-- % 627 -->


我们通常使用推断这个术语来指代给定一些其他变量的情况下计算某些变量的概率分布的过程。
当训练带有潜变量的概率模型时，我们通常关注于计算$p(\Vh\mid\Vv)$。
在推断中另一个选择是计算一个最有可能的潜变量的值来代替在其完整分布上的抽样。
在潜变量模型中，这意味着计算
\begin{align}
\Vh^* = \underset{\Vh}{\arg\max} \ \  p(\Vh\mid\Vv).
\end{align}
这被称作是最大后验推断，简称MAP推断。
<!-- % 627   -->



最大后验推断并不是一种近似推断，它只是精确地计算了最有可能的一个$\Vh^*$。
然而，如果我们希望能够最大化$\CalL(\Vv,\Vh,q)$，那么我们可以把最大后验推断看成是输出一个$q$的学习过程。%是很有帮助的。
在这种情况下，我们可以将最大后验推断看成是近似推断，因为它并不能提供一个最优的$q$。
<!-- % 627 end -->



我们回过头来看看\sec?中所描述的精确推断，它指的是关于一个在无限制的概率分布族中的$q$分布使用精确的优化算法来最大化
\begin{align}
\CalL(\Vv,{\Vtheta},q)
 = \SetE_{\RVh\sim q}[\log p(\Vh , \Vv)] + H(q).
\end{align}
我们通过限定$q$分布属于某个分布族，能够使得最大后验推断成为一种形式的近似推断。
具体的说，我们令$q$分布满足一个Dirac分布：
\begin{align}
q(\Vh\mid\Vv) = \delta(\Vh - {\Vmu}).
\end{align}
这也意味着我们可以通过$\Vmu$来完全控制$q$。
通过将$\CalL$中不随$\Vmu$变化的项丢弃，剩下的我们遇到的是一个优化问题：
\begin{align}
\Vmu^*  =  \underset{\Vmu}{\arg\max}\ \log p(\Vh = \Vmu,\Vv).
\end{align}
这等价于最大后验推断问题
\begin{align}
\Vh^* = \underset{\Vh}{\arg\max}\  p(\Vh\mid\Vv).
\end{align}
<!-- % 628 -->




因此我们能够解释一种类似于EM算法的学习算法，其中我们轮流迭代两步，一步是用最大后验推断估计出$\Vh^*$，另一步是更新$\Vtheta$来增大$\log p(\Vh^*,\Vv)$。
从EM算法角度看，这也是一种形式的对$\CalL$的坐标上升，EM算法的坐标上升中，交替迭代的时候通过推断来优化$\CalL$关于$q$以及通过参数更新来优化$\CalL$关于$\Vtheta$。
整体上说，这个算法的正确性可以得到保证，因为$\CalL$是$\log p(\Vv)$的下界。
在MAP推断中，这个保证是无效的，因为这个界会无限的松，由于Dirac分布的熵的微分趋近于负无穷。
然而，人为加入一些$\Vmu$的噪声会使得这个界又有了意义。
<!-- % 628 -->



最大后验推断作为特征提取器以及一种学习机制被广泛的应用在了深度学习中。
在稀疏编码模型中，它起到了关键作用。
<!-- % 628 -->


我们回过头来看\sec?中的稀疏编码，稀疏编码是一种在隐藏单元上加上了鼓励稀疏的先验知识的线性因子模型。
一个常用的选择是可分解的拉普拉斯先验，表示为
\begin{align}
	p(h_i) = \frac{\lambda}{2}  \exp(-\lambda \vert h_i \vert).
\end{align}
可见的节点是由一个线性变化加上噪音生成的\footnote{此处似乎有笔误，$\Vx$应为$\Vv$}:
\begin{align}
	p(\Vx\mid\Vh) = \CalN(\Vv;{\MW}\Vh + \Vb,\beta^{-1}{\MI}).
\end{align}
<!-- % 628 end -->




计算或者表达$p(\Vh\mid\Vv)$太过困难。
每一对$h_i$， $h_j$变量都是$\Vv$的母节点。
这也意味着当$\Vv$可观察时，图模型包含了连接$h_i$和$h_j$的一条活跃路径。
因此$p(\Vh \mid\Vv)$中所有的隐藏单元都包含在了一个巨大的团中。
如果模型是高斯，那么这些关系可以通过协方差矩阵来高效地建模。
然而稀疏型先验使得模型并不是高斯。
<!-- % 629 head -->



$p(\Vx\mid\Vh)$的复杂性导致了似然函数的对数及其梯度也很难得到。
因此我们不能使用精确的最大似然学习来进行学习。
取而代之的是，我们通过MAP推断以及最大化由以$\Vh$为中心的Dirac分布所定义而成的ELBO来学习模型参数。
<!-- % 629 -->


如果我们将训练集中所有的$\Vh$向量拼在一起并且记为$\MH$，并将所有的$\Vv$向量拼起来组成矩阵$\MV$，那么稀疏编码问题意味着最小化
\begin{align}
	J(\MH,\MW) = \sum_{i,j}^{}\vert H_{i,j}\vert + \sum_{i,j}^{}\Big(\MV - \MH \MW^{\top}\Big)^2_{i,j}.
\end{align}
为了避免如极端小的$\MH$和极端大的$\MW$这样的病态的解，许多稀疏编码的应用包含了权值衰减或者对$\MH$列范数的限制。
<!-- % 629 -->


我们可以通过交替迭代最小化$J$分别关于$\MH$和$\MW$的方式来最小化$J$。
两个子问题都是凸的。
事实上，关于$\MW$的最小化问题就是一个线性回归问题。
然而关于这两个变量同时最小化$J$的问题并不是凸的。
<!-- % 629 -->


关于$\MH$的最小化问题需要某些特别设计的算法诸如特征符号搜索方法{cite?}。
<!-- % 629 -->



# 变分推断和学习

<!-- % 629 -->


我们已经说明过了为什么证据下界 $\CalL(\Vv,\Vtheta,q)$是$\log  p(\Vv;\Vtheta)$的一个下界， 如何将推断看做是 关于$q$分布最大化$\CalL$ 的过程以及如何将学习看做是关于参数$\Vtheta$最大化$\CalL$的过程。
我们也讲到了EM算法在给定了$q$分布的条件下进行学习，而基于MAP推断的学习算法则是学习一个$p(\Vh \mid \Vv)$的点估计而非推断整个完整的分布。
在这里我们介绍一些变分学习中更加通用的算法。
<!-- % 629 -->


变分学习的核心思想就是我们通过选择给定有约束的分布族中的一个$q$分布来最大化$\CalL$。
选择这个分布族的时候应该考虑到计算$\SetE_q \log p(\Vh,\Vv)$的简单性。
一个典型的方法就是添加一些假设诸如$q$分布可以分解。
<!-- % 630 head -->


一种常用的变分学习的方法是加入一些限制使得$q$是一个可以分解的分布：
\begin{align}
	q(\Vh\mid\Vv) = \prod_{i}^{}q(h_i \mid \Vv).
\end{align}
这被叫做是均匀场方法。
一般来说，我们可以通过选择$q$分布的形式来选择任何图模型的结构，通过选择变量之间的相互作用来决定近似程度的大小。
这种完全通用的图模型方法叫做结构化变分推断 {cite?}。
<!-- % 630  -->


变分方法的优点是我们不需要为分布$q$设定一个特定的参数化的形式。
我们设定它如何分解，之后通过解决优化问题来找出在这些分解限制下的最优的概率分布。
对离散型潜变量来说，这意味着我们使用了传统的优化技巧来优化描述$q$分布的有限个数的变量。
对连续型潜变量来说，这意味着我们使用了一个叫做变分法的数学分支来解决对一个空间上函数的优化问题。
然后决定哪一个函数来表示$q$分布。
变分法是"变分学习"或者"变分推断"这些名字的来历，尽管当潜变量是离散的时候变分法并没有用武之地。
当遇到连续型潜变量的时候，变分法是一种很有用的工具，只需要设定分布$q$如何分解，而不需要过多的人工选择模型，比如尝试着设计一个特定的能够精确的近似原后验分布的$q$分布。
<!-- % 630  -->


因为$\CalL(\Vv,\Vtheta,q)$定义成$\log p(\Vv;\Vtheta) - D_{\text{KL}} (q(\Vh\mid\Vv) \Vert  p(\Vh\mid\Vv;\Vtheta) )$，我们可以认为关于$q$最大化$\CalL$的问题等价于最小化$D_{\text{KL}}(q(\Vh\mid\Vv)\Vert p(\Vh\mid\Vv))$。
在这种情况下，我们要用$q$来拟合$p$。
然而，我们并不是直接拟合一个近似，而是处理一个KL散度的问题。
当我们使用最大似然学习来将数据拟合到模型的时候，我们最小化$D_{\text{KL}}(p_{\text{data}} \Vert p_{\text{model}})$。
如同\fig?中所示，这意味着最大似然促进模型在每一个数据达到更高概率的地方达到更高的概率，而基于优化的推断则促进了$q$在每一个真实后验分布概率较低的地方概率较小。
这两种基于KL散度的方法都有各自的优点与缺点。
选择哪一种方法取决于在具体应用中哪一种性质更受偏好。
在基于优化的推断问题中，从计算角度考虑，我们选择使用$D_{\text{KL}}(q(\Vh\mid\Vv)\Vert p(\Vh\mid\Vv))$。
具体的说，计算$D_{\text{KL}}(q(\Vh\mid\Vv)\Vert p(\Vh\mid\Vv))$涉及到了计算$q$分布下的期望，所以通过将分布$q$设计的较为简单，我们可以简化求所需要的期望的计算过程。
另一个方向的KL散度需要计算真实后验分布下的期望。
因为真实后验分布的形式是由模型的选择决定的，我们不能设计出一种能够精确计算$D_{\text{KL}}(p(\Vh\mid\Vv) \Vert q(\Vh\mid\Vv))$的开销较小的方法。
<!-- % 631 head -->





## 离散型潜变量

<!-- % 631  19.4.1 -->

关于离散型潜变量的变分推断相对来说更为直接。
我们定义一个分布$q$，通常$q$的每个因子都由一些离散状态的可查询表格定义。
在最简单的情况中，$\Vh$是二元的并且我们做了均匀场的假定，$q$可以根据每一个$h_i$分解。
在这种情况下，我们可以用一个向量$\hat{\Vh}$来参数化$q$分布，$\hat{\Vh}$的每一个元素都代表一个概率，即$q(h_i = 1\mid \Vv) = \hat{h}_i$。
<!-- % 631 -->


在确定了如何表示$q$以后，我们只需要优化它的参数。
在离散型潜变量模型中，这是一个标准的优化问题。
基本上$q$的选择可以通过任何的优化算法解决，比如说梯度下降。
<!-- % 631 -->


这个优化问题是很高效的因为它在许多学习算法的内循环中出现。
为了追求速度，我们通常使用特殊设计的优化算法。
这些算法通常能够在极少的循环内解决一些小而简单的问题。
一个常见的选择是使用不动点方程，换句话说，就是解关于$\hat{h}_i$的方程
\begin{align}
	\frac{\partial}{\partial \hat{h}_i}\CalL = 0.
\end{align}
我们反复地更新$\hat{\Vh}$不同的元素直到收敛条件满足。
<!-- % 631 -->


为了具体化这些描述，我们接下来会讲如何将变分推断应用到二值稀疏编码模型（这里我们所描述的模型是{henniges2010binary}提出的，但是我们采用了传统且通用的均匀场方法，而原文作者采用了一种特殊设计的算法）中。
推导过程在数学上非常详细，为希望完全了解变分推断细节的读者所准备。
而对于并不计划推导或者实现变分学习算法的读者来说，可以完全跳过，直接阅读下一节，这并不会导致新的高级概念的遗漏。
那些从事过二值稀疏编码研究的读者可以重新看一下\sec?中的一些经常在概率模型中出现的有用的函数性质。
我们在推导过程中随意地使用了这些性质，并没有特别强调它们。
<!-- % 632  head -->


在二值稀疏编码模型中，输入$\Vv\in\SetR^n$，是由模型通过添加高斯噪音到$m$个或有或无的成分。
每一个成分可以是开或者关的通过对应的隐藏单元 $\Vh \in\{0,1\}^m$:
\begin{align}
	p(h_i = 1) = \sigma(b_i),
\end{align}
\begin{align}
	p(\Vv\mid\Vh) = \CalN(\Vv;\MW \Vh,{\Vbeta}^{-1}),
\end{align}
其中$\Vb$是一个可以学习的偏置的集合，$\MW$是一个可以学习的权值矩阵，${\Vbeta}$是一个可以学习的对角精度矩阵。
<!-- % 632 -->


使用最大似然学习来训练这样一个模型需要对参数进行求导。
我们考虑对其中一个偏置进行求导的过程：
\begin{align}
		& \frac{\partial}{\partial b_i} \log p(\Vv) \\
		= &  \frac{\frac{\partial}{\partial b_i} p(\Vv)}{p(\Vv)}\\
		= & \frac{\frac{\partial}{\partial b_i} \sum_{\Vh}^{} p(\Vh,\Vv)}{p(\Vv)}\\
		= &  \frac{\frac{\partial}{\partial b_i} \sum_{\Vh}^{} p(\Vh) p(\Vv\mid \Vh)  }{p(\Vv)}\\
		= &  \frac{ \sum_{\Vh}^{}  p(\Vv\mid \Vh) \frac{\partial}{\partial b_i} p(\Vh) }{p(\Vv)}\\
		= &  \sum_{\Vh}^{}  p(\Vh\mid \Vv) \frac{  \frac{\partial}{\partial b_i} p(\Vh) }{p(\Vh)}\\
		= & \SetE_{\RVh\sim p(\Vh\mid\Vv)} \frac{\partial}{\partial b_i}\log p(\Vh).
\end{align}
<!-- % 632 end -->

这需要计算$p(\Vh\mid\Vv)$下的期望。
不幸的是，$p(\Vh\mid\Vv)$是一个很复杂的分布。
$p(\Vh,\Vv)$和$p(\Vh\mid\Vv)$的图结构见\fig?。
隐藏单元的后验分布对应的是完全图，所以相对于暴力算法，消元算法并不能有助于提高计算所需要的期望的效率。
<!-- % 633 head -->


\begin{figure}[!htb]
\ifOpenSource
\centerline{\includegraphics{figure.pdf}}
\else
	\centerline{\includegraphics{Chapter19/figures/bsc}}
\fi
	\caption{包含四个隐藏单元的二值稀疏编码的图结构。（左）$p(\Vh,\Vv)$的图结构。要注意边是有向的，每两个隐藏单元都是每个可见单元的coparent。（右）$p(\Vh,\Vv)$的图结构。为了解释coparent之间的活跃路径，后验分布所有隐藏单元之间都有边。}
\end{figure}



取而代之的是，我们可以应用变分推断和变分学习来解决这个难点。
<!-- % 633 -->


我们可以做一个均匀场的假设：
\begin{align}
	q(\Vh\mid\Vv) = \prod_{i}^{}q(h_i\mid\Vv).
\end{align}
<!-- % 633 -->


二值稀疏编码中的潜变量是二值的，所以为了表示可分解的$q$我们假设模型为$m$个Bernoulli分布 $q(h_i\mid\Vv)$。
表示Bernoulli分布的一种常见的方法是一个概率向量$\hat{\Vh}$，满足$q(h_i\mid\Vv) = \hat{h}_i$。
为了避免计算中的误差，比如说计算$\log \hat{h}_i$的时候，我们对$\hat{h}_i$添加一个约束，即$\hat{h}_i$不等于0或者1。
<!-- % 633  -->


我们将会看到变分推断方程理论上永远不会赋予$\hat{h}_i\ $ $0$或者$1$。
然而在软件实现过程中，机器的舍入误差会导致$0$或者$1$的值。
在二值稀疏编码的实现中，我们希望使用一个没有限制的变分参数向量$\Vz$以及
通过关系$\hat{\Vh} = \sigma(\Vz)$来获得$\Vh$。
因此我们可以放心的在计算机上计算$\log \hat{h}_i$通过使用关系式$\log \sigma(z_i) = -\zeta(-z_i)$来建立sigmoid和softplus的关系。
<!-- % 633 -->


在开始二值稀疏编码模型的推导时，我们首先说明了均匀场的近似可以使得学习的过程更加简单。
<!-- % 633 end   -->


证据下界可以表示为
\begin{align}
& \CalL(\Vv,\Vtheta,q)\\
 = & \SetE_{\RVh\sim q}[\log p(\Vh,\Vv)] + H(q)\\
 = & \SetE_{\RVh\sim q}[\log p(\Vh) + \log p(\Vv\mid\Vh) - \log q(\Vh\mid\Vv)]\\
= & \SetE_{\RVh\sim q}\Big[\sum_{i=1}^{m}\log p(h_i) + \sum_{i=1}^{n} \log p(v_i\mid\Vh) - \sum_{i=1}^{m}\log q(h_i\mid\Vv)\Big]\\
= &  \sum_{i=1}^{m}\Big[\hat{h}_i(\log \sigma(b_i) - \log \hat{h}_i) + (1 - \hat{h}_i)(\log \sigma(-b_i) - \log (1-\hat{h}_i))\Big] \\
& +  \SetE_{\RVh\sim q} \Bigg[ \sum_{i=1}^{n}\log \sqrt{\frac{\beta_i}{2\pi}}\exp(-\frac{\beta_i}{2}(v_i - \MW_{i,:}\Vh)^2)\Bigg] \\
= &  \sum_{i=1}^{m}\Big[\hat{h}_i(\log \sigma(b_i) - \log \hat{h}_i) + (1 - \hat{h}_i)(\log \sigma(-b_i) - \log (1-\hat{h}_i))\Big] \\
& + \frac{1}{2} \sum_{i=1}^{n} \Bigg[ \log {\frac{\beta_i}{2\pi}} - \beta_i \Bigg(v_i^2 - 2 v_i \MW_{i,:}\hat{\Vh} + \sum_{j}\Big[W^2_{i,j}\hat{h}_j + \sum_{k \neq j}W_{i,j}W_{i,k}\hat{h}_j\hat{h}_k \Big] \Bigg) \Bigg]. 
\end{align}
<!-- % 634 head  -->


尽管这些方程从美学观点来看有些不尽如人意。
他们展示了$\CalL$可以被表示为少量简单的代数运算。
因此证据下界是易于处理的。
我们可以把$\CalL$看作是难以处理的对数似然函数的一个替代。
<!-- % 634  head  -->


原则上说，我们可以使用关于$\Vv$和$\Vh$的梯度下降。
这会成为一个完美的组合的推断学习算法。
但是，由于两个原因，我们往往不这么做。
第一点，对每一个$\Vv$我们需要存储$\hat{\Vh}$。
我们通常更加偏向于那些不需要为每一个样本都准备内存的算法。
如果我们需要为每一个样本都存储一个动态更新的向量，使得算法很难处理好几亿的样本。
第二个原因就是为了能够识别$\Vv$的内容，我们希望能够有能力快速提取特征$\hat{\Vh}$。
在实际应用下，我们需要在有限时间内计算出$\hat{\Vh}$。
<!-- % 634 -->


由于以上两个原因，我们通常不会采用梯度下降来计算均匀场的参数$\hat{\Vh}$。
取而代之的是，我们使用不动点方程来快速估计他们。
<!-- % 634 -->


不动点方程的核心思想是我们寻找一个关于$\Vh$的局部最大值，满足$\nabla_{\Vh}\CalL(\Vv,\Vtheta,\hat{\Vh}) = 0$。
我们无法同时高效地计算所有的$\hat{\Vh}$的元素。
然而，我们可以解决单个变量的问题：
\begin{align}
\frac{\partial}{\partial \hat{h}_i} \CalL(\Vv,\Vtheta,\hat{\Vh}) = 0 .
\end{align}
<!-- % 634 end   -->


我们可以迭代地将这个解应用到$i = 1,\ldots,m$，然后重复这个循环直到我们满足了收敛标准。
常见的收敛标准包含了当整个循环所改进的$\CalL$不超过预设的容忍的量时停止，或者是循环中改变的$\hat{\Vh}$不超过某个值的时候停止。
<!-- % 635  head -->

在很多不同的模型中，迭代的均匀场不动点方程是一种能够提供快速变分推断的通用的算法。
为了使它更加具体化，我们详细地讲一下如何推导出二值稀疏编码模型的更新过程。
<!-- % 635 -->


首先，我们给出了对$\hat{h}_i$的导数的表达式。
为了得到这个表达式，我们将方程~\eq?代入到方程~\eq?的左边：
\begin{align}
& \frac{\partial}{\partial \hat{h}_i} \CalL (\Vv,\Vtheta,\hat{\Vh})    \\
= & \frac{\partial}{\partial \hat{h}_i} \Bigg[\sum_{j=1}^{m}\Big[\hat{h}_j (\log \sigma(b_j) - \log \hat{h}_j) + (1 - \hat{h}_j)(\log \sigma(-b_j) - \log (1-\hat{h}_j))\Big] \\
& + \frac{1}{2} \sum_{j=1}^{n} \Bigg[ \log {\frac{\beta_j}{2\pi}} - \beta_j \Bigg(v_j^2 - 2 v_j \MW_{j,:}\hat{\Vh} + \sum_{k}\Bigg[W^2_{j,k}\hat{h}_k + \sum_{l \neq k}W_{j,k}W_{j,l}\hat{h}_k\hat{h}_l \Bigg] \Bigg) \Bigg]\Bigg] \\
= & \log \sigma(b_i) - \log \hat{h}_i - 1 + \log (1 - \hat{h}_i ) + 1 - \log \sigma (- b_i ) \\
 & + \sum_{j=1}^{n} \Bigg[\beta_j \Bigg(v_j W_{j,i} - \frac{1}{2} W_{j,i}^2 - \sum_{k \neq i} \MW_{j,k}\MW_{j,i} \hat{h}_k\Bigg) \Bigg]\\
 = & b_i - \log \hat{h}_i + \log (1 - \hat{h}_i ) + \Vv^{\top} {\Vbeta} \MW_{:,i} - \frac{1}{2} \MW_{:,i}^{\top} {\Vbeta}\MW_{:,i} -\sum_{j\neq i}\MW^{\top}_{:,j}{\Vbeta}\MW_{:,i}\hat{h}_j.
\end{align}
<!-- % 635   -->

为了应用固定点更新的推断规则，我们通过令方程~\eq?等于$0$来解$\hat{h}_i$：

\begin{align}
\hat{h}_i = \sigma\Bigg(b_i + \Vv^{\top} {\Vbeta} \MW_{:,i} - \frac{1}{2} \MW_{:,i}^{\top} {\Vbeta} \MW_{:,i} - \sum_{j \neq i }  \MW_{:,j}^{\top} {\Vbeta}  \MW_{:,i} \hat{h}_j \Bigg).
\end{align}
<!-- % 635 end -->

此时，我们可以发现图模型中循环神经网络和推断之间存在着紧密的联系。
具体的说，均匀场不动点方程定义了一个循环神经网络。
这个神经网络的任务就是完成推断。
我们已经从模型描述角度介绍了如何推导这个网络，但是直接训练这个推断网络也是可行的。
有关这种思路的一些想法在\chap?中有所描述。
<!-- % 636  head  -->


在二值稀疏编码模型中，我们可以发现\eqn?中描述的循环网络包含了根据相邻的变化着的隐藏单元值来反复更新当前隐藏单元的操作。
输入层通常给隐藏单元发送一个固定的信息$\Vv^{\top}\beta\MW$，然而隐藏单元不断地更新互相传送的信息。
具体的说，当$\hat{h}_i$和$\hat{h}_j$两个单元的权重向量对准时，他们会产生相互抑制。
这也是一种形式的竞争－－－两个共同理解输入的隐藏单元之间，只有一个解释的更好的才能继续保持活跃。
在二值稀疏编码的后验分布中，均匀场近似为了捕获到更多的explaining away作用，产生了这种竞争。
事实上，explaining away效应会导致一个多峰值的后验分布，所以我们如果从后验分布中采样，一些样本只有一个结点是活跃的，其他的样本在其他的结点活跃，只有很少的样本能够所有的结点都处于活跃状态。
不幸的是，explaining away作用无法通过均匀场中可分解的$q$分布来建模，因此建模时均匀场近似只能选择一个峰值。
这个现象的一个例子可以参考\fig?。
<!-- % 636 -->




我们将方程~\eq?重写成等价的形式来揭示一些深层的含义：
\begin{align}
\hat{h}_i = \sigma\Bigg(b_i + \Big(\Vv - \sum_{j\neq i} \MW_{:,j}\hat{h}_j\Big)^{\top} {\Vbeta}\MW_{:,i} - \frac{1}{2} \MW_{:,i}^{\top} {\Vbeta} \MW_{:,i}\Bigg). 
\end{align}
在这种新的形式中，我们可以将每一步的输入看成$\Vv - \sum_{j\neq i} \MW_{:,j}\hat{h}_j$而不是$\Vv$。
因此，我们可以把第$i$个单元视作给定其他单元的编码时编码$\Vv$中的剩余误差。
由此我们可以将稀疏编码视作是一个迭代的自编码器，反复的将输入的编码解码，试图逐渐减小重构误差。
<!-- % 636 -->


在这个例子中，我们已经推导出了每一次更新单个结点的更新规则。
接下来我们考虑如何同时更新许多结点。
某些图模型，比如DBM，我们可以同时更新$\hat{\Vh}$中的许多元素。
不幸的是，二值稀疏编码并不适用这种块更新。
取而代之的是，我们使用一种称为是衰减的启发式技巧来实现块更新。
在衰减方法中，对$\hat{\Vh}$中的每一个元素我们都可以解出最优值，然后对于所有的值都在这个方向上移动一小步。
这个方法不能保证每一步都能增加$\CalL$，但是对于许多模型来说却很有效。
关于在信息传输算法中如何选择同步程度以及使用衰减策略可以参考{koller-book2009} 。
 % 637 head 





## 变分法

<!-- % p 637 head   19.4.2  -->

在继续描述变分学习之前，我们有必要简单地介绍一种变分学习中的重要的数学工具：变分法。
<!-- % p 637 -->


许多机器学习的技巧是基于寻找一个输入向量$\Vtheta\in\SetR^n$来最小化函数$J(\Vtheta)$，使得它取到最小值。
这个步骤可以利用多元微积分以及线性代数的知识找到满足$\nabla_{\Vtheta} J(\Vtheta) = 0$的临界点来完成。
在某些情况下，我们希望能够找出一个最优的函数$f(\Vx)$，比如当我们希望找到一些随机变量的概率密度函数的时候。
变分法能够让我们完成这个目标。
<!-- % p 637 -->



$f$函数的函数被称为是泛函 $J[f]$。
正如我们许多情况下对一个函数求关于以向量为变量的偏导数一样，我们可以使用泛函导数，即对任意一个给定的$\Vx$，对一个泛函 $J[f]$求关于函数$f(\Vx)$的导数，这也被称为变分微分。
泛函 $J$的关于函数$f$在点$\Vx$处的泛函导数被记作$\frac{\delta}{\delta f(x)}J$。
<!-- % p 637 -->



完整正式的泛函导数的推导不在本书的范围之内。
为了满足我们的目标，讲述可微分函数$f(\Vx)$以及带有连续导数的可微分函数$g(y,\Vx)$就足够了：
\begin{align}
	\frac{\delta}{\delta f(\Vx)} \int g(f(\Vx),\Vx)d\Vx = \frac{\partial}{\partial y}g(f(\Vx),\Vx).
\end{align}
为了使上述的关系式更加的形象，我们可以把$f(\Vx)$看做是一个有着无穷多元素的向量。
在这里（看做是一个不完全的介绍），这种关系式中描述的泛函导数和向量$\Vtheta\in\SetR^n$的导数相同：
\begin{align}
	\frac{\partial}{\partial \theta_i}\sum_{j}^{}g(\theta_j,j) = \frac{\partial}{\partial \theta_i}g(\theta_i,i).
\end{align}
在其他机器学习的文献中的许多结果是利用了更为通用的欧拉-拉格朗日方程，
它能够使得$g$不仅依赖于$f$的导数而且也依赖于$f$的值。
但是本书中我们不需要完整的讲述这个结果。
<!-- % p 637  end -->


为了优化某个函数关于一个向量，我们求出了这个函数关于这个向量的梯度，然后找一个梯度的每一个元素都为$0$的点。
类似的，我们可以通过寻找一个函数使得泛函导数的每个点都等于$0$从而来优化一个泛函。
<!-- % p 638 head -->


下面讲一个这个过程是如何工作的，我们考虑寻找一个定义在$x\in\SetR$的有最大微分熵的概率密度函数。
我们回过头来看一下一个概率分布$p(x)$的熵，定义如下：
\begin{align}
	H[p] = - \SetE_x \log p(x).
\end{align}
对于连续的值，这个期望可以看成是一个积分：
\begin{align}
	H[p] = - \int p(x) \log p(x) dx.
\end{align}
<!-- % p 638 -->


我们不能简单地仅仅关于函数$p(x)$最大化$H[p]$，因为那样的话结果可能不是一个概率分布。
为了解决这个问题，我们需要使用一个拉格朗日乘子来添加一个$p(x)$积分值为1的约束。
同样的，当方差增大的时候，熵也会无限制的增加。
因此，寻找哪一个分布有最大熵这个问题是没有意义的。
但是，在给定固定的方差$\sigma^2$的时候，我们可以寻找一个最大熵的分布。
最后，这个问题还是无法确定因为在不改变熵的条件下一个分布可以被随意的改变。
为了获得一个唯一的解，我们再加一个约束：分布的均值必须为$\mu$。
那么这个问题的拉格朗日泛函可以被写成
\begin{align}
&		\CalL[p] =  \lambda_1 \Big(\int p(x)dx - 1\Big)  + \lambda_2 (\SetE[x] - \mu) +  \lambda_3 (\SetE[(x-\mu)^2] - \sigma^2)  + H[p]\\
& =  \int \Big(\lambda_1 p(x) + \lambda_2 p(x)x + \lambda_3 p(x)(x-\mu)^2 - p(x)\log p(x) \Big)dx - \lambda_1 - \mu \lambda_2 - \sigma^2\lambda_3.
\end{align}
<!-- % p 638    -->


为了关于$p$最小化拉格朗日乘子，我们令泛函导数等于$0$：
\begin{align}
	\forall x,\ \  \frac{\delta}{\delta p(x)} \CalL = \lambda_1 + \lambda_2 x + \lambda_3(x-\mu)^2 - 1 - \log p(x) = 0 .
\end{align}
<!-- % p 638  end  -->


这个条件告诉我们$p(x)$的泛函形式。
通过代数运算重组上述方程，我们可以得到
\begin{align}
	p(x) = \exp\big(\lambda_1 + \lambda_2 x + \lambda_3 (x-\mu)^2  - 1\big).
\end{align}
<!-- % p 638 end  -->

我们并没有直接假设$p(x)$取这种形式，而是通过最小化这个泛函从理论上得到了这个$p(x)$的表达式。
为了完成这个优化问题，我们需要选择$\lambda$的值来确保所有的约束都能够满足。
我们有很大的选择$\lambda$的自由。
因为只要约束满足，拉格朗日关于$\lambda$的梯度为$0$。
为了满足所有的约束，我们可以令$\lambda_1 = 1 - \log \sigma\sqrt{2\pi}$,$\lambda_2 = 0$, $\lambda_3 = - \frac{1}{2\sigma^2}$，从而取到
\begin{align}
	p(x) = \CalN(x;\mu,\sigma^2).
\end{align}
这也是当我们不知道真实的分布的时候总是使用正态分布的原因。
因为正态分布拥有最大的熵，我们通过这个假定来保证了最小可能量的结构。
<!-- % 639 head -->



当寻找熵的拉格朗日泛函的临界点并且给定一个固定的方差的时候，我们只能找到一个对应最大熵的临界点。
那是否存在一个最小化熵的概率分布函数呢？
为什么我们无法发现第二个最小值的临界点呢？
原因是没有一个特定的函数能够达到最小的熵值。
当函数把越多的概率密度加到$x = \mu + \sigma$和$x = \mu - \sigma$两个点上和越少的概率密度到其他点上时，他们的熵值会减少，而方差却不变。
然而任何把所有的权重都放在这两点的函数的积分并不为$1$也不是一个有效地概率分布。
所以不存在一个最小熵的概率分布函数，就像不存在一个最小的正实数一样。
然而，我们发现存在一个收敛的概率分布的序列，收敛到权重都在两个点上。
这种情况能够退化为混合Dirac分布。
因为Dirac分布并不是一个单独的概率分布，所以Dirac分布或者混合Dirac分布并不能对应函数空间的一个点。
所以对我们来说，当寻找一个泛函导数为$0$的函数空间的点的时候，这些分布是不可见的。
这就是这种方法的局限之处。
像Dirac分布这样的分布可以通过其他方法被找到，比如可以被猜到，然后证明它是满足条件的。
<!-- % 639  -->




## 连续型潜变量

<!-- % 639 end            19.4.3  -->


当我们的图模型包含了连续型潜变量的时候，我们仍然可以通过最大化$\CalL$进行变分推断和学习。
然而，我们需要使用变分法来实现关于$q(\Vh\mid\Vv)$最大化$\CalL$。
<!-- % 639 end   -->


<!-- % 640 head   -->
在大多数情况下，研究者并不需要解决任何变分法的问题。
取而代之的是，均匀场固定点迭代更新有一种通用的方程。
如果我们做了均匀场的近似：
\begin{align}
	q(\Vh\mid\Vv) = \prod_i q(h_i \mid\Vv),
\end{align}
<!-- % 640 mid -->
并且对任何的$j\neq i$固定了$q(h_j\mid\Vv)$，那么只需要满足$p$中任何联合分布中的变量不为$0$，我们就可以通过归一化下面这个未归一的分布
\begin{align}
	\tilde{q}(h_i \mid\Vv) = \exp \big(\SetE_{\RVh_{-i}\sim q(\RVh_{-i}\mid\Vv)}	\log \tilde{p}(\Vv,\Vh)\big)
\end{align}
<!-- % 640 mid -->
来得到最优的$q(h_i\mid\Vv)$。
在这个方程中计算期望就能得到一个$q(h_i\mid\Vv)$的正确表达式。
我们只有在希望提出一种新形式的变分学习算法时才需要直接推导$q$的函数形式。
方程~\eq?给出了适用于任何概率模型的均匀场近似。
<!-- % 640   -->




<!-- % 640   -->
方程~\eq?是一个不动点方程，对每一个$i$它都被迭代地反复使用直到收敛。
然而，它还包含着更多的信息。
我们发现这种泛函定义的问题的最优解是存在的，无论我们是否能够通过不动点方程来解出它。
这意味着我们可以把一些值当成参数，然后通过优化算法来解决这个问题。


<!-- % 640   -->
我们拿一个简单的概率模型作为例子，其中潜变量满足$\Vh\in\SetR^2$，可见变量只有一个$v$。
假设$p(\Vh) = \CalN(\Vh;0,\MI)$以及$p(v\mid\Vh) = \CalN(v;\Vw^{\top}\Vh;1)$，我们可以通过把$\Vh$积掉来简化这个模型，结果是关于$v$的高斯分布。
这个模型本身并不有趣。
为了说明变分法如何应用在概率建模之中，我们构造了这个模型。



<!-- % 640  end -->
忽略归一化常数的时候，真实的后验分布可以给出：
\begin{align}
   & p(\Vh\mid\Vv)\\
 \propto & p(\Vh, \Vv)\\
 = & p(h_1) p(h_2) p(\Vv\mid\Vh)\\
 \propto & \exp \big(-\frac{1}{2} [h_1^2 + h_2^2 + (v-h_1w_1 - h_2w_2)^2]\big)\\
 = & \exp \big( - \frac{1}{2} [h_1^2 + h_2^2 + v^2 + h_1^2w_1^2 + h_2^2w_2^2 - 2vh_1w_1 - 2vh_2w_2 + 2h_1w_1h_2w_2] \big).
\end{align}
在上式中，我们发现由于带有$h_1,h_2$乘积的项的存在，真实的后验并不能将$h_1,h_2$分开。
<!-- % 641 head  -->



展开方程~\eq?，我们可以得到
\begin{align}
& \tilde{q}(h_1\mid\Vv)\\ 
= & \exp\big(\SetE_{\RSh_2\sim q(\RSh_2\mid\Vv)} \log \tilde{p}(\Vv,\Vh)\big) \\
= & \exp\Big( -\frac{1}{2} \SetE_{\RSh_2\sim q(\RSh_2\mid\Vv)} [h_1^2 + h_2^2 + v^2 + h_1^2w_1^2 + h_2^2w_2^2 \\
& -2vh_1w_1 - 2vh_2w_2 + 2h_1w_1h_2w_2]\Big).
\end{align}
<!-- % 641  -->


<!-- % 641  -->
从这里，我们可以发现其中我们只需要从$q(h_2\mid \Vv)$中获得两个值：$\SetE_{\RSh_2\sim q(\RSh\mid\Vv)}[h_2]$和$\SetE_{\RSh_2\sim q(\RSh\mid\Vv)}[h_2^2]$。
把这两项记作$\langle h_2 \rangle$和$\langle h_2^2 \rangle$，我们可以得到：
\begin{align}
\tilde{q}(h_1\mid\Vv) = & \exp(-\frac{1}{2} [h_1^2 + \langle h_2^2 \rangle  + v^2 + h_1^2w_1^2 + \langle h_2^2 \rangle w_2^2 
\\ &	-2vh_1w_1 - 2v\langle h_2 \rangle w_2 + 2h_1w_1\langle h_2 \rangle w_2]).	
\end{align}
<!-- % 641  -->


从这里，我们可以发现$\tilde{q}$形式满足高斯分布。
因此，我们可以得到$q(\Vh\mid\Vv) = \CalN(\Vh;\Vmu,\Vbeta^{-1})$，其中$\Vmu$和对角的$\Vbeta$是变分参数，我们可以使用任何方法来优化它。
有必要说明一下，我们并没有假设$q$是一个高斯分布，这个高斯是使用变分法来最大化$q$关于$\CalL$\footnote{此处似乎有笔误。}推导出的。
在不同的模型上应用相同的方法可能会得到不同形式的$q$分布。
<!-- % 641 -->

当然，上述模型只是为了说明情况的一个简单例子。
深度学习中关于变分学习中连续型变量的实际应用可以参考{Goodfeli-et-al-TPAMI-Deep-PrePrint-2013-small}。
<!-- % 641 -->




## 学习和推断之间的相互作用

<!-- % 641 end  19.4.4   -->


<!-- %使用近似推断作为学习算法的一部分影响学习过程，反过来这也影响推断算法的准确性。 -->
在学习算法中使用近似推断会影响学习的过程，反过来这也会影响推断算法的准确性。
<!-- % 641 end -->


具体来说，训练算法倾向于以使得近似推断算法中的近似假设变得更加真实的方向来适应模型。
当训练参数时，变分学习增加
\begin{align}
	\SetE_{\RVh \sim q}\log p(\Vv,\Vh).
\end{align}
<!-- % 642  -->

对于一个特定的$\Vv$，对于$q(\Vh \mid \Vv)$中概率很大的$\Vh$ 它增加了$p(\Vh\mid\Vv)$，对于$q(\Vh \mid \Vv) $ 中概率很小的$\Vh$它减小了$p(\Vh\mid\Vv)$。
<!-- % 642 -->

这种行为使得我们做的近似假设变得合理。 %这种行为使我们的近似假设成为自我实现。
 如果我们用单峰值的模型近似后验分布，我们将获得一个真实后验的模型，该模型比我们通过使用精确推断训练模型获得的模型更接近单峰值。
<!-- % 642 -->


因此，估计由于变分近似对模型产生的伤害大小是很困难的。
存在几种估计$\log p(\Vv)$的方式。
通常我们在训练模型之后估计$\log p(\Vv;\Vtheta)$，然后发现它和$\CalL(\Vv,\Vtheta,q)$的差距是很小的。
从这里我们可以发现，对于特定的从学习过程中获得的$\Vtheta$来说，变分近似普遍是很准确的。
然而我们无法直接得到变分近似普遍很准确或者变分近似不会对学习过程产生任何负面影响这样的结论。
为了准确衡量变分近似带来的危害，我们需要知道$\Vtheta^* = \max_{\Vtheta} \log p(\Vv;\Vtheta)$。
$\CalL(\Vv,\Vtheta,q)\approx \log p(\Vv;\Vtheta)$和$\log p(\Vv;\Vtheta)\ll \log p(\Vv;\Vtheta^*)$同时成立是有可能的。
如果存在$\max_q \CalL(\Vv,\Vtheta^*,q)\ll \log p(\Vv;\Vtheta^*)$，即在$\Vtheta^*$点处后验分布太过复杂使得$q$分布族无法准确描述，则我们无法学习到一个$\Vtheta$。
这样的一类问题是很难发现的，因为只有在我们有一个能够找到$\Vtheta^*$的超级学习算法时，才能进行上述的比较。
<!-- % 642 -->




# learned近似推断

<!-- % 642  end    19.5 -->

我们已经看到了推断可以被视作是一个增加函数$\CalL$的值的优化过程。
显式地通过迭代方法比如不动点方程或者基于梯度的优化算法来执行优化的过程通常是代价昂贵且耗时巨大的。
通过学习一个近似推断许多推断算法避免了这个问题。
具体地说，我们可以将优化过程视作将输入$\Vv$投影到一个近似分布$q^* = \arg\max_q\  \CalL(\Vv,q)$的函数$f$。
一旦我们将多步的迭代优化过程看作是一个函数，我们可以用一个近似函数为$\hat{f}(\Vv;{\Vtheta})$的神经网络来近似它。



## wake sleep

<!-- % 643 head 19.5.1 -->

训练一个可以用$\Vv$来推断$\Vh$的模型的一个主要难点在于我们没有一个有监督的训练集来训练模型。
给定一个$\Vv$，我们无法获知一个合适的$\Vh$。
从$\Vv$到$\Vh$的映射依赖于模型类型的选择，并且在学习过程中随着${\Vtheta}$的改变而变化。
wake sleep算法{cite?}通过从模型分布中抽取$\Vv$和$\Vh$的样本来解决这个问题。
例如，在有向模型中，这可以通过执行从$\Vh$开始并在$\Vv$结束的原始采样来高效地完成。
然后推断网络可以被训练来执行反向的映射：预测哪一个$\Vh$产生了当前的$\Vv$。
这种方法的主要缺点是我们将只能够训练推断网络在模型下具有高概率的$\Vv$值。
在学习早期，模型分布与数据分布偏差较大，因此推断网络将不具有在类似数据的样本上嘘唏的机会。
<!-- % 643 -->


在\sec?中，我们看到做梦睡眠在人类和动物中作用的一个可能解释是，做梦可以提供蒙特卡罗训练算法用于近似无向模型中对数配分函数负梯度的负相位样本。
生物做梦的另一个可能的解释是它提供来自$p(\Vh,\Vv)$的样本，这可以用于训练推断网络在给定$\Vv$的情况下预测$\Vh$。
在某些意义上，这种解释比配分函数的解释更令人满意。
如果蒙特卡罗算法仅使用梯度的正相位进行几个步骤，然后仅对梯级的负相位进行几个步骤，那么他们的结果不会很好。
人类和动物通常连续清醒几个小时，然后连续睡着几个小时。
这个时间表如何支持无向模型的蒙特卡罗训练尚不清楚。
然而，基于最大化$\CalL$的学习算法可以通过长时间调整改进$q$和长期调整${\Vtheta}$来实现。
如果生物做梦的作用是训练网络来预测$q$，那么这解释了动物如何能够保持清醒几个小时（它们清醒的时间越长，$\CalL$和$\log p(\Vv)$之间的差距越大， 但是$\CalL$将保持下限）并且睡眠几个小时（生成模型本身在睡眠期间不被修改）， 而不损害它们的内部模型。
当然，这些想法纯粹是猜测性的，没有任何坚实的证据表明做梦实现了这些目标之一。
做梦也可以服务强化学习而不是概率建模，通过从动物的过渡模型采样合成经验，从来训练动物的策略。
也许睡眠可以服务于一些机器学习社区尚未发现的其他目的。
<!-- % 644 head -->




## learned推断的其他形式

<!-- % 644    19.5.2  -->

这种learned近似推断策略已经被应用到了其他模型中。
{Salakhutdinov+Larochelle-2010}证明了在learned推断网络中的单一路径相比于在深度玻尔兹曼机中迭代均匀场不动点方程能够得到更快的推断。
训练过程基于运行推断网络，然后应用均匀场的一步来改进其估计，并训练推断网络来输出这个精细的估计而不是其原始估计。
<!-- % 644 -->


我们已经在\sec?中已经看到，预测性的稀疏分解模型训练浅层的编码器网络以预测输入的稀疏编码。
这可以被看作是自编码器和稀疏编码之间的混合。
为模型设计概率语义是可能的，其中编码器可以被视为执行learned近似最大后验推断。
由于其浅层的编码器，PSD不能实现我们在均匀场推断中看到的单元之间的那种竞争。
然而，该问题可以通过训练深度编码器来执行learned近似推断来补救，如ISTA技术{cite?}。
<!-- % 644 -->


近来learned近似推断已经成为了变分自编码器形式的生成模型中的主要方法之一{cite?}。
在这种优美的方法中，不需要为推断网络构造显式的目标。
反之，推断网络被用来定义$\CalL$，然后调整推断网络的参数来增大$\CalL$。这种模型在\sec?中详细描述。
<!-- % 644 -->

我们可以使用近似推断来训练和使用大量的模型。
许多模型将在下一章中描述。
<!-- % 644 -->















