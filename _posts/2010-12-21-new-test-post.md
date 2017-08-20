---
layout: post
title: test post
date: 2010-12-21
categories: tech
tags: [test]
---

## 标题1

### title2

- list
- list
- list

> quto this

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
1. numlist
2. numlist

[link](http://www.baidu.com)

horizon

---
another horizon

***
