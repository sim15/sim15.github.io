---
layout: noheader
title: "15-418 project proposal"
permalink: /418-project/proposal/
---

<center><u>Project Proposal: Parallel Bulk Operations for Ordered Sets and Maps in OCaml</u></center>


<div style="text-align: justify;" markdown="1">
***<center><u>Summary</u></center>***


Ordered sets and maps are among the most widely used data structures, and in functional languages are implemented as purely functional balanced binary search trees. 
Their bulk operations (`union`, `intersection`, `difference`, `filter`) have significant inherent parallelism (independent recursive subproblems that touch disjoint data), yet standard library implementations in languages like OCaml are fully sequential. 
We're interested in parallelizing these operations using OCaml 5's shared-memory multicore support (`domains`) as our platform. 
OCaml is an interesting environment because its runtime characteristics (a coordinating garbage collector, heavyweight task creation relative to C++ work-stealing runtimes, and heavy allocation from persistent/immutable data structures) create bottlenecks that don't exist in C++ implementations.
We'll measure speedup and scaling behavior, figure out what specific runtime and architectural factors limit parallel performance, and compare against both the sequential OCaml standard library and the C++ PAM library to isolate how much of the performance gap is algorithmic versus being imposed by the runtime.

----

***<u><center>Background</center></u>***



***Ordered Sets and Maps.*** An ordered set is a collection of totally ordered typed keys (e.g., integers) and supports membership queries, insertion, deletion, and iteration in sorted order. 
An ordered map extends this by associating each key with a value (key-value pairs). 
There are several standard ways to implement them. 
Hash tables offer $$O(1)$$ expected-time point operations but don't maintain key ordering and don't efficiently support range queries, bulk merging, or set operations like union and intersection. 
Implementations with sorted arrays support binary search and good cache behavior but make inserts and deletes expensive (worst case $$O(n)$$). 
Implementations via balanced binary search trees (BSTs) like AVL trees, red-black trees, weight-balanced trees, and treaps, allow $$O(\log n)$$ work for insert and deletes, efficient ordered iteration, and support bulk operations like `union`, `intersection`, and `difference`. 
Balanced BSTs are the standard choice for workloads that need both point operations and bulk or range operations over ordered data.

***Balanced BST Implementations.*** In imperative languages like C/C++, balanced BST implementations usually mutate the tree in place. 
For example, an insertion modifies existing node pointers and rebalances by rotating nodes. 
In functional languages like OCaml, Haskell, Scala, and Erlang, the usual approach is purely functional (or "persistent") in that operations never modify existing nodes. 
Instead, they create new nodes and share the old tree's structure via path copying. 
The old version of the tree stays valid. 
This persistence makes thinking about correctness easier since there is no aliasing or mutation and eliminates data races by construction, since no thread ever writes to a location another thread might read. 
But the main cost is allocation, since every modification creates $$O(\log n)$$ new nodes and the garbage collector (something C++ doesn't have to worry about) must eventually take back the old unreachable ones.

***Parallelizing balanced BSTs.***
To add parallelism to balanced BSTs, there are two different approaches.

- *<u>Concurrent access</u>* means multiple threads perform independent point operations (like inserts, deletes, lookups) on a single shared mutable tree at the same time. The challenge is synchronization, since two threads might try to rotate the same node, or one might read a pointer that another is modifying. Using fine-grained locking, lock-free techniques using compare-and-swap, or transactional memory can allow for this access to be synchronized. Concurrent BSTs are well-studied in imperative languages with standard mutable shared state, and they are useful when many clients need to read and write the same data structure at high throughput. But the parallelism comes from many operations happening at once and not from speeding up any single operation.


- *<u>Bulk parallelism</u>* means taking a single operation that (hopefully) touches a large portion of the tree, like a `union` of million-element sets, `filter` over an massive tree, `intersection` of two large trees, and parallelizing its internal work. Many bulk operations on BSTs have a lot of opportunity for recursive decomposition into independent subproblems, like, for example, in a `union(T1, T2)`, one can choose a key to split both trees into "less than" and "greater than" halves. 
Then you can recursively union each pair of halves. These two recursive calls access disjoint data and can be run in parallel. 


Bulk parallelism speeds up individual large operations while concurrent access supports many simultaneous small operations. 
For this project, we focus on bulk parallelism. 
The purely functional setting makes this particularly nice to think about since the data structure is never mutated and the forked subproblems are independent with no synchronization needed. 

***The Functional (OCaml) Setting.*** OCaml's standard library provides `Set` and `Map` implemented as functional AVL trees, supporting `union`, `inter` (intersection), `diff` (difference), `split`, `filter`, `map`, and `fold`. 
The implementation is entirely sequential. 
OCaml 5, released in 2022, introduced shared-memory multicore parallelism for the first time in the language's history via `domains`, which are OS-level threads that can run OCaml code in parallel. 
Prior to OCaml 5, a global runtime lock prevented any parallel execution of OCaml code. 
This makes OCaml 5 a new and relatively unexplored environment for parallel data structure work.
The runtime model differs from C++, which impacts performance. For example:

- OCaml's garbage collector coordinates across domains, and each domain has its own minor heap, while the major heap is shared. The major heap requires a stop-the-world synchronization point. Since purely functional tree operations allocate a lot (every node creation produces a new heap object), GC coordination is a potentially major bottleneck that C++ doesn't suffer from. 
- Task creation is also heavier since OCaml domains are closer to OS threads than to something like Cilk's spawned tasks (which are more lightweight). This means the granularity at which forking is beneficial for performance may be much larger.
- OCaml represents all heap objects with a uniform boxed layout (every value is accessed through a pointer to a tagged, header-prefixed block on the heap, including individual tree nodes). This means tree traversal involves more indirection and more memory overhead per node than in C++ implementations, which can inline small values directly into structs, pack nodes more compactly, and use cache-friendly custom allocators to control where nodes land in memory.

These differences make OCaml an interesting platform for studying how parallel algorithms that are known to scale well in C++ behave under a runtime with fundamentally different allocation, garbage collection, and task scheduling behavior.


----

***<center>The Challenge</center>***

The main challenge will be in getting good scale-up in the presence of OCaml's runtime and modifying the parallel strategies we use to better suit this platform. 
And otherwise understanding what prevents it from scaling. 
The bulk operations we're parallelizing fit mostly recursive divide-and-conquer, so they aren't too difficult to implement naively (we think). 
At each level, a pivot splits the problem into two subproblems over disjoint key ranges with no data dependency between them.
So there is clear parallelism opportunity with diminishing returns deeper in the recursion as the subproblems gets smaller. 
The memory access pattern is hard since a balanced BST uses pointers to maintain the structure, and each node might be in an arbitrary heap location.
So traversal has poor spatial locality and a high number of memory fetches compared useful computation (which is usually just a comparison).
Parallelizing, cores will chase pointers through the same shared heap and compete for cache lines and memory bandwidth. 
The workload is also irregular since splitting a tree at a pivot produces halves whose sizes depend on the data.
So subproblems might be very unbalanced and the resulting load imbalance is not predictable ahead of time. 

On top of this, the OCaml 5 runtime adds constraints that don't exist in C++. 
Task creation is heavier since OCaml domains are OS-level threads, and will be more expensive to spawn than something like Cilk's lightweight tasks. 
We expect this to change the task granularity and maybe allow more parallelism. 
Also, garbage collection has to coordinate across the OCaml domains, since each domain has a private minor heap, but accessing the major heap requries it to stop the world across all domains. And since purely functional tree operations allocate a new node for every change in structure, heavy allocation might trigger expensive GC pauses.
Another notable detail is that OCaml's uniform boxed representation also means every tree node is a separate heap object accessed through a pointer with header overhead, which is less compact than C++ implementations that control memory layout directly. 

From this project we hope to understand concretely which of these factors (task overhead, GC coordination, cache behavior, load imbalance, or something else) dominates in practice, and at what input sizes/core counts each becomes the main constraint. And how much of the performance gap relative to C++ is fundamental vs by the runtime.

----

***<center>Resources</center>***

In terms of hardware, we plan to benchmark on the GHC cluster machines with multicore CPUs for shared-memory parallelism. 
Our starting point will be OCaml's standard library `Set` and `Map` implementations (pure functional AVL trees) and it will also be the sequential baseline. 
This source is publicly available in the OCaml compiler repository.
 
We'll also be looking at the PAM C++ library (https://github.com/cmuparlay/PAM) for performance comparison on the same operations and input sizes, but on an imperative implementation. 
We'll also take a look at the accompanying paper(s) published by the authors of this library as they have some insights from the algorithmic and C++ side of this problem (though they use Cilk and Cilk-like approaches for their implementation).  

----

***<center>Goals and Deliverables</center>***

***Plan to Achieve.*** We'll deliver a parallel OCaml 5 library implementing `union`, `intersection`, `difference`, and `filter` for ordered sets with correctness against OCaml's standard library sequential implementations. For our different implementation approaches we'll do a experimental evaluation covering:
- <u>Speedup and scaling</u> including wall-clock speedup measurements across core counts (1, 2, 4, 8, 16+ cores) for each operation, on input sizes from thousands to millions of elements. We don't have a precise speedup target at the moment but we expect meaningful speedup for large inputs given the high available parallelism in the algorithms.
- <u>Granularity threshold sensitivity</u>, measuring performance as a function of the cutoff size where run the sequential version and don't fork (if we do the straight-up fork-and-join approach). The goal is to identify where the best config is and explain what determines it in OCaml's runtime (task creation cost, GC interaction, etc.).
- <u>Understand Bottlenecks</u> with concrete measurements of what limits scaling. We'll use OCaml's GC statistics, timing, and hardware performance counters where possible to differentiate between overhead of the GC, task creation, cache pressure, and load imbalance.
- Performance comparison against the C++ PAM library (without the same runtime) on the same operations and input sizes. Analyze what specific runtime factors (GC, allocation model, task granularity, memory layout) account for the difference in performance.

We believe the implementation is achievable because the sequential algorithmic structure is pretty standard and OCaml 5's domain API provides the parallel fork-join primitives we think we'll need. The majority of the project effort will be in making the parallel implementation fast and understanding why it isn't faster (presumably mostly due to the runtime?).

***Hope to Achieve.***
- Compare multiple parallelization strategies and analyzing which performs better and why. Some ideas might include adding fork-join parallelism directly to OCaml's existing `Set` implementation if possible, using similar approaches to the C++ PAM library and their algorithmic framework, or something else more/less naive.
- Extend the library from sets to ordered maps (we suspect this will mostly just require additional bookkeeping for values)
- Look into implementing parallelized augmented maps (ordered maps where each subtree caches an aggregate value maintained throughout the mutating operations) and show a simple application (like range queries)

***If Work is Slow.*** If the full set of bulk operations is too much, we'll focus only on `union` with more focus on the performance analysis. The goal will be a single parallel operation with clear speedup measurements, bottleneck diagnosis, and a comparison against C++ PAM. Try to understand as much as possible from this one operation (since the others presumably behave quite similarly).

----

***<center>Platform Choice</center>***

We're using OCaml 5 on shared-memory multicore CPUs for the following reasons:
 
1. <u>Good fit between functional trees and fork-join parallelism.</u> Purely functional trees have no mutation + no shared mutable state. Parallelism comes entirely from forking independent recursive subproblems. There are no data races (by construction) and no need for locks/synchronization. This makes correctness easy and lets us focus on performance.
 
2. <u>Interesting runtime contrast with C++.</u> The PAM library shows these algorithms scale well in C++. OCaml's runtime is fundamentally different, with a GC that must coordinate across domains, tasks are heavier, and allocation that creates many short-lived objects. We can try to isolate the effect of these runtime and architectural differences on parallel performance.
 
3. <u>The GHC cluster machines</u> provide the shared-memory multicore hardware needed to measure parallel scaling (and we believe have OCaml set up? If not, we can find alternative hardware platforms as needed).
   

----

***<center>Tentitative Schedule</center>***

* Week of 3/30 - 4/5
  - Understand the source code of OCaml's standard library `Set` and potentially the algorithms from the literature we mentioned above. Understand both approaches and identify where they are different.
  - Set up OCaml 5 dev environment, benchmarking, correctness checks, and potentially C++ PAM for our comparisons later.
  - Implement sequential `join` and `split` for AVL trees. Implement a first parallel `union` using OCaml 5 domains with a simple granularity cutoff. Make sure it's correct.
* Week of 4/6 - 4/12
  - Implement parallel `intersection`, `difference`, and `filter` and keep tweaking the previous implementations as we go along.
  - Begin benchmarking (speedup vs. core count, effect of input size, initial granularity threshold sweep).
  - Prepare milestone report with preliminary results and updated plan.
* Week of 4/13 - 4/19
  - Keep benchmarking and iterating the design. Discuss next steps. 
  - Figure out main bottlenecks with OCaml's GC statistics and timing, etc.
  - Run comparisons against C++ PAM on the same operations and input sizes.
  - If ahead of schedule: begin stretch goals (maps, augmented maps, etc).
* Week of 4/20 - 4/26
  - Optimize and finalize all benchmarks and analysis. Make final speedup graphs, granularity threshold plots, and comparison figs.
  - Write final report and prepare poster. 
  - (Final report 4/30, poster session 5/1).

</div>
