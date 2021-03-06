---
id: 590
title: 'Paper notes: Anti-Caching'
date: 2014-06-06T11:03:39+00:00
author: Henry
layout: post
guid: http://the-paper-trail.org/blog/?p=590
permalink: /paper-notes-anti-caching/
categories:
  - Paper notes
---

### Anti-Caching: A New Approach to Database Management System Architecture</a>

_DeBrabant et. al., VLDB 2013_

**The big idea**: Traditional databases typically rely on the OS page cache to bring hot tuples into memory and keep them there. This suffers from a number of problems:

* No control over granularity of caching or eviction (so keeping a tuple in memory might keep all the tuples in its page as well, even though there's not necessarily a usage correlation between them)
* No control over when fetches are performed (fetches are typically slow, and transactions may hold onto locks or latches while the access is being made)
* Duplication of resources - tuples can occupy both disk blocks and memory pages.

Instead, this paper proposes a DB-controlled mechanism for tuple caching and eviction called anti-caching. The idea is that the DB chooses exactly what to evict and when. The 'anti' aspect arises when you consider that the disk is now the place to store recently unused tuples, not the source of ground truth for the entire database. The disk, in fact, can't easily store the tuples that are in memory because, as we shall see, the anti-caching mechanism may choose to write tuples into arbitrary blocks upon eviction, which will not correspond to the pages that are already on disk, giving rise to a complex rewrite / compaction problem on eviction. The benefits are realised partly in IO: only tuples that are cold are evicted (rather than those that are unluckily sitting in the same page), and fetches and evictions may be batched.

The basic mechanism is simple. Tuples are stored in a LRU chain. When memory pressure exceeds a threshold, some proportion of tuples are evicted to disk in a fixed-size block. Only the tuples that are 'cold' are evicted, by packing them together in a new block. When a transaction accesses an evicted tuple (the DBMS tracks the location of every tuple in-memory), all tuples that might be accessed by the transaction are scanned at once in what's called a pre-pass phase. After pre-pass completes, the transaction is aborted (to let other transactions proceed) and retrieves the tuples from disk en-masse. This gives a superior disk access pattern than virtual memory paging, since page faults are typically sequentially served where tuple blocks can be retrieved in parallel. Presumably this process repeats if the set of tuples a transaction requires changes between abort and restart, in the hope that eventually the set of tuples in memory and those accessed by the transaction converge.

Since tuples are read into memory in a block, two obvious strategies are possible for merging them into memory. The first merges all tuples from a block, even those that aren't required (those that aren't required could conceivably be immediately evicted, leading to an oscillation problem). The second merges only those tuples that are accessed, but leads to a compaction problem for those tuples left behind; the paper compacts blocks lazily during merge. Since the overall approach has some efficiency problems to solve, some optimisations are proposed:

* Only sample a proportion of transactions to update the LRU chain. Hot tuples still have a relatively higher probability of being at the front of the LRU chain.
* Some tables may be marked as unevictable (small / dimension tables?) and do not participate in the LRU chain
* LRU chains are per-partition, not global across the whole systemFuture work: expand anti-caching to handle larger-than-main-memory queries, use query optimisation to avoid fetching unnecessary tuples (e.g. those covered by an index), or store only a projection of a tuple in memory, better block compaction and reorganisation schemes. _Also see: [H-Store's documentation](http://hstore.cs.brown.edu/documentation/deployment/anti-caching/)_
