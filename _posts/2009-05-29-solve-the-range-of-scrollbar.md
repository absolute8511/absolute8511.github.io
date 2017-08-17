---
layout: post
title: 滚动条滚动范围的问题总结
date: 2009-05-29
categories: tech
tags: [vc,mfc,scrollbar]
---

最近在滚动条的问题上纠结了很久，所有问题都归结于一个滚动事件处理函数的bug，就是
```
OnHScroll(UINT nScrollCode, UINT nPos, BOOL bDoScroll)
```
问题是这样出现的：我写了一个显示位图的程序，由于图片长度很长，因此需要滚动条来滚动图片，因此我加了一个滚动条控件，并将它的滚动范围SCROLLINFO.nMax设置成一个很大的值（最大可达5000000多）。结果在滚动事件处理函数`OnHScroll(UINT nSBCode, UINT nPos, CScrollBar* pScrollBar) `里面传进来的nPos的值出现问题了，因而不能滚动到正确位置。

在网上找了很久没有结果，偶然找到了google的groups，并提了一个问题，不想一会儿就有知情人告知nPos参数虽然是UINT的类型，不过却只能接收16位以内的数据。根据这条线索，我才找到了解决方案，微软的东西真是令人不爽啊，明明是UINT却只能接收16位，害我郁闷了半天。

解决方案在这里
http://support.microsoft.com/default.aspx?scid=kb;en-us;166473

概括来说就是要在OnHScroll函数里面自己获取nPos的值，方法是使用GetScrollInfo函数，对于我的来说就是如下这样：
```
void CWavEditFormView::OnHScroll(UINT nSBCode, UINT nPos, CScrollBar* pScrollBar) 
{
    ......

    //事件处理函数的nPos只能接收16位的数据，大于16位的需要用下面的方法来获取nPos
    if(nSBCode==SB_THUMBTRACK||nSBCode==SB_THUMBPOSITION)
    {
        SCROLLINFO si;
        si.cbSize = sizeof(SCROLLINFO);
        si.fMask = SIF_TRACKPOS;
        pScrollBar->GetScrollInfo(&si);
        nPos = si.nTrackPos;
    }
    ......
}
```