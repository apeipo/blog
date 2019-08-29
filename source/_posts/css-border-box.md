---
title: CSS的box-sizing属性引起的一个问题
date: 2017-06-26 14:42:22
tags: [CSS]
category: 前端
---
整理一个box-sizing引起的样式问题。
## 问题
在使用腾讯地图的api展示地图时，发现地图的比例尺展示空白了。
如图：![-w100](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-26-14984499285621.jpg)，正常应该是：![-w100](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-26-14984502329840.jpg)

Chrome下，元素的盒子模型如下：
![-w200](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-26-14984500426038.jpg)
关键的几个css样式：
![-w200](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-26-14984501217132.jpg)
元素的border-color设置了白色，因此元素实际要展示的颜色应该是由background-color决定的，但是从盒子上可以看出，**元素内容部分(content)的高度为0**，导致元素没有展示实际的黑色。
元素的样式设置了height为2px，样式都是在style属性中定义，不存在被覆盖的情况，为什么实际高度是0呢？看着好像高度被border占了？
先回顾下CSS的盒子模型。
## 盒子模型
如下图，盒子模型描述了一个元素在页面中**占用的空间**。注意，**占用空间 != 高度X宽度**。
**默认情况下，当你指定一个CSS元素的宽度和高度属性时，你只是设置内容区域（下图中的content）的宽度和高度**。一个元素实际占用的空间还需要加上padding、border和margin。
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-26-14984486313104.gif)

回到上文中的问题，style中设定了div的高度为2，也就是上图中Content区域的高度为2。但是Chrome中实际渲染出来Content的高度为0。也就是盒子模型发生了变化？？是有什么属性会改变浏览器的盒子模型计算高度、宽度的方式么？
是的，这个属性就是box-sizing属性。
## box-sizing
[MDN文档链接](https://developer.mozilla.org/zh-CN/docs/Web/CSS/box-sizing)
box-sizing 属性用于更改用于更改CSS盒子模型计算元素宽度和高度的方式。
该属性有两个取值，**content-box**和**border-box**
### content-box
默认值，标准盒子模型。 width与height只包括Content的宽和高， 不包括边框（border）、内边距（padding）、外边距（margin）。
如一个元素的width=20px，border=10px。则在浏览器中实际的**占用宽度**是40px;
### border-box
 width 和 height 属性包括Content，Padding和Border，不包括Margin。
 如一个元素的width=20px，border=10px。则在浏览器中实际的占用宽度是20px。
 注意，在这种情况下（width=border+padding），**则会挤压Content的空间**，也就是Content的width=0。
## 总结
至此，问题原因应该明朗了，查看出问题的div的box-sizing属性，果然是border-box（页面用了bootstrap.css，将全局的div的box-sizing都设置成了border-box）。
解决方案也很简单，在页面里增加css样式，将包含地图控件下所有div的box-sizing属性设置为content-box。
## 其他问题
### border-box，如果border+padding超过了width会怎么样？
先说结果：这种情况下，Content区域会被挤压为0，元素实际占用的宽度会是border+padding。
例如 width:20px; padding:10px; border: 1px; box-sizing:border-box; 则元素的占用宽度为22，Content区域宽度为0。其盒子模型如下：
![-w200](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2017-06-26-14984595662080.jpg)





