---
title: 朴素贝叶斯法
date: 2016-09-08 13:44:41
tags: [naive-Bayes, 分类算法, 极大似然估计]
---

## 朴素贝叶斯公式

朴素贝叶斯法是基于贝叶斯定理与特征条件独立假设的分类方法。

`贝叶斯公式` 可以根据 $$$ P\left(A|B\_i\right) $$$ 计算出`后验概率` $$$ P\left(Y\_i|X\right) $$$ 的模型：

\\[
P\left(Y\_i|X\right) = {P\left(X|Y\_i\right)P\left(Y\_i\right) \over \sum\_{k=1}^n P\left(X|Y\_i\right)P\left(Y\_i\right)}
\\]

> `先验概率`和`后验概率`
>
> `先验概率`是在缺乏某个事实的情况下描述一个变量; 而后验概率是在考虑了一个事实之后的条件概率.  先验概率通常是经验丰富的专家的纯主观的估计. 比如在法国大选中女候选罗雅尔的支持率 p,  在进行民意调查之前, 可以先验概率来表达这个不确定性.
>
> `后验概率`: Probability of outcomes of an experiment after it has been performed and a certain event has occured.  
> See:<http://blog.sina.com.cn/s/blog_4ce95d300100fwz3.html>


`特征条件独立假设`：

条件概率分布

\\[ 
P(X=x|Y=c\_k)=P(X^{(1)}=x^{(1)}, X^{(2)}=x^{(2)}, \cdots, X^{(n)}=x^{(n)} | Y=c\_k), k=1,2,\cdots,K 
\\]

有指数级数量的参数，其估计实际是不可行的。实际上，假设 $$$X^{(j)}$$$ 可取值有 $$$S\_j$$$ 个，$$$j=1,2,\cdots,n$$$，$$$Y$$$ 可取值有 $$$K$$$ 个，那么参数个数为 $$$K \prod\_{j=1}^n S\_j$$$。

如果各特征 $$$X^{(i)}$$$ 独立，那么条件概率分布

\\[\begin{aligned} 
P(X=x|Y=c\_k) & = P(X^{(1)}=x^{(1)}, X^{(2)}=x^{(2)}, \cdots, X^{(n)}=x^{(n)} | Y=c\_k) \\\
& = \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k)
\end{aligned}\\]

这样可以简化条件概率的计算，但有时会牺牲一定的分类准确性。

简化后的贝叶斯公式为：

\\[
P(Y=c\_k|X=x) = {P(Y=c\_k) \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k) \over \sum\_k \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k)} \\\
k = 1,2,\cdots,K
\\]

称为`朴素贝叶斯公式`。

## 朴素贝叶斯分类

对给定的输入 $$$x$$$，通过学习到的模型计算后验概率分布 $$$P(Y=c\_k|X=x)$$$，将后验概率最大的类作为 $$$x$$$ 的类。

\\[
y = f(x) = arg \max\_{c\_k} {P(Y=c\_k) \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k) \over \sum\_k \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k)}
\\]

其中分母对所有的 $$$c\_k$$$ 都相同，所以，

\\[
y = arg \max\_{c\_k} {P(Y=c\_k) \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k)}
\\]

朴素贝叶斯法将实例分到后验概率最大的类中，这等价与期望风险最小化。

## 极大似然估计

朴素贝叶斯法中，用极大似然估计分估计 $$$P(Y=c\_k)$$$ 和 $$$P(X^{(j)}=x^{(j)} | Y=c\_k)$$$：

\\[
P(Y=c\_k) = {\sum\_{i=1}^N {I(y\_i=c\_k)} \over N},  k=1,2,\cdots,K
\\]

设第 $$$j$$$ 个特征 $$$x^{(j)}$$$ 可取值的集合为 $$${a\_{j1},a\_{j2},\cdots,a\_{j{S\_j}}}$$$，则

\\[
P(X^{(j)}=a\_{jl} | Y=c\_k) = {\sum\_{i=1}^N {I(x\_i^{(j)}=a\_{jl}, y\_i=c\_k)} \over \sum\_{i=1}^N {I(y\_i=c\_k)}}
\\]

可理解为：某分类为类别 $$$c\_k$$$ 中某特征 $$$x^{(j)}$$$ 值为 $$$a\_{jl}$$$ 的实例所占的比例，它为训练样本中此特征值 $$$a\_{jl}$$$ 对分类为 $$$c\_k$$$ 影响力的极大似然估计。

## 学习和分类：朴素贝叶斯算法(naive-Bayes algorithm)

算法：

> 输入

训练数据 $$$T={(\vec{x\_1}, y\_1),(\vec{x\_2}, y\_2), \cdots, (\vec{x\_N}, y\_N)}$$$，其中 $$$\vec{x\_i}=(x\_i^{(1)}, x\_i^{(2)}, \cdots, x\_i^{(N)})^T$$$, $$$x\_i^{(j)}$$$ 是第 $$$i$$$ 个样本的第 $$$j$$$个特征，$$$x\_i^{(j)} \in \\{a\_{j1}, a\_{j2}, \cdots, a\_{j{S\_j}}\\}$$$，$$$a\_{jl}$$$ 是第 $$$j$$$ 个特征可能取的第 $$$l$$$ 个值，$$$j=1,2,\cdots,n, l=1,2,\cdots,S\_j, y\_i \in \\{c\_1, c\_2, \cdots, c\_k\\}$$$； 实例 $$$\vec{x}$$$；

> 输出

实例 $$$\vec{x}$$$ 的分类


> 步骤

1. 计算先验概率及条件概率

	\\[\begin{aligned}
P(Y=c\_k) &= {\sum\_{i=1}^N {I(y\_i=c\_k)} \over N},  k=1,2,\cdots,K \\\
P(X^{(j)}=a\_{jl}|Y=c\_k) &= {\sum\_{i=1}^N {I(x\_i^{(j)}=a\_{jl}, y\_i=c\_k)} \over \sum\_{i=1}^N {I(y\_i=c\_k)}}, j=1,2,\cdots,n, l=1,2,\cdots,S\_j, y\_i \in \\{c\_1, c\_2, \cdots, c\_k\\}
\end{aligned}\\]

2. 对于给定的实例 $$$\vec{x\_i}=(x\_i^{(1)}, x\_i^{(2)}, \cdots, x\_i^{(N)})^T$$$，计算

	\\[ {P(Y=c\_k) \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k)}, k=1,2,\cdots,K \\]
	
3. 确定实例 $$$\vec{x}$$$ 的分类

	\\[
y = arg \max\_{c\_k} {P(Y=c\_k) \prod\_{j=1}^n P(X^{(j)}=x^{(j)} | Y=c\_k)}
\\]

## 贝叶斯估计

为了避免极大似然估计可能会出现的估计概率为0的情况，引入贝叶斯估计：

\\[\begin{aligned}
P(Y=c\_k) &= {\sum\_{i=1}^N {I(y\_i=c\_k) + \lambda} \over {N + K\lambda}},  k=1,2,\cdots,K \\\
P(X^{(j)}=a\_{jl}|Y=c\_k) &= {\sum\_{i=1}^N {I(x\_i^{(j)}=a\_{jl}, y\_i=c\_k)}+\lambda \over \sum\_{i=1}^N {I(y\_i=c\_k) + S\_j\lambda}}, j=1,2,\cdots,n, l=1,2,\cdots,S\_j, y\_i \in \\{c\_1, c\_2, \cdots, c\_k\\}
\end{aligned}\\]

当 $$$\lambda=1$$$ 时， 成为`拉普拉斯平滑(Laplace smoothing)`。

实现代码：<https://github.com/yungoo/maching-learning-quiz/blob/master/naive-bayesian/naive-bayesian.py>