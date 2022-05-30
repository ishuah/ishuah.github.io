---
layout: post
title: "A Memoir on Reference Counting"
description: A deep dive into memory leaks from a programmer's perspective
date: 2022-05-30 18:11:37
image: '/images/garbage-collection.jpg'
tags: [memory-management, garbage-collection]
---

_Photo by <a href="https://unsplash.com/@gary_at_unsplash?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Gary Chan</a> on <a href="https://unsplash.com/s/photos/garbage-collection?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>_


In 1960, John McCarthy published a paper titled [Recursive Functions of Symbolic Expressions and Their Computation by Machine, Part I](http://www-formal.stanford.edu/jmc/recursive.pdf). In it, he meticulously describes the original implementation of Lisp, the second oldest programming language still in use. As part of the language, McCarthy describes an algorithm that automatically reclaims memory. He calls it a "reclamation cycle", but in a footnote added in 1995, he writes:

_We already called this process "garbage collection", but I guess I chickened out of using it in the paper—or else the Research Laboratory of Electronics grammar ladies wouldn't let me._

McCarthy coined the term "garbage collection" and introduced the first garbage collection process. His implementation was a tracing algorithm that identifies reachable objects by depth-first search and frees up unreachable objects. Although a remarkable proof of concept, this approach had limitations. McCarthy points out in the paper:

_Its efficiency depends upon not coming close to exhausting the available memory with accessible lists. This is because the reclamation process requires several seconds to execute, and therefore must result in the addition of at least several thousand registers to the free-storage list if the program is not to spend most of its time in reclamation._

This statement implies that garbage collection calls become costly when the program runs close to memory capacity. 

In December 1960, another computer scientist, George E. Collins, published a paper titled [A Method for Overlapping and Erasure of Lists](https://dl.acm.org/doi/pdf/10.1145/367487.367501). In this paper, Collins addresses the drawbacks in McCarthy's approach and introduces reference counting, a more efficient process of reclaiming memory. Speaking of McCarthy's approach, he writes:

_McCarthy's solution is very elegant, but unfortunately it contains two sources of inefficiency. First and most important, the time required to carry out this reclamation process is nearly independent of the number of registers reclaimed. Its efficiency thereby diminishes drastically as the memory approaches full utilization. Second, the method as used by McCarthy required that a bit (McCarthy uses the sign bit) be reserved in each word for tagging accessible registers during the survey of accessibility. If, as in our own current application of list processing, much of the atomic data consists of signed integers or floating-point numbers, this results in awkwardness and further loss of efficiency._

Collins's solution worked by maintaining a count of references (pointers) to each object in memory. When a new pointer is created the counter is incremented. Inversely, the counter is decremented when a pointer is destroyed. When an object's reference count gets to zero, its memory is automatically reclaimed. To optimize this solution, Collins allocates space for a reference count only when an object has more than two references.

Collin's approach, unlike McCarthy's, is interleaved with program execution, thus better suited for programs where response time is critical. That said, this approach has a subtle drawback in that if objects have a pointer cycle, their reference count can never reach zero and therefore cannot be reclaimed. At the time, the LISP language did not allow cyclic data structures, so this issue did not affect memory reclamation.

Some languages still use reference counting without any special handling of pointer cycles. Perl 5, for instance, supports weak references, allowing programmers to deliberately avoid creating reference cycles. If data structures contain reference cycles, Perl runtime will only reclaim them once the parent thread dies. This is considered a welcome tradeoff, as opposed to implementing overhead cycle detection that would slow down execution time.

Python implements reference counting too. To address reference cycles, Python implements an algorithm called [generational cyclic garbage collection](https://rushter.com/blog/python-garbage-collector/#:~:text=no%20longer%20needed.-,Generational%20garbage%20collector,-Why%20do%20we). Reference counting runs in real-time, while cyclic garbage collection runs periodically. One of the reasons why Python maintains the GIL is to protect the reference count from race conditions. 

In June 2001, David Bacon and V.T Rajan published [Concurrent Cycle Collection in Reference Counted Systems](https://pages.cs.wisc.edu/~cymen/misc/interests/Bacon01Concurrent.pdf). In this paper, they address reference cycles collection in reference counted systems by building on McCarthy's prior work. As Bacon and Rajan observed:

_...when a reference count is decremented and does not reach zero, it is considered as a candidate root of a garbage cycle, and a local search is performed. This is a depth-first search which subtracts out counts due to internal pointers._

Bacon and Rajan present two algorithms, a synchronous algorithm, and a concurrent algorithm. The concurrent algorithm is based on the synchronous algorithm, with additional safety checks to guard against race conditions. PHP is a reference-counted language that applies reference cycle collection. As of version 5.3, PHP implements the synchronous algorithm to handle circular reference memory leaks.

Objective-C implements [Automatic Reference Counting](https://en.wikipedia.org/wiki/Automatic_Reference_Counting#cite_note-clang-4), which differs from garbage collection in that there's no background process that deallocates unused memory. Instead, ARC works by inserting reference counting code at compile-time, saving the developer the hard work of manually managing memory. Objective C has a rigid set of [memory management rules](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html). The team behind Objective C's compiler realized that these rules were dependable enough to be automated as part of the compiler. ARC does not handle reference cycles, but as with Perl, this is considered a minor setback.

Reference counting is still a relevant topic in Computer Science. In 2019, Leonardo de Moura and Sebastian Ullrich published [Counting Immutable Beans: Reference Counting Optimized for Purely Functional Programming](https://arxiv.org/pdf/1908.05647.pdf). In this paper, they describe in detail a reference counting memory reclamation strategy for [Lean 4](https://github.com/leanprover/lean4), an open source functional programming language. The performance numbers in the paper are impressive, Lean 4 beats most of the compilers in all the tests. However, Moura and Ulrich state in the paper:

_We remark that in Lean and λpure, it is not possible to create cyclic data structures. Thus, one of the main criticisms against reference counting does not apply._

Unlike  Lean and λpure, most functional languages have no restrictions on cyclic data structures, meaning the algorithm proposed won't work for them. Maybe someone will come up with a cyclic reference collection algorithm for functional languages in the near future.

### References:

* [Recursive Functions of Symbolic Expressions and Their Computation by Machine, Part I](http://www-formal.stanford.edu/jmc/recursive.pdf)
* [A Method for Overlapping and Erasure of Lists](https://dl.acm.org/doi/pdf/10.1145/367487.367501)
* [Concurrent Cycle Collection in Reference Counted Systems](https://pages.cs.wisc.edu/~cymen/misc/interests/Bacon01Concurrent.pdf)
* [Counting Immutable Beans: Reference Counting Optimized for Purely Functional Programming](https://arxiv.org/pdf/1908.05647.pdf)
* [What Is the Python Global Interpreter Lock (GIL)?](https://realpython.com/python-gil/)
* [Garbage collection in Python: things you need to know](https://rushter.com/blog/python-garbage-collector/)
* [Advanced Memory Management in Objective-C](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html)