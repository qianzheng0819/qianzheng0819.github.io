---
layout: post
title:  "animation总结"
date:   2019-05-02 19:17:00 +0800
categories: android
tags:   android
description:
---

顺着[官方文档](https://developer.android.com/training/animation/index.html?hl=zh-cn)做一个总结

这里是知乎高效学习动画的[链接](https://www.zhihu.com/question/27718787)

animate drawable大概就是帧动画,阅读官方文档即可;

AnimatedVectorDrawable比较复杂,可以实现很多类似gig的效果,但是学习成本较高,暂不深入.记录一个[开源项目](https://github.com/panzhenglian/VectorDemo)

动画分视图动画和属性动画.视图动画的父类是animation,属性动画的父类是animator.TimeInterpolator是为属性动画引入的的接口,原Interpolator继承TimeInterpolator,所以原来的interpolator属性动画都可以用.

**interpolator和evaluator**

interpolator计算interpolated fraction,evaluator根据fraction计算属性视图位置.com.daimajia.easing是一个好用的evaludtor库,不需要重温数学来计算了.

**动画当中的部分属性**

android:pivotX 表示缩放/旋转起点 X 轴坐标，可以是整数值、百分数（或者小数）、百分数p 三种样式，比如 50、50% / 0.5、50%p。需要明确的是，这里以进行动画控件的左上角为原点坐标，当属性值为数值，如 50 时，表示原点坐标加上 50px，作为起始点；如果是百分数，比如 50%，表示原点坐标加上自己宽度的 50%（即控件水平中心）作为起始点 ；如果是 50%p（字母 p 是 parent 的意思），取值的基数是父控件，因此 50%p 就是表示在原点坐标加上父控件宽度的 50% 作为起始点 x 轴坐标。
