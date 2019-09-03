---
title: UI系统中的问题
layout: post
categories: todo
tags: ''
---
## DWRITE的效率瓶颈

首先，ariel实现了runtime的动态合批，因此draw text被转换成了2d绘制中常见的picture fill，由于batching的时候texture batch的优先级很高，因此就*顺便*把draw text也丢进texture batch中了。

麻烦也就来了，这个流程大致需要做的事情是

1. 查询这次draw text需要多大尺寸的texture
2. 用d3d创建一个该尺寸的texture（BGRA）
3. 在此texture上query出DXGIResource接口，然后创建d2d的render target
4. pD2dRenderTarget->DrawText，此时为BGRA premultiplied格式
5. 转为d3d texture的R8G8B8A8的straight格式

万万没想到竟然是cpu bound，并且热点在CreateDxgiSurfaceRenderTarget上。
![hotspot1](https://github.com/lymastee/lymastee.github.io/blob/master/resources/2019-09-03/HOTSPOT1.png?raw=true)
![hotspot2](https://github.com/lymastee/lymastee.github.io/blob/master/resources/2019-09-03/HOTSPOT2.png?raw=true)

所以之后这个batch需要做一些调整，要把所有绘制文字的操作batch到一起，这个CreateDxgiSurfaceRenderTarget尽可能少才行。

## runtime的动态合批

这绝对不是一个好想法，老老实实的预先分好texture atlas，也不要做矢量绘制，老老实实地抠贴图。游戏中的HUD，之后还是需要重新实现。

## clip算法

现在的实现比较烂，懒得改了，顺着写下去吧，凑合用。
但我是真的想改，只是没精力了。
还有scroll用的axis aligned clip，是可以加速的，但是emmmm算了，去球。

## 曲线stroke的顶点数量过多

这个功能是可以优化的，如果要继续改这个鬼东西的话，多分出一个batch来就能做到，但是下面几点需要注意：

1. 和现有的fill的clip的地方（全都是triangle），之前是line-triangle的clip，现在需要改为curve-triangle的clip。
2. 画curve不在直线化之后画，而是直接传control path，用blinn loop的那个方式去画，而且可以直接选择AA或者不AA，但是这个时候需要对曲线外扩或者内缩一点，取决于曲线宽度，但是这个时候不能简单移动control path，否则误差就太大了。
3. PS（FS）中只画stroke是有很大的overdraw的，不要想太多，直接discard，因为2D的绘制时都没有传DepthStencilView，不用考虑earlyz的问题。

以上