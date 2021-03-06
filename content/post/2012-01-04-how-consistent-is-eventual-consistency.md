---
id: 334
title: How consistent is eventual consistency?
date: 2012-01-04T15:22:05+00:00
author: Henry
layout: post
guid: http://the-paper-trail.org/blog/?p=334
permalink: /how-consistent-is-eventual-consistency/
categories:
  - Uncategorized
---
[This page](http://www.eecs.berkeley.edu/~pbailis/projects/pbs/ "PBS"), from the 'PBS' team at Berkeley's [AMPLab](http://amplab.cs.berkeley.edu/) is quite interesting. It allows you to tweak the parameters of a [Dynamo](http://the-paper-trail.org/blog/?p=51 "Dynamo")-style system, then by running a series of Monte Carlo simulations gives an estimate of the likelihood of staleness of reads after writes. 

Since the Dynamo paper appeared and really popularised eventual consistency, the debate has focused on a fairly binary treatment of its merits. Either you can't afford to be wrong, ever, or it's ok to have your reads be stale for a potentially unbounded amount of time. In fact, the suitability of eventual consistency is dependent partly on the _distribution_ of stale reads; that is the speed of quiescence of a system immediately after a write. If the probability of a ever seeing a stale read due to consistency delays can be reduced to smaller than the probability of every machine in the network simultaneously catching fire, we can probably make use of eventual consistency.

Looking at many designed systems (where there is little more than conventional wisdom on how to choose R and W), it's clear that an analytical model relating system parameters to distributions of behaviour is sorely needed. PBS is a good step in that direction. It would be good to see the work extended to handle a treatment of failure distributions (although a good failure model is hard to find!). The reply latencies of write and read replicas are modelled exponentially distributed CDFs, but in reality there's a more significant probability of the reply latency becoming infinite. Once that distribution is correctly modelled, PBS should be able to run simulations against it with no change. 

A great use for this tool would be to enter some operational parameters, such as the required consistency probability, max number of nodes, availability requirements and maximum request latency, and have PBS suggest some points in the system design space that would meet these requirements with high probability. As the size of the R / W quora get larger, the variance on the request latencies gets larger, but the resilience to failures increases as does the likelihood of fresh reads. For full credit, PBS could additionally model a write / read protocol (i.e. 2-phase commit) which has different consistency properties. As [Daniel Abadi](http://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html "Abadi's PACELC") discusses, when things are running well different consistency guarantees trade off between latency and the strength of consistency. 

Nice work PBS team!
