---
layout: post
title: 使用Python和vim插件结合让Vim支持多文件夹比较
date: 2012-02-07
categories: tech
tags: [python,vim,diff]
comment: True
---

在Vim的官方网站上有一个支持2个文件夹比较的插件DirDiff, 链接: http://www.vim.org/scripts/script.php?script_id=102. 不过仅支持2个文件夹, 我对齐进行研究并改进后让其支持多个文件夹的文件进行比较.

DirDiff插件的基本原理就是先生成要比较的几个文件夹中的所有文件列表文件, 该文件的每一行对应于一个文件以及它所在的文件夹. 启用文件夹比较模式时, 会载入这个列表文件, 当选中一行时会解析出文件路径, 然后以diff模式分别打开这个文件在不同文件夹下对应的文件进行比较.

这里为了方便, 我就使用Python来生成这个特定格式的文件列表, 然后启动vim 并启动文件夹插件比较.
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
这个脚本就是读入用户输入需要比较的文件夹列表, 然后扫描文件生成一个比较列表, 启动vim并执行我修改后的多文件夹比较插件DirDiffN.

修改后的DirDiffN插件比较多, 可以从附件下载 [DirDiffN](/resource/DirDiffN.vim.zip)

将DirDiffN放入Vim的插件目录下即可.

效果预览图:
![](/img/vimdiff-n-preview.png)