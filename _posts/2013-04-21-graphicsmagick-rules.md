---
tags: [技术, osx, graphicmagick]
layout: post
category: tech
title: GraphicsMagick 比JMagick 好用太多了
---

昨天研究了半天JMagick, 无数api, 没有注释, 没有文档, 最后一次更新是09年, 怎么用?

好在今天发现了个新的操作图像的库, [GraphicsMagick](http://www.graphicsmagick.org/). 上个月刚刚更新过, 文档非常详细, 每一个参数都有解释, 所有可能的参数值各是什么意思. Mac下安装也很简单

        brew install GraphicsMagick

直接就能使用命令行来操作图片了. 命令行的具体使用文档在[这里](http://www.graphicsmagick.org/utilities.html). 比如要把一个文件夹下所有的png拼成一张大的png, 并压缩质量, 且只使用256种颜色

        gm montage colors 256 -quality 80 -background transparent *.png big.png

要把图片左右换个方向

        gm convert -flop 1.png out.png
上下换个方向
mark
        gm convert -flip 1.png out.png
无比强大, 写个批处理脚本, 能做到很多事情.

---

有个叫[im4java](http://im4java.sourceforge.net/)的库提供的api方便java程序调用这个命令行程序. 但我今天打不开, 所以还没研究.
