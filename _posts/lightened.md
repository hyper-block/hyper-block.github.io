---
title: Towards lightened VM Images
date: 5/31/2017
categories: [English, image]
tags: [VM, image, hyperlayer]
keywords: hypervisor, vmm, image, qemu, qcow2, hyperlayer
---

VM Images are too heavy to move around. We propose a layered image
system, to enable incremental transferring of images. This system
imitates the agile image system of container, but in the level of 
block store, instead of file system. 

<!--more-->

### Heavy Images

I have been working on cloud for a couple of years, and from time
to time I see difficulties with large VM images. Last week, a cusomter
gave us a ticket saying they had a problem when creating a new image.
After inspection, I found that the creating process was still going
on, as the image was so big (about 500GB) that the process was not
able to get completed within a preset time-out period (one hour).

The root cause of this incident is the heavy weight image. The size of
a VM image is usually several GBs at least, and can grow to 10s of GBs
easily, and it's not uncommon to see 100s of GBs. Such big size is a
major obstacle to transferring and operating VM images.

On the other hand, container images are much more light-weight. Each
time, only needed incremental layers are transferred, which can be as
small as a few MB. This is the key advantage of docker ---- agility.

### Lightening the Images

Actually, VM can learn from the image system of docker. Layering is 
actually a subset of a long-established feature ---- snapshot. Each
snapshot stores the differential data blocks compared with its parent
snapshot ---- roughly the same model with the layering model of 
container image. The major distinction is block-level vs file-level,
which is only an implementaion detail.

The most important missing part of block-level snapshot is a set of tools 
to export and import layers (snapshots). There was little need for
this kind of tools, until container appears. We have developed these
tools for qcow2 images, as well as LVM.

The figure below shows the model for QCoW2 images: 
(1) we suppose backing file plays the role of image, which serves all 
    the dependent layers as read-only; 
(2) and the "fronting" file plays the role of staging layer as read-write; 
(3) every layer is essentially a snapshot.

![Layered Image](/images/layered-image.png)

The two files ("fronting" and backing) in this model can not be 
integrated into one, otherwise the image layers can not be shared by 
multiple "fronting" files.

### Exported Format of Layers

The layers are exported as HYPERLAYER format:

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
In which:

1. keys and values are properties of the patch, consisting of letters (lower or 
upper), digits and underscore, but no leading digits;
2. offset and length are hex numbers with NO '0x' prefix, in unit of sector (512 bytes);
3. offsets are NOT required to be sorted;
4. new lines is a single '\n', without '\r';

For example:

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

This format is called *HyperLayer*. It‘s meant to be a generic diff/patch format 
for all block-level storage systems, including but not limited to LVM and/or QCoW2. 
With this kind of patch file, we are able to realize a block-level image system 
that is similar to docker's.



