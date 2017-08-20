---
layout: post
title: MCI处理2G以上wav文件的问题修复
date: 2010-01-21
categories: tech
tags: [mfc,mci,wav,windows]
---

这个问题在我测试查询大于2G的wav文件长度时遇到的，后来在google讨论组进行了讨论，初步可以确定这是MCI库函数中的BUG，在处理2G以上wav文件会出现，其他情况未验证。

bug情况：用mciSendCommand函数查询文件长度或查询正在播放的位置会返回错误的结果。

bug重现代码：
```C++
    MCI_SET_PARMS mciSetParms;
    mciSetParms.dwTimeFormat = MCI_FORMAT_SAMPLES;
    if(m_ErrorCode = mciSendCommand(m_wDeviceID,MCI_SET,MCI_WAIT|MCI_SET_TIME_FORMAT,
        (DWORD_PTR) &mciSetParms))
    {
        return false;
    }
    MCI_STATUS_PARMS mciStatusParms;
    mciStatusParms.dwItem = MCI_STATUS_LENGTH;
    DWORD flags = MCI_STATUS_ITEM|MCI_WAIT ;
     if(m_ErrorCode = mciSendCommand(m_wDeviceID, MCI_STATUS,flags,
        (DWORD_PTR)&mciStatusParms))
    {
        return false;
    }
```
测试2.29G大小的wav文件，查询wav文件采样数总数，mciStatusParms.dwReturn正确的返回值应该是1231253424（这个数字可以通过wav头文件信息计算得到），而代码中实际返回值为3378737072. mciStatusParms的内存布局如下：
```
cc cc cc cc b0 6f 63 c9 01 00 00 00  cc cc cc cc
```
小端机器，`b0 6f 63 c9`就是错误的返回值。可以看到错误返回值和正确数值仅仅是最高位不同。因此很有可能是API函数内部处理不当导致的。

同样的情况还会在使用MCI_STATUS_POSITION参数来查询正在播放的位置时出现。返回的位置信息有误。

另外，参考google讨论组中某位的意见，用xp自带的MPLAYER.EXE程序进行测试，因为据说它是完全用MCI实现的。测试结果是，用MPLAYER.EXE打开大于2G的wav文件时，时间轴信息无效，且时间长度无法显示。由此可以更加确定MCI的API在处理2G以上wav文件时有BUG。

希望能尽快得到修复。