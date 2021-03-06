---
id: 44
title: Reservoir Sampling
date: 2008-04-09T16:55:09+00:00
author: Henry
layout: post
guid: http://hnr.dnsalias.net/wordpress/?p=43
permalink: /reservoir-sampling/
aliases:
 - /blog/reservoir-sampling/
categories:
  - Probability
---
Right, time to get this blog back on track. I want to talk about a useful technique that's both highly practical and crops up in interview scenarios regularly.

Consider this problem: How can we efficiently randomly select  \\(k\\) items from a set  \\(S\\) of size \\(n > k\\) , where  \\(n\\) is unknown? Each member of  \\(S\\) should have an equal probability of being selected.

At first glance, this problem looks a strange mix of trivial and impossible. If we don't know \\(n\\) , how can we know how to weight our selection probabilities? And, assuming  \\(S\\) is finite, how can we not know  \\(n\\) if we're able - as we must be - to visit every element in \\(S\\) ?

To address the second point first: we can find  \\(n\\) by iterating over all elements in \\(S\\) . However, this adds a pass over the entire set that might be expensive. Consider selecting rows at random from a large database table. We don't want to bring the whole table into memory just to count the number of rows. Linked lists are another good motivating example - say we would like to select  \\(k\\) elements from a linked list of length \\(n\\) . Even if we do loop over the list to count its elements, employing the simple approach of choosing  \\(k\\) elements at random will still be slow because random access in linked lists is not a constant time operation. In general we will take on average  \\(O(kn)\\) time. By using the following technique, called 'reservoir sampling', we can bring this down to \\(\Theta(n)\\) .

We can keep our selection of  \\(k\\) elements updated on-line as we scan through our data structure, and we can do it very simply.

<!--more-->

Assume we make one pass over \\(S\\) . As we encounter a new element, we need to make a decision about whether to include it in the sample or not. Consider the  \\(i^{th}\\) element. We assume that we have already selected  \\(k\\) elements uniformly at random from the  \\(i-1\\) elements we have already encountered. We'll call that set of  \\(k\\) the _reservoir_. If we can update the reservoir so that it correctly contains  \\(k\\) elements, all selected with a probability of \\(k/i\\) , then we can show that when  \\(i=n\\) we'll have a correct sample.

With what probability should we put the  \\(i^{th}\\) element in the reservoir? This is simple: we select it with probability \\(k/i\\) . But this 'overflows' the reservoir, so we have to choose an element to remove to make space. All  \\(i-1\\) elements are in the reservoir with probability  \\(k/(i-1)\\) by our assumption, so intuition tells us that we should just pick one of the  \\(k\\) at random and evict that.

Does that work? Let's look at the probabilities. The probability that the  \\(i^{th}\\) element is in the reservoir is  \\(k/i\\) which is correct. The probability that any of the  \\(k\\) elements currently in the reservoir remain there is: \\(1-\frac{k}{i}+\frac{k}{i}\frac{(k-1)}{k}=\frac{i-1}{i}\\) . For all  \\(i-1\\) elements already considered there is, by assumption, a  \\(\frac{k}{i-1}\\) probability that a given element is in the reservoir already. Multiplying the two probabilities to give the total probability that any of the  \\(i-1\\) elements will be in the reservoir after considering the  \\(i^{th}\\) element gives:

\\(P( X_{ji} = 1 ) = \frac{k}{i-1}\frac{i-1}{i} = \frac{k}{i}\\)

where  \\(X_{ji}, 1\leq j < i\\) is a binary random variable which is 1 if element  \\(j\\) is selected to be in the reservoir after considering the first  \\(i\\) elements.

While  \\(i \le k\\) the probability of selecting element  \\(i\\) is 1 and since the reservoir will not yet be full there's no point removing elements. This gives us the 'base case' of our inductive proof: it's easy to show that this method gives us a full reservoir with element selection probability  \\(k/i\\) when \\(i=k\\) . For all \\(i >k\\) , our inductive step above allows us to update the reservoir, keeping all the selection probabilities equal to \\(k/i\\) .

This is incredibly simple to implement, and very fast - because the reservoir is fixed size we can implement it as an array and not worry about high costs for insertion or deletion. Two random numbers are needed per element, making this a very efficient solution and much quicker than random access in a linearly accessible data structure.
