---
layout: post
title: QT动态加载UI文件注意事项
date: 2009-07-28
categories: tech
tags: [qt,ui]
---

在最新的QT4版本中(QT4.1以上）加入了动态载入UI文件的功能。使用如下:

```C++
QUiLoader loader;
QFile file("calculator.ui");
file.open(QFile::ReadOnly);
QWidget *formWidget = loader.load(&file, this);
file.close();

QMetaObject::connectSlotsByName(this);

QVBoxLayout *layout = new QVBoxLayout;
layout->addWidget(formWidget);
setLayout(layout);

setWindowTitle(tr("Calculator Builder"));
```

以上代码是放在从QWidget派生的自定义类中的构造函数中的。这样调用自定义类的show函数就会显示用designer设计好的界面。

经过使用和观察，发现这个动态加载对UI文件是有限制要求的，不过在官方文档中并未找到相关说明，因此也只能是作为一种总结了。也许官方正在打算改进。

限制1：UI必须是QWidget窗体或QFrame部件，不能是其他类型，如QDialog，QMainWindow

限制2：UI的顶层窗体必须具有布局，也就是窗体的布局不能是“打破布局”这一项。

如果不满足上述两个限制，QUiLoader是不能加载这样的UI文件的。