---
tags: [游戏, 技术]
layout: post
category: tech
title: 可变长度数值编码
tagline: Varint
---
游戏的消息协议中经常会需要在前后台之间发送些数字. 这里只讨论自定义的二进制协议.  

一般的做法是根据业务需要, 分析每个数字可能的范围, 如果是[-128, 128) 之间的数字, 就发送byte; [-32,768, 32,768) 之间就发送short; [-8,388,608, 8,388,608) 之间就发送medium; [-21亿, 21亿] 之间发送int. 比这再多就是long. 如果数值一定不会是负的, 那同样的字节数可以多覆盖一倍的范围.

以上做法有几个弊端:

* 需要事先知道这个数值的范围, 问策划或者是程序员自己判断.

* 数值范围变化时需要改变多条网络协议. 所有用到这个数值的协议都需要改.

* 前后台定义协议时很复杂, 需要约定清楚每个数值的范围. 是byte? 还是short? 还是unsigned byte?

* 一个数值如果只有很小的可能性才会是个很大的数值的话, 就必须按照最大的数值算. 比如一个数值基本都是0, 只有很小的可能才会是10万左右, 那设计协议时, 这个数值必须至少按照medium发, 需要3个字节.

如果选择的数值范围小了, 没有考虑到某些极端情况下可能出现的大数值, 或者认为出现的可能太低而忽视, 就会造成bug. 轻的显示出错, 严重的就所有消息接收出错. 比如网络包大小设计用short发送, 最大65k. 正常情况不会超过这个数值. 如果服务器某条消息超过了这个长度, 75k按照short发, 客户端以为是10k大小的包. 后面的消息就全错了.

---

### Varint 可变长度数值编码  

自动根据实际发送的数值大小, 使用不用的字节数. 

* [0, 128) 之间使用1个字节

* [128, 16,384) 之间使用2个字节

* [16,384, 2,097,152) 之间使用3个字节

* [2,097,152, 268,435,456) 之间使用4个字节

* 其它int, 负数. 使用5个字节

这种编码方式能解决以上所有的问题

* 不需要实先和客户端/策划约定大小范围. 只需要区分int和long

* 策划需要改变数值范围时, 不需要修改网络协议

* 数值不需要根据其可能达到的最大值占用流量. 小数字甚至只需要占用1个字节.

不过它也有弊端

* 可能消耗更多的cpu, 因为需要更多的判断

* 可能占用更多的字节数. 比如负数一定需要5个字节; 如果一个数值的范围一定只会是0-200. 那么本来只需要1个byte就行, 现在可能会需要2个. (不过谁又能确保这个数值一定只会是0-200, 将来不会改呢?)

---

### 性能探究  

更少的字节数意味着更少的写到内存和写到网络的消耗. 我这里比较的是: 数值在不同取值范围内, `按照int固定占用4个字节写入buffer`和`按照varint写入buffer`, 所需要的cpu时间.

下面的图上, x坐标是不同的数值取值, y坐标是所需要的cpu时间

![-10m-1b](/images/2013_02_17/-10m-1b.png)

固定写int的消耗是固定的2ns. varint根据所需要字节数的增加, 需要更多的判断和更多的写内存消耗. 占用5字节时需要7ns.

![0-200](/images/2013_02_17/0-200.png)

局部放大. 数值在[0, 128) 范围内, varint只需要一次判断, 占用一个字节, 只需要1ns. 占用2个字节时, 需要3ns.

![-10m-10m](/images/2013_02_17/-10m-10m.png)

[16,385, 2,097,152) 之间, 占用3个字节, 消耗4ns. 占用4个字节时, 消耗6ns. 占用5个字节时, 消耗7ns.

综上, Varint写入的消耗很小, 几乎可以忽略不记.

---

### 源码  

以下是从java的[protocol buffer](https://code.google.com/p/protobuf/)库中提取出的. 其它语言相应的代码也可以从protocol buffer相应语言的库中寻找.

encode int

    public static void writeVarInt32(ChannelBuffer buffer, int value) {
        while (true){
            if ((value & ~0x7F) == 0){
                buffer.writeByte(value);
                return;
            } else{
                buffer.writeByte((value & 0x7F) | 0x80);
                value >>>= 7;
            }
        }
    }

encode long
    
    public static void writeVarint64(ChannelBuffer buffer, long value) {
      while (true) {
        if ((value & ~0x7FL) == 0) {
          buffer.writeByte((int)value);
          return;
        } else {
          buffer.writeByte(((int)value & 0x7F) | 0x80);
          value >>>= 7;
        }
      }
    }

decode int

    public static final int readVarInt32(ChannelBuffer buffer){
          byte tmp = buffer.readByte();
          if (tmp >= 0) {
            return tmp;
          }
          int result = tmp & 0x7f;
          if ((tmp = buffer.readByte()) >= 0) {
            result |= tmp << 7;
          } else {
            result |= (tmp & 0x7f) << 7;
            if ((tmp = buffer.readByte()) >= 0) {
              result |= tmp << 14;
            } else {
              result |= (tmp & 0x7f) << 14;
              if ((tmp = buffer.readByte()) >= 0) {
                result |= tmp << 21;
              } else {
                result |= (tmp & 0x7f) << 21;
                result |= (tmp = buffer.readByte()) << 28;
                if (tmp < 0) {
                  // Discard upper 32 bits.
                  for (int i = 0; i < 5; i++) {
                    if (buffer.readByte() >= 0) {
                      return result;
                    }
                  }
                  throw MALFORMED_VARINT;
                }
              }
            }
          }
          return result;
    }

decode long

    public static long readVarint64(ChannelBuffer buffer){
      int shift = 0;
      long result = 0;
      while (shift < 64) {
        final byte b = buffer.readByte();
        result |= (long)(b & 0x7F) << shift;
        if ((b & 0x80) == 0) {
          return result;
        }
        shift += 7;
      }
      throw MALFORMED_VARINT;
    }