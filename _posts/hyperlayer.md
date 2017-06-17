---
title: The Hyperlayer Format
date: 6/16/2017
categories: [English, hyperlayer]
tags: [hyperlayer, image]
keywords: container, docker, image, block, lvm, hyperlayer
---

Hyperlayer is meant to be a generic diff/patch format for all block-level storage 
systems, including but not limited to LVM and/or QCoW2. With hyperlayer, we can 
realize a novel layered image system for block-level.

<!--more-->

### The Format

```
HYPERLAYER/1.0
<key>: <value>
<key>: <value>
<key>: <value>
<key>: <value>

D <offset> <length> <hash algorithm> <hash value>
D <offset> <length> <hash algorithm> <hash value>
D <offset> <length> <hash algorithm> <hash value>

W <offset> <length>
<data of $length*512 bytes>
W <offset> <length>
<data of $length*512 bytes>
W <offset> <length>
<data of $length*512 bytes>
...

```
In which:

1. keys and values are properties of the patch, consisting of letters (lower or 
upper), digits and underscore, but no leading digits;
2. offset and length are hex numbers with NO '0x' prefix, in unit of sector (512 bytes);
3. offsets are NOT required to be sorted;
4. new lines is a single '\n', without '\r';
5. D is short for 'depends', indicating the prior layers' range [offset, offset+length)
   has a hash value of <hash value>, by hash algorithm <hash algorithm>
6. W is short for 'write', indicating the new layer is achieved by writing data on the range.
7. Hash algorithms *must* include CRC32, others are optional, like SHA1, MD5, etc.

### Example

```
HYPERLAYER/1.0
Parent: 68044d4a72dfcf15018cfa6b4baf89361913d93d
Author: Huiba Li <lihuiba@gmail.com>
Date: Thu May 4 15:34:37 2017 +0800 

D 10 16 CRC32 df973e29
D 20 16 CRC32 406457f9

W 0 8
<data of 4096 bytes>
W 800 80
<data of 65536 bytes>
...
```

With a block-level layered image system, the containers will get hardened with LVM
based storage driver, and the virtual machines will get lightened storage driver. And
they will be able to get converged with each other, and co-exist better on a single 
platform.





