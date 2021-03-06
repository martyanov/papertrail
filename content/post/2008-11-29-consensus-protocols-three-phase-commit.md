---
id: 103
title: 'Consensus Protocols: Three-phase Commit'
date: 2008-11-29T14:35:36+00:00
author: Henry
layout: post
guid: http://hnr.dnsalias.net/wordpress/?p=103
permalink: /consensus-protocols-three-phase-commit/
aliases:
  /blog/consensus-protocols-three-phase-commit/
categories:
  - computer science
  - Distributed systems
  - Note
tags:
  - 2PC
  - 3PC
  - consensus
  - Distributed systems
---
Last time we looked extensively at two-phase commit, a consensus algorithm that has the benefit of low latency but which is offset by fragility in the face of participant machine crashes. In this short note, I'm going to explain how the addition of an extra phase to the protocol can shore things up a bit, at the cost of a greater latency.

<!--more-->

The fundamental difficulty with 2PC is that, once the decision to commit has been made by the co-ordinator and communicated to some replicas, the replicas go right ahead and act upon the commit statement without checking to see if every other replica got the message. Then, if a replica that committed crashes along with the co-ordinator, the system has no way of telling what the result of the transaction was (since only the co-ordinator and the replica that got the message know for sure). Since the transaction might already have been committed at the crashed replica, the protocol cannot pessimistically abort - as the transaction might have had side-effects that are impossible to undo. Similarly, the protocol cannot optimistically force the transaction to commit, as the original vote might have been to abort.

This problem is - mostly - circumvented by the addition of an extra phase to 2PC, unsurprisingly giving us a _three-phase commit_ protocol. The idea is very simple. We break the second phase of 2PC - 'commit' - into two sub-phases. The first is the 'prepare to commit' phase. The co-ordinator sends this message to all replicas when it has received unanimous 'yes' votes in the first phase. On receipt of this messages, replicas get into a state where they are able to commit the transaction - by taking necessary locks and so forth - but crucially do not do any work that they cannot later undo. They then reply to the co-ordinator telling it that the 'prepare to commit' message was received.

The purpose of this phase is to communicate the result of the vote to every replica so that the state of the protocol can be recovered no matter which replica dies.

The last phase of the protocol does almost exactly the same thing as the original 'commit or abort' phase in 2PC. If the co-ordinator receives confirmation of the delivery of the 'prepare to commit' message from all replicas, it is then safe to go ahead with committing the transaction. However, if delivery is not confirmed, the co-ordinator cannot guarantee that the protocol state will be recovered should it crash (if you are tolerating a fixed number  \\(f\\) of failures, the co-ordinator can go ahead once it has received  \\(f+1\\) confirmations). In this case, the co-ordinator will abort the transaction.

If the co-ordinator should crash at any point, a recovery node can take over the transaction and query the state from any remaining replicas. If a replica that has committed the transaction has crashed, we know that every other replica has received a 'prepare to commit' message (otherwise the co-ordinator wouldn't have moved to the commit phase), and therefore the recovery node will be able to determine that the transaction was able to be committed, and safely shepherd the protocol to its conclusion. If any replica reports to the recovery node that it has not received 'prepare to commit', the recovery node will know that the transaction has not been committed at any replica, and will therefore be able either to pessimistically abort or re-run the protocol from the beginning.

So does 3PC fix all our problems? Not quite, but it comes close. In the case of a network partition, the wheels rather come off - imagine that all the replicas that received 'prepare to commit' are on one side of the partition, and those that did not are on the other. Then both partitions will continue with recovery nodes that respectively commit or abort the transaction, and when the network merges the system will have an inconsistent state. So 3PC has potentially unsafe runs, as does 2PC, but will always make progress and therefore satisfies its liveness properties. The fact that 3PC will not block on single node failures makes it much more appealing for services where high availability is more important than low latencies.

Next time I talk about consensus, it will be to try and describe Paxos, which is really a generalisation of 2PC and 3PC which has found massive popularity recently in building real-world distributed replicated state machines such as Chubby.

As ever - comments or questions, feel free to [get in touch](mailto:henry.robinson@gmail.com).
