---
tags: [技术, osx, jmagick]
layout: post
category: tech
title: OSX下安装JMagick
---

[Magick](www.imagemagick.org/)是个非常强大的图片处理库, java下也有使用jni调用它的binding: [JMagick](www.jmagick.org/). 和jni相关的东西总是很麻烦.

1. 安装Magick (事先已安装[Homebrew](http://mxcl.github.io/homebrew/))

        brew install imagemagick

2. 下载JMagick包, [下载地址](http://downloads.jmagick.org/). 我因为要使用[另一个库](https://github.com/haraldk/TwelveMonkeys/tree/master/imageio/imageio-jmagick)中的类, 这个库用了6.2.4版, 我也就下载安装了这个版本

        curl -O http://downloads.jmagick.org/6.2.4/JMagick-6.2.4-1.tar.gz
        tar xfvz JMagick-6.2.4-1.tar.gz
        cd JMagick-6.2.4-1
        // 需要jni.h包头, 我用1.7带的不行, 用1.6的就行
        ./configure --with-java-includes=/System/Library/Frameworks/JavaVM.framework/Headers/
        make
        sudo make install

3. lib下生成的竟然是个.so的, 改名成mac下用的dylib
        
        sudo cp lib/libJMagick.so /usr/local/lib/libJMagick.dylib

4. java下使用编译出来的lib/jmagick.jar或者用maven里的6.2.4版, 加jvm参数启动 -Djava.library.path=/usr/local/lib