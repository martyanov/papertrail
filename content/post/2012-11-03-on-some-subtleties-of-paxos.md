---
id: 427
title: On some subtleties of Paxos
date: 2012-11-03T19:02:22+00:00
author: Henry
layout: post
guid: http://the-paper-trail.org/blog/?p=427
permalink: /on-some-subtleties-of-paxos/
aliases:
  - blog/on-some-subtleties-of-paxos
categories:
- Distributed systems
- paxos
tags:
- paxos
---
There's one particular aspect of the Paxos protocol that gives readers of this blog - and for some time, me! - some difficulty. This short post tries to clear up some confusion on a part of the protocol that is poorly explained in pretty much every major description.

<!--more-->



This is the observation that causes problems: two different nodes can validly accept different proposals for the same Paxos instance. That means that we could have a situation where node  \\(A\\) has accepted a proposal  \\(P=(S, V)\\), and node  \\(E\\) has accepted a proposal  \\(P'=(S', V')\\). We can get there very easily by having two competing proposers that interleave their prepare phases, and crash after sending an accept to one node each. On the face of it, this is a very concerning situation - how can two nodes both believe a different value has been accepted? Doesn't this violate one of the consensus guarantees of uniformity?

The answer lies in the fact that the nodes doing the accepting are not (necessarily) the nodes that 'learn' about the agreed value. Paxos calls for a distinguished category of nodes, called 'learners', which hear about acceptance events from the nodes doing the accepting. We call the latter nodes 'acceptors', and say that learners 'commit' to a value once they can be sure it's never going to change for a given Paxos instance.

But when does a learner know that a value has really been accepted? It can't just go on the first acceptance that it receives (since as we have shown, two different acceptors can have accepted different values and may race to send their values to the learners). Instead, a learner must wait for a majority of acceptors to return the same proposal. Once  \\(N/2 + 1\\) acceptors are in agreement, the learner can commit the value to its log, or whatever is required once consensus is reached. The rest of this post shows why this is both necessary and sufficient.

If no majority of acceptors have accepted the same value, it's trivial to see why a learner cannot commit to a value sent by any acceptor, for the same race-based argument made earlier. A more interesting case is the following: suppose that a majority of acceptors have accepted a proposal with the same value, but with different sequence numbers (i.e. proposed by a different proposer). Can a learner commit to that value once it has learnt about all the acceptances? In the following section we show that it can.

### Conditions for learner commit

**Theorem:** _Let a majority of acceptors have accepted some proposal with value  \\(V\\), and let  \\(P = (S,V)\\) be the proposal with the largest sequence number  \\(S\\) amongst all those acceptors. Then there is no proposal  \\(P' = (S', V')\\) that is accepted at any node with  \\(S'>S\\) and  \\(V' != V\\)._

**Proof:** We proceed in two steps. The first shows that when \\(P\\) is accepted, there is no proposal already accepted at any node with a later sequence number. Assume that this is false, and some node has accepted \\(P'\\). Then a majority of acceptors must have promised to accept only proposals with sequence number \\(S'' > S'\\). Since \\(S < S'\\),  \\(P\\) cannot be accepted by a majority of acceptors, contradicting the assumption.

Second, we prove by a very similar argument that once  \\(P\\) has been accepted, no node will accept a proposal like  \\(P'\\). Again, assume this is false. Then a majority of acceptors must have sent a \\(promise\\) of  \\((S', V')\\) in order for the proposer of  \\(P'\\) to be sending \\(accept\\) messages. If so, then either that same majority should have ignored the \\(accept\\) message for  \\(P\\) (since  \\(S < S'\\)), or  \\(V' = V\\) if the \\(accept\\) of \\(P\\) happened before the proposal of \\(P\\) (by the first half of this proof we know that there is no proposal with a later sequence number than  \\(S\\) already accepted, so the proposer is guaranteed to choose  \\(V\\) as the already accepted value with the largest sequence number). In either case there is a contradiction;  \\(P\\) has not been accepted or  \\(V = V'\\).

What this theorem shows is that once a value has been accepted by a majority of acceptors, no proposal can change it. The sequence number might change (consider what happens if a new proposer comes along and runs another proposal over the same instance - the sequence number will increase at some acceptors, but the proposer must choose the majority value for its accept message). But since the majority-accepted value will never change, the learners can commit a value when they hear it from a majority of acceptors.

### Fault tolerance

Now it's instructive to think about what this means for Paxos' fault-tolerance guarantees. Imagine that a proposal was accepted at a minimal (i.e.  \\(N/2+1\\) nodes) majority of acceptors before the proposer crashed. In order for the value to be committed by a learner, every one of those acceptors must successfully send its accepted value on to the learners. So if a single node in that majority fails, that Paxos instance will not terminate for all learners. That appears to be not as fault-tolerant as we were promised.

There are several ways to interpret this fact. The first is that Paxos only guarantees that it will continue to be correct, and live, with up to  \\(N/2 + 1\\) failures; and _failing to reach agreement for a single proposal does not contravene these properties_. It's also true that if the proposer dies before sending any accept messages, that proposal will also never complete. However, another proposer can always come along and finish that instance of the protocol; it's this that is no longer true if a majority of acceptors fail.

The second interpretation is that it makes sense for acceptors to also act as learners, so that they can update their values for a given Paxos instance once they realise that consensus is complete. It's often true that learners and acceptors are the same thing in a real Paxos deployment, and the aim is usually to have as many acceptors up-to-date as possible.

So that's a short look at how the distribution of accepted proposals can evolve during Paxos, and how the protocol guarantees that eventually the cluster will converge on a value that will never change.
