---
title: "BlueStore(3)--Source code reading"
categories:
  - blog
tags:
  - Ceph
  - BlueStore
  - overlay
  - extent
---

## Single write...?

### Basic Idea:

When we want an overwrite,the basic idea to void journal to guarantee the consistency of data is simple:write the data to another space and then change the meta data:

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628361.png)

---


![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628471.png)

### Byte extends or block extends?

If we only use byte extends to store the data of a Object,after many overwrites,they can turn into many small fragments,making it difficult to read,and use much extra space to store the meta data.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628301.png)

On the other hand,using extends that are block-aligned,can reduce the size of meta data,and improve the speed of reading.However,block extends bring another
problem--- `random small write`.

Here, small write means the length of the data to write is smaller than a block.Of cource we can apply RMW(Read-Modify-Write) or WAL(Writeahead Log) for small write,they are both inefficient.

In BlueStore,data of an Object stores in both byte extends(overlays) and block extends(extends).If there are not too many overlays,a small write written directly to an overlay.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628737.png)

Otherwise,write it into writeahead log(then asynchronously commit to disk).

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628612.png)

## Objects on disk

So,an `Object ` contains:
* Meta data (list of extends and overlays) (Rocksdb)
* Extends byte data (raw disk)
* Overlays byte data (Rocksdb)

Like inode in filesystem,the meta data of an Object is stored in a struct called Onode,which is stored in Rocksdb.Overlays byte data is also store in Rocksdb.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628802.png)

## Read operation

When reading an Object(partly),firstly,get the Onode information:first look up it in the Onode LRU cache memory,if not found,apply an Rocksdb query with the Object's name.

In Onode,stl::map is used to index the list of overlays and extends for fast location.

Logically,overlays byte data(stored in Rocksdb) covers extends byte data(on disk),and that's the reason it is called `overlay`.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628897.png)

The read process first skips the extends and overlays which are beyond the read range.Then reads the overlays and extends segment by segment to buffer.

## Object clone(COW)

If COW is set in configuration,copy the Onode and overlay byte data of the source Object for a new one,and add a `SHARE` flag to each extent.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628985.png)

## Write operation

Write operation is a bit complicate.Write process has to care serveral things:
* Head and tail
  * do overlay write or WAL?
  * some old overlays may be splitted? trimmed? removed?
  * if apply WAL and an extend has `SHARE` flag,`copy it` first
  * if have to copy,need `allocate` a new extend;
* Extents
  * overwrite extent has a `SHARE` flag?
  * these upon overlays should be removed.
  * some old extents may be splitted? trimmed? removed?
  * allocate new extent(s)

It's worth mentioning that,in BlueStore,before WAL,all the overlays have to  be writen to disk using **WAL**.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226629062.png)

## extent/overlay trim

In writing process,when writing a new extent or overlay,an old extent may be trimmed,splitted or removed:

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226628232.png)

