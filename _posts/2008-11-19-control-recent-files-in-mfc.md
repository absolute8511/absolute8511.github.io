---
layout: post
title: VC中MFC程序手动控制最近文件列表
date: 2008-11-19
categories: tech
tags: [vc,mfc]
---

默认的MFC单文档程序可以支持最近的文件列表，但是它却不一定是我们需要的，因此我在这里总结出手动控制的方法，以备不时之需。

默认的最近文件列表是通过MRU file list来实现的，它通过将最近打开的文件写入注册表，然后读取到菜单上实现的。这一切默认都是通过打开和保存这些菜单操作来实现。当你选择一个列表时，就会调用相应的事件响应函数。

下面是默认的操作内容：

```C++
BOOL CWinApp::OnOpenRecentFile(UINT nID)   
{   
         ASSERT_VALID(this);   
         ASSERT(m_pRecentFileList != NULL);   
     
         ASSERT(nID >= ID_FILE_MRU_FILE1);   
         ASSERT(nID < ID_FILE_MRU_FILE1 + (UINT)m_pRecentFileList->GetSize());   
        int nIndex = nID - ID_FILE_MRU_FILE1;   
         ASSERT((*m_pRecentFileList)[nIndex].GetLength() != 0);   
     
         TRACE2("MRU: open file (%d) '%s'.\n", (nIndex) + 1,   
                         (LPCTSTR)(*m_pRecentFileList)[nIndex]);   
     
        if (OpenDocumentFile((*m_pRecentFileList)[nIndex]) == NULL)   
                 m_pRecentFileList->Remove(nIndex);   
     
        return TRUE;   
}
```

现在，我要实现的是在任何情况下添加自己的文件到列表中，然后编写自己的处理函数。方法很简单，实现如下：

1 首先，在想添加文件路径的地方添加代码
```
theApp.AddToRecentFileList(fileName);
```
theApp是应用程序对象，可以通过AfxGetApp获得它的对象指针。这样就会在菜单上添加fileName的最近文件列表，注意最好是全路径名，否则下面打开操作可能会有问题。

2 然后就是重载应用程序的OpenDocumentFile操作

在你点击最近文件列表后，就会调用程序的OpenDocumentFile函数，所以在此函数中添加自己代码即可。

在CApp类中添加该虚函数后，自动创建的函数里面有一句话
```
return CWinApp::OpenDocumentFile(lpszFileName);
```

不做任何操作的话，会调用前面的OnOpenRecentFile函数，然后执行默认的操作。如果你的应用程序不支持文档操作的话，此函数就会执行失败。因此要添加自己的代码。

```C++
CDocument* CWavEditFormApp::OpenDocumentFile(LPCTSTR lpszFileName) 
{
//返回null会删除当前记录，返回当前文档则不做文档处理   
CFileFind finder;
if(finder.FindFile(lpszFileName))
{

//自己的处理代码


   return ((CFrameWnd *)this->m_pMainWnd)->GetActiveView()->GetDocument();
}
else
{
   MessageBox(NULL,_T("文件不存在！"),_T("文件打开错误"),MB_OK);
   return NULL;
}
// return CWinApp::OpenDocumentFile(lpszFileName);
}
```

上面的代码，首先检测该文件是否存在，不存在返回NULL，这样就会删除当前记录。存在的话就执行自己的代码，然后返回以前的文档，由于返回的是以前的文档，因此不会再有对文档的操作了。

通过上面2步后，你就能自己添加最近列表然后自己处理点击最近列表的处理函数了。