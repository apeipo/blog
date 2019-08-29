---
title: 逻辑回归思考
date: 2017-06-14 17:01:22
tags: [MachineLearning]
category: 机器学习
---
参考链接：
[logistic回归详解一：为什么要使用logistic函数](http://blog.csdn.net/bitcarmanlee/article/details/51154481)
[logistic回归详解二：损失函数（cost function）详解](http://blog.csdn.net/bitcarmanlee/article/details/51165444)
[*浅析Logistic Regression](https://chenrudan.github.io/blog/2016/01/09/logisticregression.html)
## 从线性回归到逻辑回归
对于线性回归来说，模型的假设是输入向量x和真实值y之间存在关系：y = wx + b，即用wx + b去逼近样本的真实值。
拓展到广义的线性模型（机器学习第三章P56），可以令wx + b去逼近y的衍生函数，例如 ln^y = wx + b。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-14-14973232448037.jpg)
对于二分类问题，逻辑回归提出的假设是，令wx + b逼近样本为正例的概率。参考上图，即假设：P = g^-1 * (wx + b)。此时，需要一个合适的函数g，能将wx + b 映射到概率[0, 1]之间，因此引入了sigmoid函数。

## 为什么要用sigmoid函数？
理想的模型函数是如图3.2的红色函数，其中z = wx + b，即线性模型的预测结果。如果z>0则判定为正例，z<0则为反例。
![-w500](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-14-14972714947069.jpg)
但是，这个函数存在不连续，在z=0的点不可导等特点，数学上处理起来不方便。而**sigmoid函数，刚好满足二分类的需求，并且具有良好的连续性**。
>线性函数的值越接近正无穷，概率值就越接近1；线性值越接近负无穷，概率值越接近0，这样的模型是逻辑斯蒂回归模型(李航.《统计机器学习》)

## 逻辑回归损失函数怎么来的？
机器学习或者统计机器学习常见的损失函数：

1. 平方损失函数（quadratic loss function): L(Y,f(X))=(Y−f(x))^2
2. 绝对值损失函数(absolute loss function): L(Y,f(x))=|Y−f(X)|
3. 对数损失函数（logarithmic loss function) 或**对数似然损失函数**(log-likehood loss function):
L(Y,P(Y|X))=−logP(Y|X), P(Y|X)表示样本X标记正确的概率。
逻辑回归使用的是对数损失函数，根据对数损失定义，可以得出单个样本的损失函数如下：

![-w400](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-14-14973239587272.jpg)
将两个式子合并为一个：

![-w400](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-14-14973241686186.jpg)

全体样本的损失函数可以表示为：

![-w400](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-14-14973241924422.jpg)

定义损失函数之后，剩下的就是求解的过程，逻辑回归求解方法有梯度下降、牛顿法、BFGS等。
## 最小二乘、极大似然、梯度下降
[知乎问题](https://www.zhihu.com/question/24900876)
机器学习的框架是：**模型、目标和算法**。模型是指输入到输出的映射方式，目标是让模型"更好"地拟合数据，或者说让模型损失(Cost Function)达到最小，算法是指达到目标使用的方法。
最小二乘和梯度下降都属于求解方法（优化算法），最小二乘一般用于线性模型，梯度下降很多模型都会使用，比如逻辑回归（线性模型也可以用梯度下降）。
极大似然法是一种推导方式，使用极大似然法可以从概率的角度推导出模型的损失函数（比如逻辑回归的对数似然）。
>极大似然估计：在已经得到试验结果的情况下，我们应该寻找**使这个结果出现的可能性最大的那个theta作为真实theta的估计**

对应到逻辑回归中，就是寻找参数theta，使得每个样本属于其真实标记的概率越大越好（机器学习 - P59）


