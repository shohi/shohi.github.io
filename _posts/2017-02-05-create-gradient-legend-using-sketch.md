---
layout: post
title: 使用sketch制作渐变图例
category : design
guid: e5290ceb-eba1-11e6-9a6a-acbc32c984c7
tags : [sketch]

---
{% include JB/setup %}

使用`sketch`制作渐变颜色，有几个步骤：


`1.`  画一个长方形区域，设置颜色填充方式及方向, 如flat linear或者radial

<img src="/assets/images/sketch/gradient-vertical.png" width="500" alt="gradient-vertical">

当填充方式`fill`选择为linear或者radial时，长方形区域上会出现渐变起始位置和终止位置所表示的点(以下称为渐变点)。点上的颜色对应了`Fill`栏下的长条上指定的颜色。拖动渐变点，将其调整为水平方向，同时保持颜色顺序与长条上的颜色一致,同时点击长条上的竖框，修改渐变点的颜色为`#00ff00`和`#ff0000`.

<img src="/assets/images/sketch/gradient-horizontal.png" width="500" alt="gradient-horizontal">



`2.`  将长方形区域划分成几个颜色段，并指定其首尾颜色。选中渐变线，鼠标点击线上的点即可添加渐变点，拖动渐变点可以调整其位置。新增两个渐变点，分别制定其颜色为`#ffff00`和`#ff7d00`，并将填充方式选为radial.

<img src="/assets/images/sketch/gradient-multiple.png" width="500" alt="gradient-multiple">



`3.`  添加上图例名称及说明，最终效果如下
![gradient-legend](/assets/images/sketch/gradient-legend.png)



## 问题
1. 如何精确地设定各颜色段范围，而不是使用拖动的方式?


*The End.*
