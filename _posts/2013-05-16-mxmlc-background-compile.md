---
tags: [技术, flash, swf, mxmlc, flex sdk, java]
layout: post
category: tech
title: mxmlc 后台编译
tagline: 
---

写了个服务器程序开在工作机器上, 可以把资源压缩打包成个swf文件, 在调用mxmlc压制swf文件时, 老是弹出mxmlc程序, 打断我当前的工作, 苦恼很久. 不想放弃mxmlc, 自己用java写压制逻辑. 终于被我找到个mxmlc的编译参数 `-compiler.headless-server`[^tuto], 使编译在后台运行, 不再获取focus


---

[^tuto]: [mxmlc compiler args](http://www.senocular.com/flash/tutorials/as3withmxmlc/)