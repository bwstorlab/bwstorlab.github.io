---
title: "BlueStore(2)--Pain of FileStore"
categories:
  - blog
tags:
  - Ceph
  - BlueStore
  - FileStore
  - journal
  - WAL
---


**FileStore** is the most widely-used **ObjectStore** in Ceph.However,the journal used in FileStore may become a bottleneck for writing.

The journal can be used in different modes by the FileStore.Commonly used modes are `writeahead` (ext4, XFS) and `parallel` (Btrfs).
Other modes are: `No journal` (discouraged), `Trailing` (Quote from the docs: Deprecated, never use.)

### No Journal

Without a journal transactions are immediately scheduled.oncommit acknowledgements are fired after a sync.

Acknowledgements are therefore done in bulk when the FileStore decides to sync.

![](http://irq0.org/articles/ceph/_img/ceph_journal_no.png)

Becuase there is no filesystems that provide `atomic` writes/updates and given that `O_ATOMIC` never made it into the Kernel.So a hardware failure when doing some overwrites using this mode may lead to dirty data.
![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226517481.png)

### Writeahead

Write transactions to journal and once it is committed schedule the transaction and acknowledge.

This mode is for write in place filesystems like XFS and ext4.

A file on disk stores commit_op_seq, the sequence number of the journal entry from which a replay would start. It is incremented on every sync.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226517516.png)

When the system starts up,it first read the journal,and `redo` these entries that has not been commit.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226517527.png)

Sometimes,journal entries may be incorrect,Ceph use a CRC32C checksum to verify the correctness of the journal entries.


![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226517537.png)

### Parallel
Journal and schedule transactions at the same time. This mode is designed for copy on write filesystems like Btrfs. They provide a stable snapshot to rollback to.


![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226518718.png)

On a journal replay the current, dirty filesystem is rolled back to the previous snapshot. This snapshot plus the journal entries would then lead to a consistent state.

![](http://cezvf.img47.wal8.com/img47/544731_20160503164529/146226517499.png)

### Tailing

Trailing is a deprecated mode that first executes the transaction and then adds it to the journal.

### Penalty
The biggest problem of FileStore may be the journal penalty,for each write operation,we have to write twice:first into the journal,then to the file storing the Objects.A researcher did some test of writing with the journal is stored on the same disk as the osd data:

```bash
Device:             wMB/s
sdb1 - journal      50.11
sdb2 - osd_data     40.25
```
Traditional enterprise SATA disks can support about 110 MB/sec for a sequential write.So the penalty is about half.

In the past,we can only using an individual SSD disk to hold the journal to get better performance,now, we have another choice -- **BlueStore**.

[Ceph OSD Journal](http://irq0.org/articles/ceph/journal)

[Ceph performance: interesting things going on](http://www.sebastien-han.fr/blog/2013/12/02/ceph-performance-interesting-things-going-on/)
