---
title: 敏捷VM映像
author: 李慧霸
date: 6/5/2017
categories: [中文, image]
tags: [VM, image, hyperlayer]
keywords: hypervisor, vmm, image, qemu, qcow2, hyperlayer
---

虚拟机映像一般都比较大，因而难以搬移。针对这一问题我们提议一种分层映像系统，
以使能映像的增量传输。这一系统模仿了容器的非常敏捷的映像系统，但处于块存储级别，
而不是文件系统级别。

<!--more-->

### 沉重的映像

我在云计算领域已经工作了许多年，时不时就会遇到处理超大VM映像的问题。上一周，有一位
客户提工单说他们在创建新映像时遇到了困难。在调查之后，我发现映像的创建过程仍在持续，
因为映像非常之大（大约500GB）以至于该流程无法在预定义的时间上限（一小时）内完成。

这一问题的根源是重量级的映像。虚拟机映像通常至少有几GB大，并且可轻易增长到几十GB，
几百GB也不罕见。这么大的尺寸是传输和操作映像的主要障碍。

在另一面，容器映像则轻量得多。每一次只需要传输增量层次，数据量可以低至几MB。这是容器
的关键优势————敏捷。


### 减轻映像

事实上，VM映像可以效仿容器。层化其实是快照技术这一历史悠久的功能特性的子集。每次快照
都存储相对上一次快照的差异数据————基本上跟容器映像的分层模型相同，最主要的区别在于
块存储级别vs文件存储级别，而这可以视为一种实现细节。

块级快照最主要的缺失项是一组导入、导出映像层次（快照）的工具集。业内之前对此几乎没有需求，
直到容器出现。我们分别针对QCoW2文件以及LVM开发了这些工具。

下图展示了针对QCoW2文件的分层映像模型：
(1) 假设 backing file 用作映像，它包含所有的依赖层次，提供只读服务；
(2) "fronting" file 是可读可写的staging层;
(3) 每个层次实质上都是快照；

![Layered Image](/images/layered-image.png)

这一模型中的两个文件("fronting" and backing)不能合成为一个，否则映像中的层次
难以被多个 "fronting" files 共享。

### 导出格式

映像层次被导出为HYPERLAYER格式，如下所示:

```
HYPERLAYER/1.0
<key>: <value>
<key>: <value>
<key>: <value>
<key>: <value>

<offset> <length>
<data of $length*512 bytes>
<offset> <length>
<data of $length*512 bytes>
<offset> <length>
<data of $length*512 bytes>
...

```
其中：

1. key和value是该patch文件的属性，由大小写字母、数字和下划线组成，并且首字符不能是数字；
2. offset和length是16进制数字，没有'0x'前缀，以sector为单位（512字节）；
3. offset不需要依照某种特定顺序排列；
4. 换行符是单个'\n'，没有'\r'；

例如：

```
HYPERLAYER/1.0
Parent: 68044d4a72dfcf15018cfa6b4baf89361913d93d
Author: Huiba Li <lihuiba@gmail.com>
Date: Thu May 4 15:34:37 2017 +0800 

0 8
<data of 4096 bytes>
800 80
<data of 65536 bytes>
...
```

这种格式叫做*HyperLayer*，它被设计为一种通用的块存储系统diff/patch格式，可用于（但不限于）
LVM和QCoW2。基于这种patch文件，我们可以实现跟docker类似的基于块存储的映像系统。
