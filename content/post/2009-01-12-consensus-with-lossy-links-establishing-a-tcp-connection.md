---
id: 110
title: 'Consensus with lossy links: Establishing a TCP connection'
date: 2009-01-12T13:51:27+00:00
author: Henry
layout: post
guid: http://hnr.dnsalias.net/wordpress/?p=110
permalink: /consensus-with-lossy-links-establishing-a-tcp-connection/
categories:
  - computer science
  - Distributed systems
tags:
  - consensus
  - Distributed systems
  - tcp
---
After a hiatus for the Christmas break, during which I travelled to the States, had a job interview, went to Vegas, became an uncle and got a cold, I'm back on a more regular posting schedule now. And I've got lots to post about.

Before I talk about other theoretical consensus protocols such as Paxos, I want to illustrate a consensus protocol running in the wild, and show how different modelling assumptions can lead to protocols that are rather different to the *PC variants we've looked at in the [last](http://hnr.dnsalias.net/wordpress/?p=90) [couple](http://hnr.dnsalias.net/wordpress/?p=103) of posts. We've been considering situations like database commit, where many participants agree en-masse to the result of a transaction. We've assumed that all participants may communicate reliably, without fear of packet loss (or if the packets are lost then the situation is the same as if the host that had sent the packet had failed).

The Transmission Control Protocol (TCP) gives us at least some approximation to a reliable link due to the use of sequence numbers and acknowledgements. However before we can use TCP both hosts involved in a point to point communication have to establish a connection: that is, they must both agree that a connection is established. This is a two-party consensus problem. Neither party can rely on reliable transmission, and can instead only use the IP stack and below to negotiate a connection. IP does not give reliable transmission semantics to packets and works only on a best-effort principle. If the network is noisy or prone to outages then packets will be lost. How can we achieve consensus in this scenario?

Those who have been reading this blog as far back as my explanation of [FLP impossibility](http://hnr.dnsalias.net/wordpress/?p=49) will probably be thinking that this is a trick question. FLP impossibility shows that if there is an unbounded delay in the transmission of a packet (i.e. an asynchronous network model) then consensus is, in general, unsolvable. Lossy links can be regarded as delaying packet delivery infinitely - therefore it seems very likely that consensus is unsolvable with packet loss.

In fact, this is completely true. Consensus with arbitrary packet loss is an unsolvable problem, even in an otherwise synchronous network. In this post I want to demonstrate the short and intuitive proof that this is the case, then show how this impossibility is avoided where possible in TCP connection establishment.

<!--more-->

## Theoretical Lossy Links

(A much more formal presentation of this proof can be found in Lynch's [Distributed Algorithms](http://www.amazon.co.uk/Distributed-Algorithms-Kaufmann-Management-Systems/dp/1558603484/ref=sr_1_1?ie=UTF8&s=books&qid=1231768254&sr=8-1), chapter 5.)

<div id="attachment_127" style="width: 297px" class="wp-caption alignnone">
  <a href="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-1.png"><img src="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-1.png" alt="Figure 1: Two party consensus protocol - Execution A" title="Figure 1: Two party consensus protocol -  Execution A" width="287" height="300" class="size-medium wp-image-127" /></a>
  
  <p class="wp-caption-text">
    Figure 1: Two party consensus protocol - Execution A
  </p>
</div>

Figure 1 shows an instance of an abstract consensus protocol between two parties. At the end of the protocol, both parties have agreed upon a value. The number, and pattern, of messages is irrelevant here. For simplicity in our protocol each party sends a reply to every message it receives, but that's not a requirement of this proof.

Recall from previous discussions of consensus that any legitimate protocol must be _valid_ - that is, the value agreed upon must be proposed by one of the participants. So let's imagine that this was the execution when both parties started with 1 as their proposed value (for binary consensus), so the only valid result is for both parties to decide 1.

<div id="attachment_128" style="width: 297px" class="wp-caption alignnone">
  <a href="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-2.png"><img src="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-2.png" alt="Figure 2: Two-party consensus with single message loss - Execution B" title="Figure 2: Two-party consensus with single message loss - Execution B" width="287" height="300" class="size-medium wp-image-128" /></a>
  
  <p class="wp-caption-text">
    Figure 2: Two-party consensus with single message loss - Execution B
  </p>
</div>

Now, imagine what happens when the last message in the trace is lost. Figure 2 shows the situation - we'll call this trace Execution B. Process 2 sees exactly the same message trace as Execution A, and so must decide 1 as before - our protocol executes deterministically at each process, and therefore depends only on the initial state and the list of messages received. Therefore, by agreement, process 1 must also decide 1 even though it didn't receive the last message in the protocol.

<div id="attachment_129" style="width: 297px" class="wp-caption alignnone">
  <a href="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-3.png"><img src="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-3.png" alt="Figure 3: Two-party consensus with double message loss - Execution C" title="Figure 3: Two-party consensus with double message loss - Execution C" width="287" height="300" class="size-medium wp-image-129" /></a>
  
  <p class="wp-caption-text">
    Figure 3: Two-party consensus with double message loss - Execution C
  </p>
</div>

We can then make the same argument about the second-to-last message. Figure 3 shows Execution C, in which process 1 sees exactly the same trace as Execution B, and therefore must decide 1 as before. So, again, by agreement, process 2 must decide 1 again.

<div id="attachment_130" style="width: 297px" class="wp-caption alignnone">
  <a href="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-4.png"><img src="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-4.png" alt="Figure 4: Two-party consensus with only one delivered message - Execution D" title="Figure 4: Two-party consensus with only one delivered message - Execution D" width="287" height="300" class="size-medium wp-image-130" /></a>
  
  <p class="wp-caption-text">
    Figure 4: Two-party consensus with only one delivered message - Execution D
  </p>
</div>


  
By repeated application of this argument we can unravel the protocol to that seen in Execution D in figure 4. In this execution, process 1 is sending a single message to process 2. Our inductive argument tells us that both process 1 and process 2 must decide 1, even though process 1 doesn't receive any messages! Consider what happens if instead of proposing 1 initially, process 2 proposes 0. It must still decide 1, because process 1 will decide 1 and we require both processes to decide the same value.

<div id="attachment_122" style="width: 310px" class="wp-caption alignnone">
  <a href="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-5.png"><img class="size-medium wp-image-122" title="Figure 5" src="http://the-paper-trail.org/wp-content/uploads/2010/01/figure-5.png" alt="Figure 5: Two-party consensus with no messages delivered - Execution E" width="300" height="216" /></a>
  
  <p class="wp-caption-text">
    Figure 5: Two-party consensus with no messages delivered - Execution E
  </p>
</div>

Now, let's finally take away this last remaining message, with process 1 proposing 1 and process 2 proposing 0. Process 1 sees the same trace as before and must, therefore, decide 1. So, yet again by validity, we require that process 2 decide 1 _even though it has not received any messages, and even though its initial value was 0!_

To finally elicit a contradiction, we have to consider what happens if process 1 proposes 0 instead of 1 in execution E. Process 2 sees exactly the same trace as before, and therefore must decide 1. So by agreement process 1 must decide 1. But this breaks validity - neither process has proposed 1!

What we have done is shown that, in the case of arbitrary message loss, processes must default their decision to some value in order to be correct in every execution. But this amounts to guessing, and the chances of getting that right even for the two party case are no better than 1/2. Therefore there are some executions where processes will decide incorrectly, and this means our consensus protocol is broken.

## Lossy Links In Practice

Well, there's a depressing result. As long as there are lossy links, consensus will be impossible. And links are _always_ going to be lossy, especially practical ones like the air our wireless transmissions work through.

Of course, things aren't actually anywhere near that bad - otherwise networks would never work at all. We need to keep in mind that all our proof has shown is that in certain cases our consensus protocol decides incorrectly. How bad a problem is that in practice? Well, it depends on the decision being made. Choosing to launch a nuclear missile by accident is a bit of a problem. However, not establishing a TCP connection when both parties were happy to make one is not the end of the world. Therefore, all we have to do is ensure that when our protocol fails, it fails in a way that is non-destructive.

In the case of TCP, this is easy. We'll have both parties default to 0 - that is, not establish a connection - unless all messages are delivered correctly. So it is possible that, even though both parties may wish to establish a connection, a connection may not be created if messages are lost. This means that progress may occasionally not be made. The easy way around that is to have one of the parties initiate another instance of the protocol when a timeout is detected without the first instance of the protocol having completed. All that is required is for one of these instances to successfully complete; this approach costs us a bit of a set-up delay but buys an awful lot of robustness. It is of course possible that none of these repeat instances may complete successfully if the link is broken; in this case the protocol breaks the termination property we require of consensus if the retries occur indefinitely - otherwise eventually the initiator gives everything up as lost and the protocol terminates by defaulting to 0.

[In fact, re-initiating the protocol can cause some problems if the messages are not lost, but merely delayed by the network. TCP has some structure to help prevent duplicate connections become established.]

TCP connections can therefore use the simplest protocol imaginable. The initiating process sends a connection request to a remote host, which then acknowledges the request. The loss of either message is detected by the initiator via a timeout, upon which it resends a connection request.

There is in fact a lot more to TCP connection establishment, such as agreement upon a sequence number and the already mentioned avoidance of duplicate connections - but the very simple skeleton of the protocol remains the same.

The delivery of a packet via TCP may also be considered an instance of a consensus protocol - both parties must agree that the packet has been delivered before moving the delivery window forward to allow for sending of later packets. As we have seen, TCP's claim to 'reliable' transmission must therefore also be something of a white lie; but again we simply default to 0 or 'undelivered'. Here again, repeated retries in the face of a 100% lossy link cause either the delivery protocol to never terminate, or to eventually default to undelivered which breaks the supposed reliability. In the general case 100% broken end-to-end links are rare in the Internet without the host having failed which is why TCP works almost as advertised. TCP contains a number of mechanisms to ensure that in partly lossy links reliable delivery is achieved (essentially appealing to the fact that with enough retries every packet will get through).

So, despite another impossibility proof, the real world thankfully copes fairly well with lossy links. As with asynchrony, the reason it does is that the failure conditions are sufficiently rare that they don't occur often enough to stymie repeated use of a protocol, and when they do occur the consequences are not disasterous.
