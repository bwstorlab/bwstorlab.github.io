---
title: "BlueStore(4)--StupidAllocator"
categories:
  - blog
tags:
  - Ceph
  - BlueStore
  - StupidAllocator
---


Since BlueStore works without any filesystem,when writes come,new extents may be allocated.Currently BlueStore use a `StupidAllocator` to do this job.

## FreeListManager

FreeListManager just keeps a list of free spaces.Actually it uses a b-tree map(by Google) in memory for fast location other than a simple list.

For persistent storage,FreeListManager use the key-value db Rocksdb to store the free list.

## StupidAllocator

#### Multipul freelists

StupidAllocator keeps several(about 10) freelists.The length of free blocks in freelist[i] is in `[ 2^i , 2^(i+1) ) * BASE`

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146228964528.png)

#### Allocation

When we want an extent of `len` from `hint`,StupidAllocator tries to ensure the continuity of the extend to be allocated.And gives priority to ensuring it's location is after `hint`.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146228964547.png)
#### Deallocation

StupidAllocator first add the free space to the right freelist.Then,the free block with some adjacent blocks may come into a bigger one and may be move to another freelist(promote).

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146228964565.png)


