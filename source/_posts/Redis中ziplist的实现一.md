---
title: Redis中ziplist的实现(一)
date: 2021-07-12 00:08:55
tags: [Redis]
categories: [数据库,Redis]
---
ziplist 是一个双向链表，但是不同于熟知的利用头指针和尾指针所形成的双向链表，ziplist 而是采用了一个特殊的实现

# 区别
普通的双向链表因为在每一个节点上都含有一个头节点和尾节点，所以在遍历的时候可以依次沿着每一个节点进行递归，而且在新增或者删除的时候，直接移动指针即可，并且每一个节点在内存中并不要求是连续的，只需要指针指向对应节点的内存地址即可。

但是 redis 使用的是内存，在内存满了的情况下就会造成 redis 的不可用，需要依据淘汰策略清理内存。

试想下如果 ziplist 像普通的双向链表一样，每一次新增一个 entry 都去申请一块新的内存，实现起来确实比较简单。

但是由于每一次的 value 大小都是不一样的，每一次都去申请一块新的内存，很容易造成很多的内存碎片。
现象之一就是，如果后续需要新增一个 value，却发现内存中还有很多内存碎片，但是大小都比当前 value 小，造成了 redis 不得不提前进行淘汰


所以 redis 作者在 Hash、zset、set 这三种结构中加入了一个额外的逻辑，如果数据量小则会使用 ziplist 作为其数据的存储结构



# ziplist
ziplist 会在初始化的时候直接分配一整块内存，默认初始化的大小为 `ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE`，而这两个值的大小分别是 (4 * 2 + 2) + 1 = 11字节

## 结构
在 ziplist.c 文件的顶部注释中，其结构如下：
>  `<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>`

* zlbytes
用来计算整个 ziplist 大小，主要用于 resize，大小为 4bytes

* zltail
这个表示 ziplist 从第一个字节到最后一个 entry 的偏移量，主要用于在尾部新增元素使用

* zllen
表示整个 ziplist 中 entry 的数量，最大值为 2^16-1，

* zlend
作为一个特殊的尾节点，大小为1个 byte，这个节点存在的意义就是方便进行尾插

其中 zltail 可以理解普通链表的尾节点，


## entry
entry 的结构如下：`<prevlen> <encoding> <entry-data>`
其中 prevlen 表示上一个节点的长度，方便进行链表从尾部向头部进行遍历，在遍历的过程中，只需要移动 prevlen 就可以得到前一个 entry 地址
而 encoding 则代表该 content 的类型以及长度，按照 redis 源码的注释，整理如下：

### String
如果存储内容是 String，那么 encoding 的值会随着 string 的长度改变而改变
<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Redis%E4%B8%ADziplist%E7%9A%84%E5%AE%9E%E7%8E%B0/string.png" width=500 height=256 style="margin: 0 auto;"/>

例如：如果要存储的字符串是长度为 30 的全英文字符串（或者字节长度为30的中文）
那么这个 entry 所的结构就如下图所示
<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Redis%E4%B8%ADziplist%E7%9A%84%E5%AE%9E%E7%8E%B0/clipboard_20210715_011629.png" width=500 height=256 style="margin: 0 auto;"/>

其中 prevlen 代表上一个 entry 的长度，当从尾部进行遍历，可以直接偏移 prevlen 个长度然后直接得出上一个 entry 的内存地址。
由于存储的字符串长度小于 63，则采用「00pppppp」的方式存储字符串长度，而 30 个长度对应的 16 进制正好是 1e，所以这个字节就存储的是 1e，而后续的 data 存储的就是真正的原始内容了

> tips:一个字节=2个16进制数，=8个2进制数

### int
如果存储的内容是 int 类型，那么 ziplist 的 encoding 则会通过另一种方式进行存储
<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Redis%E4%B8%ADziplist%E7%9A%84%E5%AE%9E%E7%8E%B0/int.png" width=500 height=256 style="margin: 0 auto;"/>
其中 encoding 有 5 中类型，分别对应的是 int 的几个大小

这里还有一个小细节：
因为 ff(11111111) 这种 encoding 表示的是尾节点，但是在 f0(11110000) 至 fe(11111110) 之间还有 4 个 bit 没有被使用，分别是 0001 至 1101，也就是 1 到 13，于是 redis 的作者就想，能不能把这部分的空间利用起来呢？

所以如果 int 的值处于 1-13 之间，那么此时直接和 encoding 存储在一起，避免额外开辟一个字节的空间来存储数据，减少内存的使用


