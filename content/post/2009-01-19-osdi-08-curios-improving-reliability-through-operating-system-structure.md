---
id: 139
title: "OSDI '08 - CuriOS: Improving Reliability Through Operating System Structure"
date: 2009-01-19T17:28:01+00:00
author: Henry
layout: post
guid: http://hnr.dnsalias.net/wordpress/?p=139
permalink: /osdi-08-curios-improving-reliability-through-operating-system-structure/
categories:
  - computer science
  - Operating systems
tags:
  - Operating systems
  - osdi
  - paper review
---
The second paper from OSDI that I'll mention here is one I'll only treat briefly - partly because it's a bit lightweight compared to some, and partly because I'm writing in a hurry. [CuriOS: Improving Reliability Through Operating System Structure](http://www.usenix.org/events/osdi08/tech/full_papers/david/david.pdf) attacks a problem with recovery from errors in microkernel operating systems.

<!--more-->

The problem looks like this. Imagine you have a service, say a file system server, which as a client you are talking to. Suddenly it crashes. How should the service recover? The service may have lost any state that it was maintaining, which impacts clients - for example, the file system server may no longer recognise the file handles the clients present for reading. Worse, if the crash was caused by a software bug, it is possible that the server has corrupted state belonging to other clients before it crashes, resulting in incorrect behaviour and perhaps crashing the clients as a knock-on effect.

The paper identifies several mechanisms for dealing with this problem, and then notes the flaws in each. Persisting the state across a restart may not work, since as noted above the state may be corrupted before the crash occurs. Restarting the server, as mentioned above, loses client state. Checkpointing the server and client, to roll back to a known-good previous state works, but is rather expensive and requires clients to deal with going back in time. Engineering clients to deal with server crashes at any time also works, but is expensive in terms of code complexity and pushes a lot of logic for failure detection into the client.

CuriOS is an object-oriented operating system which includes a design that mitigates the problems of server crashes. The idea behind it is extremely simple. Client state is no longer stored with the server, but with the clients themselves. However, it's mapped into a memory area that the clients cannot themselves access - the rationale being that this avoids security problems from clients being able to edit how they appear to a server. When a function is called against a server, the calling client's state is mapped into the server's address space - and only that client's state. This helps prevent errors from propagating across all clients since there is no physical way a server can corrupt two client's state at once.

That, in a nutshell, is the new contribution of the paper. It's very simple, but a very neat idea. As the paper notes, pushing state onto the client is not new (NFS does it, for one), but isolating the state - even from the client itself - is the key step.

The rest of the paper is taken up with content including a review of how popular microkernel designs (such as Minix3, L4, Chorus and EROS) fail when services fail. One noteworthy section is the inclusion of a set of observations regarding design principles for error recovery and fault isolation in microkernels. These are: first, make sure that addresses (such as file handles) persist across server restarts to ensure transparency. Secondly prevent clients from either timing out or issuing new calls during a period of recovery to prevent errors from propagating back to the client. Thirdly, client state should persists across server restarts and finally, client state should be isolated.

The evaluation involves measuring the memory and CPU overhead of the implementation in CuriOS. CuriOS supplies 'protected objects' which are services behind wrappers that take care of the mapping of client state and the restarts upon failure (signalled by C++ exceptions). CPU overhead is non-trivial - the time per call nearly doubles (but is still on the order of microseconds). The main cause of this cost is identified as the time taken to flush the TLB between switching between page tables on the switch into a protected object. Recovery time is not quantified beyond saying it's 'a few hundred microseconds'. The memory issue is similarly fudged. A lot of space is dedicated to explaining why state must at least be 1KB - it has to be rounded up in size to the nearest number of pages. However, the best figure is 'on the order of tens of kilobytes' when 'there are a small number of clients'. It's not clear what small means in this context, or if it's realistic. It's also not clear how much of this is a consequence of the CuriOS design, and how much is just to be expected with microkernels.

The ability to recover from errors is measured, slightly strangely. Two kinds of faults are injected into servers (one for every test). Bit flips simply mutate the contents of a register. These errors may go unnoticed, but if they are detected then CuriOS restarts the server. The second type of fault is a memory error, which simply returns a kind of 'bad read' code. These are always detected, and CuriOS is able to recover every time.

Although these are sound faults to inject, there's a weird reluctance in the paper to compare the behaviour of CuriOS against existing systems. The paper simply says 'all these faults would result in a service or system failure in most existing operating systems' - if true, it would have strengthened the case of CuriOS to show how many more faults it is immune to than traditional microkernels. The authors also measure the percentage of times the faults are recovered from - but do not precisely characterise what they mean by 'recovered', other than to say 'useable'. Sometimes clients are disconnected - which I thought was one of the conditions they were trying to avoid - but instead they count these as successes.

There are things to like about this paper, and things to be a bit suspicious of. The evaluation feels weirdly fudged, but the idea is very nicely motivated, and this seems like a decent solution. The overhead of the context switches are high; does this impact the savings gained from a microkernel design where context switches into the kernel are in general avoided?
