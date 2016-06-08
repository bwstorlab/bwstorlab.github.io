---
layout:     post
title:      CEPH CRUSH algorithm source code analysis
date:	    2016-5-19
summary:    CRUSH algorithm analysis
categories: Original
thumbnail:  cogs
tags:
 - CEPH
 - CRUSH
 - GDB
---

## Before all
Before reading this article, you need to be familiar with CEPH's basic operations on Pools and CRUSH maps, and have a preliminary reading about the source code. (welcome to my [csdn blog](http://blog.csdn.net/snow168rain/article/details/51612364)~~~)

## Analysis method
Firstly, we write an example c [code](https://github.com/PinkGabriel/CEPH_related/blob/master/librados_example/rados_write.c) invoking  **librados** to write an object to a pool. Then we use GDB(CGDB is recommended) to trace the program running meanwhile we'll focus on some variables related to CRUSH. So we'll know how CRUSH exactly runs in the source code.

## Contents

* How to trace  
  * Compile CEPH  
  * Get function stack  
* Tracing process  
  * input ---> PGID  
  * PGID ---> OSD set  

## 1. How to trace

### 1.1 Compile CEPH
First of all, we should install CEPH using source code, and when doing `./configure`, we'll add compile flags like this `./configure CFLAGS='-g3 –O0' CXXFLAGS='-g3 –O0'`. `-g3` means MACRO infos is generated. `-O0` is **important**, it means shutdown the compiler optimization, if not, when using GDB following the program, most variables are optimized out. After configure, make and sudo make install.
P.S. `-O0` only suits for **experimental** occasion, in production environment compiler optimization surely should be on.

### 1.2 Get function stack
As we know, CRUSH core function is `crush_do_rule`(mapper.c line 785). By this function, we can divide the whole CRUSH calculation process into 2 periods: **input -> PGID** and **PGID -> OSD set**. For the first period, we use GDB to get the function stack:

1. compile the example code with -g flag
2. gdb [rados_write](https://github.com/PinkGabriel/CEPH_related/blob/master/librados_example/rados_write.c)
3. then we enter the GDB interface. Before run we add a breakpoint:`b crush_do_rule`
4. `r` and then we stop at the crush_do_rule function.
5. `bt full` we'll get the function stack and we can print the debug info out to a file using GDB log. Then let's look deep into this process.

The function stack is like this:

```
#12 main
#11 rados_write
#10 librados::IoCtxImpl::write
#9 librados::IoCtxImpl::operate
#8 Objecter::op_submit
#7 Objecter::_op_submit_with_budget
#6 Objecter::_op_submit
#5 Objecter::_calc_target
#4 OSDMap::pg_to_up_acting_osds
#3 OSDMap::_pg_to_up_acting_osds
#2 OSDMap::_pg_to_osds
#1 CrushWrapper::do_rule
#0 crush_do_rule
```

## 2. Tracing process
The CRUSH calculation can be summarized like this:
**INPUT**(*object name & pool name*) ---> **PGID** ---> **OSD set**. In this article, we only focus on the calculation.

### 2.1 input ---> PGID
All the funtions in the stack above do this work. You can read the source code following the orders. As a result, I'll list all the **crucial** transformations from input to pgid.

First in `rados_ioctx_create`, `lookup_pool` get **poolid** by **pool name** and encapsulate poolid into a `librados::IoCtxImpl` type variable `ctx`;

Then in `rados_write` **object name** is encapsulated into **oid**;
And then in `librados::IoCtxImpl::operate`, **oid** and **oloc**(comprising **poolid**) are packed into a `Objecter::Op *` type variable **objecter_op**;

Through all kinds of encapsulations, we arrive at this level: `_calc_target`. We get still unchanged **oid** and **poolid**. And we read out the informations of the target **pool**. 

![oid&oloc](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c2.png) 
![pool](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c3_2.png) 

(in my cluster, pool "neo" id is 29, name of object to write is "neo-obj") 

In `object_locator_to_pg`, the **first calculation** begins: `ceph_str_hash` hashes object name into a `uint32_t` type value as so-called `ps`(placement seed) 

![oidhash](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c6_2.png) 

Then we get **PGID**. Not long ago, I think pgid is a single value while it's not. **PGID** is a struct type variable comprising **poolid** and **ps**. 

![pgid](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c7.png) 

But what is the input **x** of `crush_do_rule`? Let's move on. Then in `_pg_to_osds` there is a line `ps_t pps = pool.raw_pg_to_pps(pg); //placement ps`. the **pps** is **x**. How is **pps** calculated? In this function: `crush_hash32_2(CRUSH_HASH_RJENKINS1,ceph_stable_mod(pg.ps(), pgp_num, pgp_num_mask),pg.pool());` 

![hash32_2](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c9_2.png)

`ps` **mod** `pgp_num_mask` and the result(i.e. `a`) hashes with **poolid**(`b`). That is what we call `pps`, i.e., **x**.

![x](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c11_2.png)

so we get the input **x** of the second period.
I draw a flow chart to show the first period.

![first period](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c13.png)

P.S. you can find something in PG's name and object name.

### 2.2 PGID ---> OSD set

Before we get started in this part, we must make clear several concepts.

*weight* VS *reweight*

![weight&reweight](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c15.png)

[ref](http://cephnotes.ksperis.com/blog/2014/12/23/difference-between-ceph-osd-reweight-and-ceph-osd-crush-reweight) here. 
“**ceph osd crush reweight**” sets the CRUSH weight of the OSD. This weight is an arbitrary value (generally the size of the disk in TB or something) and controls how much data the system tries to allocate to the OSD.
“**ceph osd reweight**” sets an override weight on the OSD. This value is in the range 0 to 1, and forces CRUSH to re-place (1-weight) of the data that would otherwise live on this drive. It does *not* change the weights assigned to the buckets above the OSD, and is a corrective measure in case the normal CRUSH distribution isn’t working out quite right. (For instance, if one of your OSDs is at 90% and the others are at 50%, you could reduce this weight to try and compensate for it.)

*primary-affinity*

Primary affinity is 1 by default (i.e., an OSD may act as a primary). You may set the OSD primary range from 0-1, where 0 means that the OSD may NOT be used as a primary and 1 means that an OSD may be used as a primary. When the weight is < 1, it is less likely that CRUSH will select the Ceph OSD Daemon to act as a primary.

*PG* VS *PGP*

pg-num is the number of PGs, pgp-num is the number of PGs that will be considered for placement, i.e. it's the pgp-num value that is used by CRUSH, not pg-num. For example, consider pg-num = 1024 and pgp-num = 1. In that case you will see 1024 PGs but all of those PGs will map to the same set of OSDs. When you increase pg-num you are splitting PGs, when you increase pgp-num you are moving them, i.e. changing sets of OSDs they map to.
PG and PGP are important concepts. More discuss can be seen [here](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2015-May/001610.html)

After knowing these concepts, let's begin the second part: PGID -> OSD set. Now we are at `do_rule`:
```
void do_rule(int rule, int x, vector<int>& out, int maxout, const vector<__u32>& weight)
```
Let's see some parameters in runtime. `x` is the `pps` we've got, rule is the crushrule's number in memory(not ruleid, in my crushrule set, this rule's id is 3), `weight` is **reweight** we've mentioned and it's scaled up from 1 to 65536. Then we define `rawout[maxout]` to store OSD set, `scratch[maxout * 3]` for calculation use. Then we go into **crush_do_rule**.

![2nd](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c16.png)
![2nd](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c17.png)

#### **PGID -> OSDset OUTLINE**
Next, we'll look into 3 functions below. BTW, `firstn` means replica storage, CRUSH need to select n osds to store these n replicas. `indep` means erasure code storage. We only focus on replica method. 

* **`crush_do_rule`**: do crushrules **iteratively**
* **`crush_choose_firstn`**: choose buckets or devices of specified type **recursively**
* **`crush_bucket_choose`**: **directly** choose a son of the input bucket

#### **crush_do_rule**
First here is my crushrule of the target pool and my cluster hierarchy:

![clusterinfo](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c19.png)

what is worth mentioning, **step emit** is typically used at the end of a rule, but may also be used to pick from **different trees** in the same rule. More detailed can be seen at offical [site](http://docs.ceph.com/docs/master/rados/operations/crush-map/#crush-map-rules).
In this function there are several important variables: `scratch[3 * result_max]` and `a`, `b`, `c` points to 0, 1/3, 2/3 locations of scratch array. And make `w = a`, `o = b`. `w` is used as a FIFO queue for taking a BFS traversal in CRUSH map. `o` stores the results of `crush_choose_firstn`. `c` stores the final OSD set result. After each `crush_choose_firstn`, if the results are not OSD, `o` exchanges with `w`. So `w` would be the input of the next call of `crush_choose_firstn`.
As mentioned, crush_do_rule does crushrules **iteratively**. You can see the rules in memory:

![rules](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c21.png)

step 1 put root rgw1 in `w`(enqueue);

step 2 would run `crush_choose_firstn` to choose 1 rack-type bucket from root rgw1.

Let's step into `crush_choose_firstn`.

#### **crush_choose_firstn**
This function chooses buckets or devices of specified type **recursively** and would deal with collision, failure occasion.
if the step is a **choose** step, the function would call `crush_bucket_choose` to do the **direct** choose; if the step is a **chooseleaf** step, the function would run recursively until it gets leaf nodes.

#### **crush_bucket_choose**
This is the ***most important*** function in CRUSH. Because default bucket type is **straw** and in most occasions we would use straw bucket, we'll look into `bucket_straw_choose`:
call:
```
case CRUSH_BUCKET_STRAW:
	return bucket_straw_choose((struct crush_bucket_straw *)in, x, r);
```
definition: 
```
static int bucket_straw_choose(struct crush_bucket_straw *bucket,
			       int x, int r)
{
	__u32 i;
	int high = 0;
	__u64 high_draw = 0;
	__u64 draw;

	for (i = 0; i < bucket->h.size; i++) {
		draw = crush_hash32_3(bucket->h.hash, x, bucket->h.items[i], r);
		draw &= 0xffff;
		draw *= bucket->straws[i];
		if (i == 0 || draw > high_draw) {
			high = i;
			high_draw = draw;
		}
	}
	return bucket->h.items[high];
}
```
Let's see the variables in runtime:

![straw](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c22_2.png)

We can see, the bucket `root rgw1`'s id is -1, `type = 10` means root, `alg = 4` means straw type. Here the `weight` is the OSD weight we set scales up by 65536(i.e. 37 * 65536 = 2424832).
Then let's look into the loop:
For **each** son bucket of the input bucket, for instance in the picture above, `rack1`, `crush_hash32_3` hashes `x`, `bucket id`(rack1's id), `r`(current selection's order number), these 3 variables into a `uint32_t` type value, then the result **&** `0xffff`, and then multiplies by `straw`(rack1's straw value, straw calculation seen below), finally we get this value, in one loop, for one son bucket(rack1 here). We calculate every son bucket all over the loop and pick the biggest. So a son bucket has been selected. Nice job! Here is the instance calculation process:

![straw_cal](http://o7dj8mc3t.bkt.clouddn.com/blog_crush/c24.png)

So bucket -16 is selected, even its straw value is a little smaller.

#### **Conclusion**
We've looked into every important part of CRUSH calculation process. The rest is iteration and iteration, recursion and recursion, until we've selected all the OSDs. See, it's simple.

#### **About the straw value**
Detailed code can be seen in src/crush/builder.c `crush_calc_straw`. Anyhow, straw value is positive relation to the OSD weight. And [straw2](http://www.spinics.net/lists/ceph-devel/msg21635.html) is being developped.

Share your ideas with me <ustcxjy@gmail.com>~
