---
layout: post
title: 有关内存DC和双缓冲位图的问题汇总
date: 2009-12-21
categories: tech
tags: [mfc,dc,bitmap]
---

本文对最近在使用双缓冲画图遇到的问题进行一个总结。
双缓冲是画图中使用频繁的手法，用于防止绘图闪烁的问题。
使用框架：
```C++
CDC m_memDC;

//初始化兼容的内存DC
m_memDC.CreateCompatibleDC(pDC);
CBitmap m_bmpWave;

//创建设备兼容的位图缓冲
if(!m_bmpWave.CreateCompatibleBitmap(pDC,m_bmpwidth,m_bmpheight))
{
    ErrorMessageBox(IDS_CREATEBITMAPERROR);
    return false;
}
//将位图选到内存DC中用于绘图
m_pOldBmp = m_memDC.SelectObject(&m_bmpWave);
//此处使用内存DC进行各种绘图操作

    .......

pDC->BitBlt(0,0,width,height,&m_memDC,0,0,SRCCOPY);     //将内存图像拷贝到当前设备DC中完成绘图的显示

//后继处理
```
框架就这样，说说我遇到的问题。

内存位图无法显示在设备DC中
------

当然，我的代码比较多。所以这种情况很难调试，只能是估计可能存在的问题。通过调试和分析，找出了问题的所在。由于一些原因，我用到了两个内存DC，我首先将Bitmap通过SelectObject选入到第一个内存DC中进行绘图，然后再第二个内存DC中将同一个Bitmap选入并进行变形操作，然后将第二个内存DC通过BitBlt拷贝到当前设备DC中。刚开始时，位图没有拷贝成功，问题在于第二个DC在选入Bitmap时，第一个内存DC并没有将Bitmap选出，所以问题应该是Bitmap不能同时选入两个不同的DC进行操作。问题找到了，解决方法也就有了，要么在第二个DC选入之前，第一个DC先选出。要么只使用一个内存DC。

关于这个问题后来在MSDN中也看到了相关的说明如下：

An application can select a bitmap into memory device contexts only and into only one memory device context at a time.

证明我的想法是正确的。看来文档还是要多看看，很多问题其实在文档中都有说明。

内存位图拷贝后，显示出来的是黑白的
------

这个就是位图的创建方式的问题了，CreateCompatibleBitmap(pDC,m_bmpwidth,m_bmpheight)，这个创建位图的第一个参数表示设备DC，位图将会使用和设备DC兼容的颜色和格式。如果传入的是内存DC的指针，那么该位图就使用内存DC的颜色板。而内存DC默认为黑白模式的。因此使用内存DC创建的位图自然也是黑白的。

MSDN的解释：

When a memory device context is created, GDI automatically selects a monochrome stock bitmap for it.

If pDC is a memory device context, the bitmap returned has the same format as the currently selected bitmap in that device context.

好了，解决方案就是使用当前设备DC创建位图而不是内存DC。
