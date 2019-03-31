---
title: Ray Tracing in the next week - Bouding Volume Hierarchies
date: 2019-3-31 10:59
categories:
- 游戏开发
- 图形渲染
---
最近在看Ray Tracing 系列，打算继续写博客来记录相关内容。

第二章是关于层次包围盒的内容。
作者认为这部分内容是最难的也是最重要的一部分内容，而这部分能让ray tracing更快，因此也提前放在第二章来讲，这部分也重构了hitable类，为后续rectangle和box做准备。
    
光线碰撞的检测在ray tracer中是最耗时的瓶颈，时间也跟场景物体的个数呈线性相关。然而同一个模型会被重复的判断，因此可以用二叉查找的思想来进行对数级的优化。
由于我们对同一个模型会发射数以万计的射线，因此我们可以先进行预处理，对模型进行排序，之后对每条射线进行次线性的搜索。
最常用的两种排序族分别是：1）区域划分； 2）物体划分；第二种相对来说实现起来比较简单，而且对大多数模型来说速度和第一种差不多。

包围盒的关键思想是找到一个完全覆盖所有物体的盒子。例如说，当你找到一个包围10个物体的包围方盒。此时如果光线不会射中这个方盒，那就肯定不会射中那10个物体。因此相关的伪代码就是这样的：
``` github
if (ray hits bounding object)
    return whether ray hits bounded objects
else
    return false
```
将物体划分为多个子集是非常关键的。各个物体只包围在（属于）某一个包围盒中，各个包围盒可以重复。
![包围盒示意图](https://github.com/coderooookie/coderooookie.github.io/blob/master/img/2-2-1.jpg)


