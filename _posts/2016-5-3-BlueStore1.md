---
title: "BlueStore(1)--an implementation of ObjectStore"
categories:
  - blog
tags:
  - ceph
  - bluestore
  - filestore
  - journal
---

The BlueStore(also called `NewStore` before) is an implementation of **ObjectStore** like **FileStore**. With the help of BlueStore, objects are now stored **directly** on the block device, **without** any filesystem interface.
### What does ObjectStore store?
In Ceph,ObjectStore stores data of Objects and Collections.

* Object
  * byte data
  * xattrs
  * omap_header,omap_entries
* Collection
  * xattrs

All **ObjectStore** objects are identified as a named object(**ghobject_t** and **hobject_t**) in a named collection (**coll_t**).**ObjectStore** operations support the `creation`, `mutation`, `deletion` and `enumeration` of objects within a collection.  Enumeration is in sorted key order (where keys are sorted by hash). **Object** names are globally unique. 

Each **object** has four distinct parts: `byte data`, `xattrs`, `omap_header` and `omap entries`. The `data` portion of an object is conceptually equivalent to a file in a file system. Random and Partial access for both read and write operations is required. The ability to have a sparse implementation of the data portion of an object is beneficial for some workloads, but not required. There is a system-wide limit on the maximum size of an object, which is typically around 100 MB.

**Xattrs** are equivalent to the extended attributes of file systems. **Xattrs** are a set of `key/value` pairs.  Sub-value access is not required. It is possible to enumerate the set of xattrs in key order.At the implementation level, xattrs are used exclusively internal to Ceph and the implementer can expect the total size of all of the xattrs on an object to be relatively small, i.e., less than `64KB`. Much of Ceph assumes that accessing xattrs on temporally adjacent object accesses (recent past or near future) is inexpensive.

**omap_header** is a single blob of data. It can be read or written in total.

**Omap** entries are conceptually the same as `xattrs` but in a different address space. In other words, you can have the same key as an xattr and an omap entry and they have distinct values. Enumeration of xattrs doesn't include omap entries and vice versa. The **size** and **access** characteristics of omap entries are very different from xattrs. In particular, the value portion of an omap entry can be quite large (**MB**s).  More importantly, the interface must support efficient range queries on omap entries even when there are a large numbers of entries.

A **collection** is simply a grouping of objects. Collections have names (**coll_t**) and can be enumerated in order.  Like an individual object, a collection also has a set of xattrs.

### How to call ObjectStore?
Like database,Ceph use transactions to support atomicity.But the transaction is a bit from db's transaction:only for `write`,not for read.

**For write operations**

---

```c++
  int queue_transactions(
    Sequencer *osr, vector<Transaction>& tls,
    TrackedOpRef op = TrackedOpRef(),
    ThreadPool::TPHandle *handle = NULL);
```
 A **Transaction** represents a sequence of primitive **mutation** operations.
 Three events in the life of a Transaction result in callbacks. Any Transaction can contain any number of callback objects (Context) for any combination of the three classes of callbacks:
 
`on_applied_sync, on_applied, and on_commit.`

The `on_applied` and `on_applied_sync` callbacks are invoked when the modifications requested by the Transaction are visible to subsequent ObjectStore operations, i.e., `the results are readable`. The only conceptual difference between `on_applied` and `on_applied_sync` is the specific thread and locking environment in which the callbacks operate. 

`on_applied_sync` is called directly by an ObjectStore execution thread. It is expected to execute quickly and must not acquire any locks of the calling environment. Conversely, `on_applied` is called from the separate Finisher thread, meaning that it can contend for calling environment locks.

NB, `on_applied` and `on_applied_sync` are sometimes called `on_readable` and `on_readable_sync`.  The `on_commit` callback is also called from the Finisher thread and indicates that all of the mutations have been durably committed to stable storage (i.e., are now software/hardware crashproof).

The operations supported by ObjectStore:

```c
      OP_NOP =          0,
      OP_TOUCH =        9,   // cid, oid
      OP_WRITE =        10,  // cid, oid, offset, len, bl
      OP_ZERO =         11,  // cid, oid, offset, len
      OP_TRUNCATE =     12,  // cid, oid, len
      OP_REMOVE =       13,  // cid, oid
      OP_SETATTR =      14,  // cid, oid, attrname, bl
      OP_SETATTRS =     15,  // cid, oid, attrset
      OP_RMATTR =       16,  // cid, oid, attrname
      OP_CLONE =        17,  // cid, oid, newoid
      OP_CLONERANGE =   18,  // cid, oid, newoid, offset, len
      OP_CLONERANGE2 =  30,  // cid, oid, newoid, srcoff, len, dstoff

      OP_TRIMCACHE =    19,  // cid, oid, offset, len  **DEPRECATED**

      OP_MKCOLL =       20,  // cid
      OP_RMCOLL =       21,  // cid
      OP_COLL_ADD =     22,  // cid, oldcid, oid
      OP_COLL_REMOVE =  23,  // cid, oid
      OP_COLL_SETATTR = 24,  // cid, attrname, bl
      OP_COLL_RMATTR =  25,  // cid, attrname
      OP_COLL_SETATTRS = 26,  // cid, attrset
      OP_COLL_MOVE =    8,   // newcid, oldcid, oid

      OP_STARTSYNC =    27,  // start a sync

      OP_RMATTRS =      28,  // cid, oid
      OP_COLL_RENAME =       29,  // cid, newcid

      OP_OMAP_CLEAR = 31,   // cid
      OP_OMAP_SETKEYS = 32, // cid, attrset
      OP_OMAP_RMKEYS = 33,  // cid, keyset
      OP_OMAP_SETHEADER = 34, // cid, header
      OP_SPLIT_COLLECTION = 35, // cid, bits, destination
      OP_SPLIT_COLLECTION2 = 36, /* cid, bits, destination
				    doesn't create the destination */
      OP_OMAP_RMKEYRANGE = 37,  // cid, oid, firstkey, lastkey
      OP_COLL_MOVE_RENAME = 38,   // oldcid, oldoid, newcid, newoid

      OP_SETALLOCHINT = 39,  // cid, oid, object_size, write_size
      OP_COLL_HINT = 40, // cid, type, bl

      OP_TRY_RENAME = 41,   // oldcid, oldoid, newoid
```

**For read operations**


---

ObjectStore has the following functions for reading,call it directly.

```
//exists -- Test for existance of object
bool exists (cid,oid);

//stat -- get information for an object
int stat    (cid,oid,st,allow_eio);

//read -- read a byte range of data from an object
int read    (cidoid,offset,len,bl,op_flags,allow_eio);

//fiemap -- get extent map of data of an object
//Returns an encoded map of the extents of an object's data portion
int fiemap  (cid,oid,offset,len,bl);

//getattr -- get an xattr of an object
int getattr (cid,oid,name,value);

//getattrs -- get all of the xattrs of an object
int getattrs(cid,oid,aset);

//list_collections -- get all of the collections known to this ObjectStore
int list_collections    (ls);

//does a collection exist?
bool collection_exists  (cid);

//collection_getattr - get an xattr of a collection
int collection_getattr  (cid,name,value,size);

//collection_getattrs - get all xattrs of a collection
int collection_getattrs (cid,aset);

//is a collection empty?
bool collection_empty   (cid);

//return the number of significant bits of the coll_t::pgid.
int collection_bits     (cid);

//list contents of a collection that fall in the range [start, end) and no more than a specified many result
int collection_list     (cid,start,end,sort_bitwise,max,ls,next);

// Get omap contents
int omap_get            (cid,oid,header,out);

// Get omap header
int omap_get_header     (cid,oid,header,allow_eio);

// Get keys defined on oid
int omap_get_keys       (cid,oid,keys);

// Get key values
int omap_get_values     (cid,oid,keys,out);

// Filters keys into out which are defined on oid
int omap_check_keys     (cid,oid,keys,out);

//Returns an object map iterator
get_omap_iterator       (cid,oid);
```
