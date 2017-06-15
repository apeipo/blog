---
title: 梯度下降笔记
date: 2017-06-14 17:02:22
tags: [MachineLearning]
category: 机器学习
---
关于梯度下降两篇比较易懂的文章：
[从导数谈起](http://www.cnblogs.com/jianxinzhou/p/3950518.html)
[线性回归及梯度下降](http://blog.csdn.net/xiazdong/article/details/7950084)
## 导数
>一个函数在某一点的导数描述了这个函数在这一点附近的变化率

先从1维上来说，例如对于函数y = x^2 。 要求得他的极值点，很简单：对x求导 y' = 2x，令2x = 0，求解得到当x=0时，y取得极值。
如果拓展到2维呢？如 z(x, y) = x^2 + y^2 。还是一样的方法，不同的是需要对x和y分别求**偏导**。
z'(x) = 2x ; z'(y) = 2y。令z'(x)和z'(y)等于0，可以解方程得到z(x, y)的极值点为（0, 0）。
同样的方法可以拓展到多维，对每个变量求偏导，并分别求得偏导为0时的变量值，结果向量就是函数的极值点。
## 梯度
导数描述了函数的变化率
对于平面上的曲线，一个点的导数可以理解该点切线的斜率，切线的正方向，是函数的上升方向。
对于多维的空间，就是该点的梯度，梯度的正方向，是函数的上升反向，反之就是下降方向。

>对于函数F(X)（X = {x0,x1,…,xn})而言，梯度是F(X)对各个分量求偏导后的结果,代表了F(X)在各个方向的变化率

## 梯度下降
既然让导数为0可以得到函数的极值点，那为什么要梯度下降？
因为很多时候，我们知道导数的表达式，但是没法求导数为0的方程（比如导数是 exp(x)+(ln(x))^2 + x^5 这种..）。这种情况下，我们只能根据根据每个点的导数值（即梯度），选择梯度的反方向去逼近函数的极小值点，这种方法就是梯度下降，反之就是梯度上升。

### 梯度下降步骤
例如对于单变量线性回归（theta0 + theta1 * x）的损失函数：
![-w300](http://7xrhmq.com1.z0.glb.clouddn.com/2017-06-14-14974242248427.jpg)
其梯度下降的方法：
(1)先确定向下一步的步伐大小，称为Learning rate；
(2)任意给定一个初始值：theta0, theta1；
(3)确定一个向下的方向，并向下走预先规定的步伐，并更新；
(4)当下降的高度小于某个定义的值，则停止下降；
![-w392](http://7xrhmq.com1.z0.glb.clouddn.com/2017-06-14-14974243096274.jpg)
### 注意点
1. 初始点不同，获得的最小值也不同，因此梯度下降求得的只是局部最小值
2. 降的步伐大小非常重要，因为如果太小，则找到函数最小值的速度就很慢，如果太大，有可能呈之字型下降。如图：
![-w200](http://7xrhmq.com1.z0.glb.clouddn.com/2017-06-14-14974245759806.jpg)
3. 如果Learning rate取值后发现J function 增长了，则需要减小Learning rate的值。
4. 梯度下降是通过不停的迭代，为了减少迭代次数，因此引入了Feature Scaling（**特征标准化**），使得取值范围**大致**都在-1<=x<=1之间。

如果不同scale的特征，没有进行标准化，例如某个特征范围是[0, 1]，而另外一个特征是[100w, 200w]。可以想象，等高线的图会特别扁或者特别细。此时，梯度下降的步长得取得特别小，会导致另外一个特征的收敛特别慢。

## 梯度下降的种类
[梯度下降法的三种形式BGD、SGD以及MBGD](http://www.cnblogs.com/maybe2030/p/5089753.html)
这篇文章有详细的介绍，这里只进行简单的总结。
### BGD（批量梯度下降）
最原始的梯度下降方法， 每次迭代时都使用所有的样本来进行更新。
对于上文中线性回归的损失函数，对每个变量求偏导后可以得出梯度下降的变化式：
![-w300](http://7xrhmq.com1.z0.glb.clouddn.com/2017-06-14-14974267407348.jpg)
可以看出，每次参数的迭代，都需要全体的样本参与。由于是全体参与迭代，参数稳定收敛，但是在训练集很大时，训练效率会非常低。
	
### SGD（随机梯度下降）
[维基百科](https://en.wikipedia.org/wiki/Stochastic_gradient_descent)
利用单个样本来进行迭代，解决样本量大时BGD迭代速度慢的问题。
如上文，将线性回归的损失函数进行转换：
![-w400](http://7xrhmq.com1.z0.glb.clouddn.com/2017-06-14-14974280777979.jpg)
这个式子可以解释为，整体的损失可以拆解为单个样本的损失之和，如果能将单个样本的损失减小，则总的损失也会减小（**减小单个样本的损失来逼近总体损失的极小值**）。
**可以将SGD理解样本全集只有1个时的BGD。**
对单个样本的cost函数求偏导后得出theta的变化式：
![-w300](http://7xrhmq.com1.z0.glb.clouddn.com/2017-06-14-14974284080883.jpg)
SGD每次迭代只有一个样本参与，因此可以大大增加迭代速度，提升训练速度。但是由于每次迭代都是局部进行，迭代比较"盲目"。学习过程，目标函数可能会存在波动：
![-w200](http://7xrhmq.com1.z0.glb.clouddn.com/2017-06-14-14974298909255.png)
SGD 收敛过程中的波动，可能会帮助目标函数跳入另一个可能的更小的极小值，但是也有可能跳出原来全局的极小值。
#### SGD和BGD的伪代码

```py
#BGD
choose initial vector of parameters and learning rate
Repeat until an approximate minimum is obtained:
  params_grad = evaluate_gradient(loss_function, data, params)
  params = params - learning_rate * params_grad

#SGD
choose initial vector of parameters and learning rate
Repeat until an approximate minimum is obtained:
  np.random.shuffle(data)
  for example in data:
    params_grad = evaluate_gradient(loss_function, example, params)
    params = params - learning_rate * params_grad
```
### MBGD（小批量梯度下降法）
SGD和BGD的折衷方法，即每次迭代使用指定数量的一批样本。


