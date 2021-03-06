---
id: 528
title: On Raft, briefly
date: 2013-10-31T12:03:51+00:00
author: Henry
layout: post
guid: http://the-paper-trail.org/blog/?p=528
permalink: /on-raft-briefly/
categories:
  - Uncategorized
---
[Raft](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf) is a new-ish consensus implementation whose great benefit, to my mind it, is its applicability for real systems. We briefly discussed it internally at Cloudera, and I thought I'd share what I contributed, below. There's an underlying theme here regarding the role of distributed systems research in practitioners' daily work, and how the act of building a distributed system has not yet been sufficiently well commoditised to render a familiarity with the original research unnecessary. I think I'd argue that bridging that gap further is necessary: no matter how much fun it is to read all these papers, it shouldn't be a pre-requisite to being successful in implementing a distributed system. I have more to write on this.

> "The trouble with Paxos is that it's 'only' a consensus algorithm; a theoretical achievement but not one necessarily suited to building practical systems. Remember that the demonstration that a correct, message-optimal protocol even existed was the main contribution. To that end, a lot of practical considerations were left by the wayside. Leader election is an exercise for the reader (since Paxos is robust to bad implementations where there are several leaders, it doesn't matter what election scheme is used). Paxos is not concerned with 'logs' at all; that it can be used to build replicated-state machines with durable logs is a corollary, not the main theorem. 
> 
> Raft fills in a ton of these gaps, and more power to them for doing so. The leader election algorithm is set in stone. There are additional constraints to ensure that updates are seen and processed in hole-free order (Paxos doesn't guarantee this), which is exactly what you want from a distributed log. Raft also specifies a view-change algorithm, which Paxos does not, but VS replication does. The huge effort required to get [ZOOKEEPER-107](https://issues.apache.org/jira/browse/ZOOKEEPER-107) committed shows how hard this is to retrofit onto an existing system.
> 
> So: there's a tendency to conflate 'distributed replicated <blah> with strong consistency properties' with 'consensus algorithm'. Consensus shows you can agree on a single value, multi-Paxos shows you can agree on a bunch of them, but neither give you a complete system for a replicated log which is actually what most of our distributed systems want to interact with."
