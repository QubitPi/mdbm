MDBM - High-speed database
==========================

**The purpose of this fork is to make _Java_ wrapper of Yahoo's [MDBM](https://github.com/yahoo/mdbm)
indexable/importable for Maven projects.**

Table of contents
=================

* [Introduction](#introduction)
    - [MDBM Stands Top on Certain Use Cases](#mdbm-stands-out-on-certain-use-cases)
    - [Speed](#speed)
    - [Data Distribution](#data-distribution)
    - [Tips](#tips)
* [How is MDBM different from Redis?](#how-is-mdbm-different-from-redis)

# Introduction

Back in 1979, AT&T released a lightweight database engine written by Ken Thompson, called DBM
(http://en.wikipedia.org/wiki/Dbm). In 1987 Ozan Yigit created a work-alike version, SDBM, that he released to the
public domain.

The DBM family of databases has been quietly powering lots of things "under the hood" on various versions of unix, such
as sendmail rulesets on an early version of linux.

A group of programmers at SGI, including Larry McVoy, wrote a version based on SDBM, called MDBM, with the twist that it
memory-mapped the data.

This is how MDBM came to Yahoo, over a decade ago, where it has also been quietly powering lots of things
"under the hood". Yahoo improved performance for some of its particular use cases, and adding lots of features (some 
might say too many).

## MDBM Stands Out on Certain Use Cases

These days, all the cool kids are saying "NoSQL", and "Zero Copy", for high performance, but MDBM has been living it for
well over a decade. From my experience, **MDBM has its top ranking on some uses cases in which trivial setup & 
extremely fast(I'm serious - faster than any other tech I've seen) key-value data search is required** 

The exact definition of "NoSQL" has gotten a bit muddy these days, now including "not-only-SQL". But at it's core, it
means optimizing the structure and interface to your DB to maximize performance for your particular application.

There are a number of things that SQL databases can do that MDBM can not. MDBM is a simple key-value store. You can
search for a key, and it will return references to the associated value(s). You can store, overwrite, or append a value
to a given key. The interface is minimal. You can iterate over the database, but there are no "joins", "views" or
"select" clauses, nor any relationship between tables or entities unless you explicitly create them.

So, if MDBM doesn't have any of these features, why would you want to use it?

1. **simplicity**
2. **raw performance**

## Speed

On hardware that was current several years ago, MDBM performed 15 million QPS for read/write locking, and almost 10
million QPS for partitioned locking. Both with latencies well under 5 microseconds.

Performance: (based on LevelDB benchmarks) Machine: 8 Core Intel® Xeon® CPU L5420 @ 2.50GHz

| Test            | MDBM    | LevelDB  | KyotoCabinet | BerkeleyDB |
|-----------------|---------|----------|--------------|------------|
| Write Time      | 1.1 μs  | 4.5 μs   | 5.1 μs       | 14.0 μs    |
| Read Time       | 0.45 μs | 5.3 μs   | 4.9 μs       | 8.4 μs     |
| Sequential Read | 0.05 μs | 0.53 μs  | 1.71 μs      | 39.1 μs    |
| Sync Write      | 2625 μs | 34944 μs | 177169 μs    | 13001 μs   |

NOTES: These are single-process, single-thread timings. LevelDB does not support multi-process usage, and many features
must be lock-protected externally. MDBM iteration (sequential read) is un-ordered. Minimal tuning was performed on all
of the candidates.

How does MDBM achieve this performance? There are two important components.

"Memory Mapping" - It leverages the kernel's virtual-memory system, so that most operations can happen in-memory.
"Zero-Copy" - The library provides raw pointers to data stored in the MDBM. This requires some care (memory leak), but
if you need the performance, it's worth it. If you want to trade the performance for safety, it's easy to do that too.

### Memory Mapping

Behind the scenes, Linux (and many other operating systems) keep often used parts of files in-memory via the
virtual-memory subsystem. As different disk pages are needed, memory pages will be written out to disk (if they've
changed) and discarded. Then the needed pages are read in to memory. MDBM leverages this system by explicitly telling
the VM system to load (memory-map) the database file. As pages are modified, they are written out to disk, but writes
can be delayed and bunched up until some threshold is reached, or the pages are needed for something else.

This means less wear-and-tear on your spinning-rust or solid-state disks, but it also makes a huge difference in
performance. Disks are perhaps an order-of-magnitude (10x) slower than memory for sequential access
(reading from beginning to end, or always appending to the end of a file). However, for random access (what most DBs
need), disks can be 5 orders-of-magnitude (100,000 times) slower than memory. Solid state disks fare a bit better, but
there's still a huge gap.

If there is a lot of memory pressure, you can "pin" the MDBM pages so that the VM system will keep them in memory. Or,
you  could let the VM page parts in and out, with some performance hit. But what if your dataset is bigger than your 
available memory? Out of the box, MDBM can run with two (or more) levels, so you can have a "cache" MDBM that keeps
frequently used items together in memory, and lets less used entries stay on-disk. You can also use "windowed-mode"
where MDBM explicitly manages mapping portions in and out of memory itself (with some performance penalty).

### Zero-Copy

Lets look at what used to be involved in sending a message out over the network:

* user assembles pieces of the message into one big buffer in memory (1st copy)
* user calls network function
* transition to kernel code
* kernel copies user data to kernel memory (second copy)
* kernel notifies device driver
* driver copies data to device (third copy)
* transition back to user space

Each one of these copies (and transitions) has a very noticeable performance cost. The linux kernel team spent a good
amount of time and effort reducing this to:

* user gives list of pieces to kernel (no copy)
* transition to kernel code
* kernel sets up DMA (direct-memory-access) for network card to read and send pieces
* transition back to user space

If you're connecting to a remote SQL DB over the network, you're incurring these costs for the request and the response
on both sides. If you're connecting to a local service, then you can replace the driver section with a copy to userspace
for the DB server. (This completely ignores network/loopback latency, and any disk writes for the server)

For something like LevelDB, you still have to wait to copy data for the kernel, and DMA it to the disk.
(LevelDB appends new entries to a "log" file, and squashes the various log files together as another pass over the data)

For an MDBM in steady state, you can do a normal store with the cost of one memory copy. To avoid that extra copy, you
can reserve space for a value, and update it in-place. The data will be written out to disk eventually by the VM system,
but you don't have to wait for it. NOTE: you can explicitly flush the data to disk, but for highest performance, you
should let the VM batch up changes and flush them when when there is spare I/O and cycles available.

Because the data is explicitly mapped into memory, once you know the location of a bit of data, you can treat it like
any other bit of memory on the stack or the heap.

## Data Distribution

MDBM allows you to use various hashing functions on a file-by-file basis, including FNV, Jenkins, and MD5. So, you can
usually find a decent page distribution for your data. However, it's hard to escape statistics, so you will end up with
pages that have higher and lower occupancy than other pages. Also, if your values are not uniformly-sized, then you may
have some individual DB entries that vary wildly from the average. These factors can all conspire to reduce the space
efficiency of your DB.

MDBM has several ways to cope with this:

* It can split individual pages in two, using a form of
  [Extendible Hashing](https://en.wikipedia.org/wiki/Extendible_hashing).
* It has a feature called "overflow pages" that allows some pages to be larger than others.
* It has a feature called "large objects" that allows very big single DB entries, which are over a (configurable) size 
  to be placed in a special area in the DB, outside of the normal pages.

## Tips

This all sounds great, but there are some costs of which you should be aware.

On clean shutdown of the machine, all of the MDBM data will be flushed to disk. However, in cases like power-failure and
hardware problems, it's possible for data to be lost, and the resulting DB to be corrupted. MDBM includes a tool to
check DB consistency. However, you should always have contingencies. One way or another this is some form of redundancy.

MDBM use typically falls into a few categories:

1. The DBs are cached data. So the DB can be truncated/deleted and will fill with appropriate data over time.
2. The DBs are generated in bulk somewhere (i.e. Hadoop grid), and copied to where they are used. They can be re-copied 
   from a source or peer. If they are read-only during use, then corruption is not an issue.
3. The data represents transient data (monitoring), for which it's loss is less critical.
4. The data needs to persist and is dynamically generated. We typically have some combination of redundancy across 
   machines/data-centers, and logging the data to another channel. In case of damage, data can be copied from a peer, or 
   re-generated from the logged data.
   
There is one other cost. Because MDBM gives you raw pointers into the DB's data, you have to be very careful about
making sure you don't have array over-runs, invalid pointer access, or the like.

If you do run into a problem, MDBM does provide "protected mode", where pages of the DB individually become writable
only as needed. However, this comes at a noticeable performance cost, so it isn't used in normal production.

You shouldn't let the preceding costs scare you away, just be aware that some care is required. Redundancy is always
your friend.

MDBM has been used in production for over a decade, for things both small (a few KB) and large (10s of GB). One recent 
project has DBs ranging from 5MB to 10GB spread across 1500 DBs (not counting replicas) for a total dataset size of 4
Terabytes.

When I first encountered MDBM, we had scaled out what was one of the largest Oracle instances (at the time) in about
every direction it could be scaled. Unfortunately, the serving side was having trouble expanding enough to meet latency
requirements. The solution was a tier of partitioned (aka “sharded”), replicated, distributed copies of the data in
MDBMs.

# How is MDBM different from Redis?

MDBM is more about simplicity and raw speed, whereas Redis is more for networked, distributed use, with structured data.

If you want a local store for simple key/value lookups, MDBM is easier to use. If you need it to be very fast & low 
latency, MDBM does that. MDBM can also let you update existing data, in-place, so it's very efficient for things like 
storing and updating statistics and monitoring data.

There were some benchmarking a while ago between LevelDB, KyotoCabinet, BerkeleyDB, and a few others. MDBM was the 
fastest of the bunch.

If you want to do range scans, split your data over multiple machines, or have your individual data entries treated as 
special types (lists, sets, hashes, etc), then Redis would be a better fit.

Note: The Redis documentation specifically mentions that Redis doesn't perform well with very small or very large (1kb)
 keys. MDBM does hash(and size) comparisons before doing a full key string-comparison, even within a page, so it 
 doesn't have this problem.

There isn't currently any open-source code to use MDBM as a distributed data-store. However, there has been some talk 
of providing a memcached-compatible wrapper. If that's useful, let us know. Community interest might help make it 
happen (or get it done sooner).

Examples of some things MDBM is currently used for:

- Hostname/service to IP mappings (DNS replacement/caching).
- ID to permission mappings.
- Configuration data.
- Storing large serving datasets (>4TB).
- Caching serving datasets.
- Storing monitoring data.
