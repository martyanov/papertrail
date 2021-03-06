---
id: 568
title: 'Paper notes: DB2 with BLU Acceleration'
date: 2014-05-14T18:02:15+00:00
author: admin
layout: post
guid: http://the-paper-trail.org/blog/?p=568
permalink: /paper-notes-db2-with-blu-acceleration/
categories:
  - Paper notes
---
### DB2 with BLU Acceleration: So Much More than Just a Column Store</a> _Raman et. al., VLDB 2013_

**The big idea**: IBM's venerable DB2 technology was based on traditional row-based technology. By moving to a columnar execution engine, and crucially then by taking full advantage of the optimisations that columnar formats allow, the 'BLU Acceleration' project was able to improve read-mostly BI workloads by a 10 to 50 times speed-up. <!--more-->BLU is a single-node system which still uses the DB2 planner (although changed to generate plans for new columnar operators). This paper describes the basic workings of the execution engine itself. Of interest is the fact that the row-based execution engine can continue to co-exist with BLU (so existing tables don't have to be converted, pretty important for long-time DB2 customers). Not much is said about the overall execution model; presumably it is a traditional Volcano-style architecture with batches of column values passed between operators. Neither is much said about resource management: BLU is heavily multi-threaded, but the budgeting mechanism for threads assigned to any given query is not included.

### On-disk format:

Every column may have multiple encoding schemes associated with it (for example, short-code-word dictionaries for frequent data, larger-width code-words for infrequent values; key is that code-words are constant-size for a scheme). Columns are grouped into column groups; all values from the same column group are stored together. Column groups are stored in pages, and a page may contain one or more regions. A single region has a constant compression scheme for all columns stored within it. Within a region, individual columns are stored in fixed-width data banks. Finally, a page contains a tuple map which is a bitset identifying to which region each tuple in the page belongs. If there are two regions in a page, the tuple map has one bit per tuple, and so on. Each page contains a contiguous set of 'tuplets' (projection of a tuple onto a column group), so the tuple map is dense. All column groups are ordered by the same 'tuple sequence number'. The TSN to page mapping is contained in a B+-tree.

Nullable columns are handled with an extra 1-bit nullable column.

Each column has a synopsis table associated with it which allows for page-skipping, and is stored in the same format as regular table data.

### Scans:

There are two variants of on-disk column access. The first, 'LEAF', evaluates predicates over columns in a column group. It does so region-by-region, producing a bitmap with a 1 for every tuplet which passed evaluation for every region. The bitmaps are then interleaved using some bit-twiddling magic to produce a bitmap in TSN order. It's not clear if the only output from LEAF is the validity bitmap. Predicate evaluation can be done over coded data, and using SIMD-efficient algorithms.

The second column access operator, 'LCOL', loads columns either in coded or uncompressed format. Coded values can be fed directly into joins and aggregations, so it can often pay not to decompress. LCOL respects a validity bitmap as a parameter, and produces output which contains only valid tuplets.

### Joins:

Joins follow a typical build-probe two phase pattern. Both phases are heavily multi-threaded. In the build-phase, join keys are partitioned. Each thread partitions a separate subset of the scanned rows (presumably contiguous by TSN). Then each thread builds a hash-table from one partition each. Partitions are sized according to memory (in order to keep a partition in a single level of the memory hierarchy).

Joins are performed on encoded data, but since the join keys may be encoded differently in the outer and the inner, the inner is re-encoded to the outer's coding scheme. At the same time, [Bloom filters](http://en.wikipedia.org/wiki/Bloom_filter) are built on the inner, and later pushed down to the outer scan. Interestingly, the opposite is also possible: if the inner is very large, the outer is scanned once to compute a Bloom filter used to filter the inner. This extra scan is paid for by not having to spill to disk when the inner hash table gets large. Sometimes spilling is unavoidable; in that case either some partitions of the inner and outer are spilled or, depending on a cost model, only the inner is spilled. This has the benefit of not requiring early materialisation of non-join columns from the outer.

### Aggregations:

Aggregations are also two-phase. Each thread creates its own local aggregation hash-table based on a partition of the input. In the second phase, those partitions are merged: each thread takes a partition of the output hash table and scans every local hash-table produced in the first phase. In this way, every thread is writing to a local data-structure without contention in both phases. Therefore there must be a barrier between both phases, to avoid a thread updating a local hash table while it's being read in phase 2.

If a local hash table gets too large, a thread can create 'overflow buckets' which contain new groups. Once an overflow bucket gets full, it gets published to a global list of overflow buckets for a given partition. Threads may reorganise their local hash table and corresponding overflow buckets by moving less-frequent groups to the overflow buckets. A recurring theme is the constant gathering of statistics to adapt operator behaviour on the fly.
