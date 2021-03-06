---
id: 347
title: "FLP and CAP aren't the same thing"
date: 2012-03-25T20:55:34+00:00
author: Henry
layout: post
guid: http://the-paper-trail.org/blog/?p=347
permalink: /flp-and-cap-arent-the-same-thing/
categories:
  - Distributed systems
---
An [interesting question](http://www.quora.com/Distributed-Systems/Are-the-FLP-impossibility-result-and-Brewers-CAP-theorem-basically-equivalent) came up on [Quora](http://www.quora.com) this last week. Roughly speaking, the question asked how, if at all, the [FLP](http://the-paper-trail.org/blog/?p=49) theorem and the [CAP theorem](http://the-paper-trail.org/blog/?p=290) were related. I'd thought idly about exactly the same question myself before. Both theorems concern the impossibility of solving fairly similar fundamental distributed systems problems in what appear to be fairly similar distributed systems settings. The CAP theorem gets all the airtime, but FLP to me is a more beautiful result. Wouldn't it be fascinating if both theorems turned out to be equivalent; that is effectively restatements of each other?

<!--more-->

> #### What the two theorems mean
>
> The **FLP theorem** states that in an asynchronous network where messages may be delayed but not lost, there is no consensus algorithm that is guaranteed to terminate in every execution for all starting conditions, if at least one node may fail-stop.</strong>
>
> The **CAP theorem** states that in an asynchronous network where messages may be lost, it is impossible to implement a sequentially consistent atomic read / write register that responds eventually to every request under every pattern of message loss.</ul>

Without ever having tackled the problem formally, I had speculated that they might be equivalent, based on a few observations. First, consensus and serialisable atomic objects are very closely related. Both involve causing a set of nodes to come to some agreement about shared state. In the case of consensus, it's the value proposed, in the case of atomic objects it's the order and identity of any operations performed. Secondly, the failure modes in both theorems appeared similar. CAP deals with 'partitions', which is the non-delivery of a subset of messages, and FLP deals with 'faulty nodes', which are nodes that only take a finite number of steps in any execution, steps which include message receipt. It seemed likely that both failure models could be used to describe the other. Thirdly, both problems deal with safety and liveness properties in distributed systems in the context of failures.

I wrote an answer up quickly to the question, and formulated a proof sketch typical for these kind of questions: if two problems are equivalent, then a solution to one is a solution to the other, and vice versa. My answer was, like so many informal arguments of this nature, concise, convenient, convincing and wrong.

### The Importance Of Context

[Emin Gun Sirer](http://www.cs.cornell.edu/people/egs/) pointed out was what wrong with my argument. One direction of the equivalence still looks good - a solution to CAP could be used to formulate a solution to the FLP problem. The problem is in the other direction - with my assertion that an FLP solution could be used to solve CAP. The distinction arises when we consider how each theorem treats nodes that aren't receiving the messages that are being sent to them. In FLP, such nodes are failed, and exempt from having to achieve consensus. In CAP, such nodes are only partitioned. Here's the difference: a CAP solution requires that any live node be able to correctly serve requests, _even if it has not received any messages_. So a partitioned node in FLP does _not_ have to achieve consensus, since it is considered failed, but the same node in CAP must - somehow - keep up with the activity of the rest of the system.

To reinforce this point, let's take a look at how [Gilbert and Lynch](http://dl.acm.org/citation.cfm?id=564601) proved their formalisation of the CAP theorem. They show a simple, intuitive result. If there is a permanent partition between two disjoint subsets of nodes in the system, call them  \\(G_1\\) and \\(G_2\\) , then a write to  \\(G_2\\) can never cause any different execution to occur in  \\(G_1\\) (because the only way that  \\(G_2\\) can influence  \\(G_1\\) is to send a message). But  \\(G_1\\) might have to respond to a read request **after** the initial write to \\(G_2\\) . To be sequentially consistent,  \\(G_1\\) must return the value of the write. It can't, because it can never learn of the write. CAP does not identify partition with failure. FLP does, at least in the single node case.

This blows my argument out of the water, because I was relying on equal treatment of nodes that were completely partitioned from all others in both settings. But that's not the case. So it is _not_ obvious that a solution to FLP can provide a solution to CAP.

### Proving Yourself Wrong

It would be unsatisfying only to establish that my original argument was flawed, because doing so doesn't actually settle the original question: can CAP and FLP be considered equivalent? Instead of trying to prove the positive result, armed with a bit more clarity about the difference between the two theorems, let's try and prove the negative; in particular that a solution to the FLP theorem will _not_ provide a solution to the CAP theorem.

A brief aside about the validity of this proof technique: you might reasonably be uncomfortable with me reasoning about 'positive' solutions to the CAP or FLP theorem, since respected researchers have already established in peer reviewed articles that no solution to either exists. However, it's still reasonable to effectively fantasise about what would be true _if_ positive solutions existed, and indeed it's fundamental to posing questions about equivalence. To appeal to a more commonly considered problem - computer scientists spend a lot of time considering what would happen if a positive solution to  \\(P ?= NP\\) was found, even though it's very possible (and usually considered likely) that no positive solution exists.

So let's pretend we have this magical black box, which will provide a solution to the FLP problem. That is, if every node in the system runs the algorithm in this box, then non-trivial consensus will always be reached in finite time at all non-faulty nodes, even where there is a single faulty node in the system, under all initial conditions and all network behaviours. Could we use this algorithm to solve CAP?

To prove otherwise, we're going to use a very similar argument to the one Gilbert and Lynch made, which is a proof by contradiction - we're going to assume that the algorithm can solve CAP, and then show an example of where it could not.

In the following, I'm going to use 'achieves consensus' as the target problem to solve. To do that, I need to quickly argue that any solution to consensus (ignoring the possibility of failures) can be used to implement an atomic sequentially consistent object. This is well established in the literature - the construction is called a _distributed finite state machine_. It works by having any node (i.e. one that receives a request) propose a value (i.e. the next write operation to execute, and its result). A round of consensus is performed, where the nodes either agree to the 'next operation' proposal, or vote it down. If they vote it down, the proposer tries again. There are solutions (such as [Paxos](http://the-paper-trail.org/blog/?p=173)) which ensure that every valid proposal is eventually accepted. An object implemented in this way is necessarily sequentially consistent, since all requests are _ordered_ by the process of consensus deciding the next operation to execute. Read requests may be served locally without performing consensus, since all nodes know about the 'most recent' write operation.

Let's set up our network as follows. There will be one distinguished node, \\(F\\) , which is the 'failing' node. It does not receive any messages from the rest of the system, and therefore takes a finite number of steps and then stops. We'll call the rest of the network \\(G = N - \{F\}\\) . In order to be a solution to CAP,  \\(F\\) must itself _decide_ on a value, since it may receive a read request to which it must respond in finite time. We'll show that even with a solution to FLP, there must be some execution where  \\(F\\) does not know the right answer.

(Note - this is an extremely simple argument to make informally, but I'm hoping to show how one might think about this and more difficult problems more formally).

Imagine that a <tt>write(\\(X\\) )</tt> operation is initiated by a client of some node in \\(G\\) , followed by read issued at \\(F\\) , all the time using the FLP algorithm we were given. Then, in order to be a correct CAP solution, F must respond to the read with the value \\(X\\) . To do that, it takes some finite series of steps \\(S=S_0\rightarrow S_1\rightarrow S_2\dots\rightarrow S_n\\) , and then takes no further action, because it is receiving no messages to spur it on.

Now imagine that instead, with the same exact initial conditions, a write(\\(Y\\) ) operation is initiated, again by a client of some node in \\(G\\) . Again, to be correct,  \\(F\\) must respond to a subequent read with the value \\(Y\\) . To do so, it takes another series of steps \\(T=T_0\rightarrow T_1\rightarrow T_2\dots\rightarrow T_n\\) . However,  \\(T\\) and  \\(S\\) _must be exactly the same sequence_. Why? Because the behaviour of a node is governed the algorithm it is executing, the initial conditions, and the messages it receives. The first two are exactly the same in both executions, and since no messages are being delivered, so is the third. Therefore, the value that  \\(F\\) has decided upon by the final step of both  \\(T\\) and  \\(S\\) must be the same value. But for correctness,  \\(F\\) is required to return _different_ values in each execution. This is a contradiction, hence the FLP-solving algorithm cannot be a solution to CAP, which was our initial assumption.

At this point you might be concerned - doesn't FLP guarantee that all nodes agree on the same value? That is, if we had a magical FLP-solver, wouldn't that guarantee that  \\(F\\) saw the same value as every other node in \\(G\\) ? The key observation here is that, from the perspective of FLP, _ \\(F\\) has failed_. From the perspective of CAP, it has not, and must continue to participate correctly in certain activities of the system. So the algorithm is correctly solving consensus as defined by FLP, but the requirements of CAP are too strong.

If consensus cannot be achieved even with an FLP solution in CAP, we cannot use it to construct any kind of solution to CAP.

### What We Have Learnt

This result (if I've made no mistakes) is arguably more interesting than if the original equivalence assertion was true. To be an interesting impossibility result in distributed systems, it's usually true that you want to place the fewest restrictions possible on the environment in which you're establishing that result. The idea is that you give your result _every chance_ to be wrong, by allowing the environment few restrictions in the tricks it can pull to overcome the obstacles you're constructing. Then, if your result _still_ turns out to be true, you know you have proved something really quite strong. (There way 'strong' and 'weak' are often used can be confusing - _weak_ assumptions lead to _strong_ impossibility results and vice versa).

So FLP, with its strictly weaker restrictions - all messages are eventually delivered, faulty nodes don't have to achieve consensus - is by this definition a stronger result than CAP, which allows messages to be lost forever and forces partitioned nodes to participate in the system. It's therefore much more surprising - and the authors in the [original paper](http://dl.acm.org/citation.cfm?id=214121) remark on this - because it's so much more unexpected.

Conversely, CAP appears much more humdrum. Is it really surprising to anyone that it's impossible to maintain sequentially consistent state in a distributed system if nodes cannot talk to each other? It's not too far away from discussing what's possible in a network that can experience total node failure - nothing at all, which is why all papers disregard it as a consideration. The utility of CAP comes from telling systems designers that they must be prepared to work in a world where availability or consistency are compromised, but is rather cataclysmic about the situations in which they might be threatened.

It would be more interesting to relax one or two assumptions - what if partitions are only temporary? What if we allow the client to participate (i.e. a client that has seen a write is allowed to convey that fact to a replica that it issues a read from)? What if partitioned nodes were excluded from participation? If we can show CAP to hold in these circumstances, it greatly restricts our ability to design correct, available systems that experience much weaker failures. _That_ would be interesting.
