---
layout: post
title: 强制刷新test文章
date: 2012-02-07
categories: tech
tags: [test]
comments: True
---

## h1

- list
- list2

Python脚本如下:
```Python
#!/usr/bin/env python
# coding:UTF-8

import os,sys,shutil,subprocess
from os import path
# 将dirlist中的目录列表下的文件放在一个文件词典filesdict中,
# 词典的key是dirlist中某个目录下的文件名(不含该目录名),
# value是dirlist中存在对应的key的所有目录
def getfilesfromdirs(dirlist):
    filesdict = {}
    for onedir in dirlist:
        onedir = path.realpath(onedir)  # 规范化目录名
        #处理目标路径下(含子目录下)的所有文件
        for root,dirs,files in os.walk(onedir):
            # 记录每个文件名所属的目录,同名文件则会属于多个不同目录.
            # 这样就生成了文件名到文件所在目录的倒排索引
            for onefile in files:
                # 若该文件名的键值还未创建则先创建
                onefile = os.path.join(root,onefile)[len(onedir):]
                if onefile not in filesdict:
                    filesdict[onefile] = []
                filesdict[onefile] += [onedir]
    return filesdict

def vimDirDiffN(diffdirs):
    diffbuffer = './vimdirdiffntmpbuffer'
    filesdict = getfilesfromdirs(diffdirs)
    fileslines = []
    for key,values in filesdict.iteritems():
        fileslines += ['    File: '+key+' @ ' + ','.join(values)]
    write2fileline = str(len(diffdirs))
    fileslines = '\n'.join(fileslines)
    dirsymbol = 'A'
    for onedir in diffdirs:
        dirline = '<' + dirsymbol + '>' + '=' + onedir
        write2fileline += '\n' + dirline
        fileslines = fileslines.replace(onedir,'<' + dirsymbol + '>')
        dirsymbol = chr(ord(dirsymbol) + 1)
    write2fileline += '\n\n' + fileslines
    fp = open(diffbuffer,'w')
    fp.write(write2fileline)
    fp.close()
    subprocess.Popen('gvim -c ":DirDiffN ' + path.join(os.getcwd(),diffbuffer) + '"',shell=True)

if __name__ == "__main__":
    if len(sys.argv) > 2:
        diffdirs = sys.argv[1:]
        for index in range(0,len(diffdirs)):
            diffdirs[index] = path.realpath(diffdirs[index])
        vimDirDiffN(diffdirs)
    else:
        print "Interactive Mode."
        diffdirs = raw_input('input the dirs you want to diff(in List["A","B"]): ')
        if diffdirs == '':
            diffdirs = ['f:/PYOutput/old','f:/PYOutput/new','f:/PYOutput/third_merge']
        else:
            diffdirs = eval(diffdirs)
        for index in range(0,len(diffdirs)):
            diffdirs[index] = path.realpath(diffdirs[index])
        vimDirDiffN(diffdirs)
        raw_input('press to exit.')
```

效果预览图:
![](/img/vimdiff-n-preview.png)