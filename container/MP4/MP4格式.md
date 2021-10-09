MP4 简明介绍

本文只给出关于 MP4 的简要说明，一些诸如 box 语法等前提知识，请参考网络博客进行预学习。

深入了解，足够阅读能力：ISO/IEC 14496-12 ISO base media file format

简要了解：

* https://www.jianshu.com/p/7aa6b20f7cb7
* https://www.filefix.org/format/mp4.html

## 1. 准备

MP4 基于 box， box 定义如下：

```c
struct box {
    int size; // 整个 box 的大小，此致为0，表示整个文件都属于此box
    int type; // box 的类型，由四个字符构成，比如：”ftyp”
    int *body; // box 的有效内容，body 的大小为 size - 8
}
```

还存在扩展的box，即 size 为 1 时，上述 body 的前 8 bytes为 largesize，表示实际 size ，此举意在支持大于4G 的 box。本文忽略。

MP4在形式上表现为 box 包含子 box，比如 `moov` 中，包含了所有的非媒体数据相关box。

## 布局

对于一个典型的mp4 而言，其典型布局如下：

![基本布局](./container/MP4/assets/layout_brief.png)

`ftyp` box 表示此文件的format 形式为MP4。`moov` 包含了全局信息，也可以认为是媒体数据的描述信息，用于说明 `mdat` 媒体数据。 `mdat` box 则是纯粹的 Audio/Video/Subtitle 数据信息。

需要注意的是，"ftyp" 是确定MP4的关键字符。

下图是一个真实的MP4结构：

![sample layout](./container/MP4/assets/real_layout.png)

可以看出其是嵌套的层次结构。

## MOOV 中的数据信息

`moov` 包含的主要信息由下图列出：

![moov info](./container/MP4/assets/moov_info.png)

**注1：**

通过 chunk 的偏移地址 + sample 的size 能后获得特定sample的绝对存储位置，即 `stsc` + `stco` 能 获得sample的空间信息，`stts` + `ctts` 能获得sample 的时间信息

**注2：**

黑色表示强制出现的

红色字体或虚线 表示特定情况出现

蓝色为备注

**注3**: 

`sdtp` 是一个optional box, 如果其出现，其为每个 sample 表明了一些特征：

```c
is_leading 		// 0: leading 属性未知； 1: 在 I帧 之前有依赖的 leading 帧(因此不能解码)

			   // 2: 非 leading picture    3: 在 I帧 之前没有依赖的 leading 帧(因此能解码)

sample_depends_on 	  // 0: 此 sample 的依赖性未知； 1: 此sample 依赖其他sample (non-I frame)

					// 2: 此sample 不依赖其他 sample (I frame); 3: 保留

sample_is_depended_on 	// 0: 其他 sample 是否依赖此sample 未知； 1: 其他 sample 可能依赖此sample(非自由的)

					  // 2: 其他 sample 不依赖此sample (自由的); 3: 保留

sample_has_redundancy 	  // 0: 此 sample 是否冗余编码未知； 1: 此sample 冗余编码

						// 2: 此 sample 没有冗余编码；3:保留
```

当执行诸如快进的“trick-mode”时，可以通过 sample 是否依赖其他 sample 的特点来定位可独立解码的 sample。类似地，当执行随机访问时，可能需要定位先前的同步sample或随机访问恢复点，并从同步 sample 或随机访问恢复点的 pre-roll starting point 前滚到所需的点。在前滚时，不需要检索或解码没有其他依赖的sample。

 

**注4：**

​     sample_flags 是u32大小的结构：

 ```c
 bit(4)
 
 reserved=0;
 
 unsigned int(2) is_leading;
 
 unsigned int(2) sample_depends_on;
 
 unsigned int(2) sample_is_depended_on;
 
 unsigned int(2) sample_has_redundancy;
 
 bit(3)
 
 sample_padding_value;
 
 bit(1)
 
 sample_is_non_sync_sample; // 提供了 stss box 相同的作用
 
 unsigned int(16) sample_degradation_priority;
 ```

**注5:** 

leading sample 也即 leading picture，是 HEVC 中的图片类型。是在 DTS 在 随机访问点（帧内随机访问图像IRAP，有三种：IDR，CRA， BLA）之后, 但输出(PTS) 在随机访问点之前的图片类型。另外还有Trailing pictures，其输出和显示顺序上都跟随IRAP和 leading picture。

参考： https://www.hevcbook.de/hevc-picture-types/

通过图中说明，可以得到 track 的全部信息，timescale，duration，codec相关信息，sample 的时空信息，DRM 信息 以及 mp4 是否 fragment mp4 等。

[^注意]: 在 fragment mp4 中，图中的 sample 相关的时/空信息，可能不存在与 `moov` 中，其存在于 `moof` 中。见下面介绍。

