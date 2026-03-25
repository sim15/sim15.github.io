---
layout: noheader
title: "15-418 project proposal"
permalink: /418-project/proposal/
---

<center><u>Project Proposal: Parallel Ordered Sets and Maps in OCaml via Join-Based Algorithms</u></center>


<div style="text-align: justify;" markdown="1">
***<center><u>Summary</u></center>***

<!-- 
We'll implement parallel versions of ordered set and map data structures in OCaml 5, based on the join-based algorithmic framework developed by Blelloch, Ferizovic, and Sun. OCaml's standard library provides ordered sets and maps backed by AVL trees, but all operations are sequential. We'll parallelize the bulk operations — `union`, `intersection`, `difference`, and `filter` — using OCaml 5's multicore support (domains), then measure speedup, diagnose bottlenecks, and compare against the existing C++ PAM library to understand how language runtime characteristics (garbage collection, allocation model, task creation overhead) affect parallel performance on shared-memory multicore hardware.

We'll investigate how to effectively parallelize bulk operations on balanced binary search trees (BSTs) (specifically `union`, `intersection`, `difference`, and `filter`) in a purely functional setting. Functional languages represent ordered sets and maps using persistent (immutable) balanced trees, and their bulk operations are entirely sequential despite having significant inherent parallelism. We'll implement and evaluate parallel strategies for these operations using OCaml 5's multicore support as our platform, measure how well they scale, and analyze what runtime and architectural factors limit parallel performance compared to analogous C++ implementations. -->


Ordered sets and maps are among the most widely used data structures, and in functional languages are implemented as purely functional balanced binary search trees. 
Their bulk operations (`union`, `intersection`, `difference`, `filter`) have significant inherent parallelism (independent recursive subproblems that touch disjoint data), yet standard library implementations in languages like OCaml are fully sequential. 
We're interested in parallelizing these operations using OCaml 5's shared-memory multicore support (`domains`) as our platform. 
OCaml is an interesting environment because its runtime characteristics (a coordinating garbage collector, heavyweight task creation relative to C++ work-stealing runtimes, and heavy allocation from persistent/immutable data structures) create bottlenecks that don't exist in C++ implementations.
We'll measure speedup and scaling behavior, figure out what specific runtime and architectural factors limit parallel performance, and compare against both the sequential OCaml standard library and the C++ PAM library to isolate how much of the performance gap is algorithmic versus being imposed by the runtime.

----

***<u><center>Background</center></u>***
 
<!-- An **ordered set** is a collection of keys from a totally ordered type (e.g., integers), stored in a balanced binary search tree (BST) so that an in-order traversal produces the keys in sorted order. An **ordered map** pairs each such key with a chosen value. 
OCaml's standard library `Set` and `Map` modules, Haskell's `Data.Set` and `Data.Map`, and Scala's `TreeSet`/`TreeMap` are all implemented this way.
In a **purely functional** (persistent) implementation, operations never mutate the existing tree. Instead, they return a new tree that shares most of its structure with the old one. This is important: it means there are no data races by construction, and old versions of the tree remain valid. 
OCaml's standard library uses AVL trees (a self-balancing BST where the heights of any node's two subtrees differ by at most 1) with this persistent approach.
The standard operations on these structures include `insert`, `delete`, `lookup`, `split`, `union`, `intersection`, `difference`, and `filter`. The "bulk" operations (`union`, `intersection`, `difference`, `filter`) are interesting parallelization targets because they must process potentially every element in both input trees and contain recursive subproblems that may be independent.

 -->


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


<!-- 
# Proposal: Parallel Ordered Sets and Maps in OCaml via Join-Based Algorithms

## Summary

We'll implement parallel versions of ordered set and map data structures in OCaml 5, based on the join-based algorithmic framework developed by Blelloch, Ferizovic, and Sun. OCaml's standard library provides ordered sets and maps backed by AVL trees, but all operations are sequential. We'll parallelize the bulk operations — `union`, `intersection`, `difference`, and `filter` — using OCaml 5's multicore support (domains), then measure speedup, diagnose bottlenecks, and compare against the existing C++ PAM library to understand how language runtime characteristics (garbage collection, allocation model, task creation overhead) affect parallel performance on shared-memory multicore hardware.

## Background

### The Data Structures

An **ordered set** is a collection of keys from a totally ordered type (e.g., integers), stored in a balanced binary search tree (BST) so that an in-order traversal produces the keys in sorted order. An **ordered map** is the same idea but each key is paired with a value. These are fundamental data structures — OCaml's standard library `Set` and `Map` modules are exactly this, implemented using AVL trees (a self-balancing BST where the heights of any node's two subtrees differ by at most 1).

The standard operations on these structures are `insert`, `delete`, `lookup`, `split`, `union`, `intersection`, `difference`, and `filter`. Of these, the "bulk" operations — `union`, `intersection`, `difference`, `filter` — are the ones with enough internal work to benefit from parallelism, since they must process potentially every element in both input trees.

### The Join-Based Framework

The key insight from the literature (Blelloch et al., "Joinable Parallel Balanced Binary Trees," SPAA 2016 / ACM TOPC 2022) is that all of these bulk operations can be built on top of two primitives:

- **`join(L, k, R)`**: Takes a left tree `L`, a key `k`, and a right tree `R` where all keys in `L` < `k` < all keys in `R`, and produces a single valid balanced BST. This is the only function that needs to know about the balancing scheme (AVL rotations, etc.). It runs in O(|height difference|) time.

- **`split(T, k)`**: Takes a tree `T` and a key `k`, and returns `(L, present, R)` where `L` contains all keys less than `k`, `R` contains all keys greater than `k`, and `present` indicates whether `k` was in `T`. This is built from `join`.

Once you have `join` and `split`, `union` works as follows:

```
union(T1, T2) =
  if T1 is empty, return T2
  if T2 is empty, return T1
  let (k, _) = root of T1
  let (L2, _, R2) = split(T2, k)
  let L' = union(left(T1), L2)    -- independent of the next line
  let R' = union(right(T1), R2)   -- independent of the previous line
  return join(L', k, R')
```

The critical observation: the two recursive calls to `union` operate on completely disjoint data. `L'` is built from elements less than `k` in both trees, `R'` from elements greater than `k`. They share nothing. This means they can be computed **in parallel** by forking two independent tasks.

`intersection` and `difference` follow the same recursive pattern — split one tree by the root of the other, recurse independently on the two halves, combine results with `join`. `filter` similarly recurses on the two subtrees independently and combines.

### Why This Is a Good Parallelization Target

The parallelism here is **recursive, divide-and-conquer**: each level of recursion doubles the number of independent subproblems. For two trees of size `n`, the algorithm has O(n log n) work (total operations) and O(log² n) span (longest sequential dependency chain). This means there is a large amount of available parallelism for large inputs — the ratio of work to span grows polynomially with input size.

However, the parallelism is **irregular**: the two halves produced by a split may be very different sizes depending on where the pivot falls, so load balancing is not trivial. Additionally, every `join` allocates a new tree node (the data structure is purely functional / persistent), so the algorithm is allocation-heavy.

## The Challenge

There are several aspects that make this problem challenging and interesting from a parallel systems perspective:

**Granularity control.** Forking a parallel task has overhead — creating a domain or task, scheduling it, synchronizing on its result. For small subtrees, this overhead exceeds the benefit of parallelism. We need to find the right threshold: below some subtree size, stop forking and run sequentially. If the threshold is too low, we waste time on overhead. If it's too high, we leave parallelism on the table. Finding and characterizing this tradeoff is a core part of the project.

**Allocation pressure and garbage collection.** Purely functional trees allocate a new node for every `join`. In C++ (where the PAM library is implemented), allocation is cheap and there is no GC. In OCaml, every allocation interacts with the garbage collector. Under parallel execution with multiple domains, GC behavior becomes more complex — OCaml 5 uses a per-domain minor heap but a shared major heap. We expect GC to be a significant factor in parallel scaling, and diagnosing how much it matters is one of the key questions.

**Irregular workload and load balancing.** The split operation divides a tree at a particular key, and the resulting halves may be very unequal. This means the forked sub-problems may have very different amounts of work, leading to load imbalance. Understanding how this affects speedup, and whether it can be mitigated, is another question.

**Memory access patterns.** Tree data structures are pointer-heavy. Traversing a tree means following pointers that may point anywhere in memory, giving poor spatial locality compared to array-based structures. Under parallel execution on a multicore machine, this interacts with the cache hierarchy — each core's cache will see many misses. We want to measure and understand how this affects scaling.

**Comparison with C++ runtime model.** The PAM library uses C++ with Cilk (a work-stealing scheduler) where spawning a parallel task is extremely lightweight. OCaml 5's domains are heavier-weight. This difference in task creation cost directly affects the optimal granularity threshold, the achievable speedup, and the scalability ceiling. Comparing the two runtimes on the same algorithmic framework isolates the effect of the language/runtime on parallel performance.

## Resources

**Hardware:** We'll use the GHC cluster machines, which have multicore CPUs suitable for shared-memory parallelism.

**Starting code:** We'll start from OCaml's standard library `Set` and `Map` implementations (pure functional AVL trees) as the sequential baseline. The source is publicly available in the OCaml compiler repository. We'll implement the parallel versions ourselves.

**Reference implementation:** The PAM C++ library (https://github.com/cmuparlay/PAM) will serve as a reference for correctness testing and performance comparison.

**Key references:**

1. Blelloch, Ferizovic, Sun. "Joinable Parallel Balanced Binary Trees." ACM TOPC, 2022. — The algorithmic framework: how `join`/`split` work, how bulk operations are parallelized, work/span bounds.
2. Sun, Ferizovic, Blelloch. "PAM: Parallel Augmented Maps." PPoPP 2018. — The library design and interface, practical performance results in C++.
3. Sun. "Join-based Parallel Balanced Binary Trees." PhD Thesis, CMU, 2018. — Comprehensive reference for algorithmic details.
4. Sun, Blelloch. "Implementing Parallel and Concurrent Tree Structures." PPoPP 2019 Tutorial. — High-level overview of the framework.

**Language/tools:** OCaml 5.x with the `Domain` module for parallelism. We may also use the `domainslib` library which provides a task pool and parallel primitives on top of raw domains.

## Goals and Deliverables

### Plan to Achieve

- A working OCaml library implementing parallel `union`, `intersection`, `difference`, and `filter` for ordered sets, built on `join` and `split`, using OCaml 5 domains for parallelism.
- Correctness validation against OCaml's standard library sequential implementations on a range of input sizes and distributions.
- Speedup measurements across core counts (1, 2, 4, 8, 16+ cores) for each operation, on input sizes ranging from small (thousands) to large (millions of elements).
- Analysis of granularity threshold: performance as a function of the sequential cutoff size, identifying the sweet spot and explaining why it falls where it does.
- Analysis of scaling bottlenecks: is the limitation GC pressure, task creation overhead, load imbalance, or cache behavior? Provide measurements to support conclusions.
- Comparison of our OCaml implementation's performance against the C++ PAM library on the same operations and input sizes, with discussion of what runtime/language factors explain the differences.

### Hope to Achieve

- Extend from sets to maps (maps add key-value pairs; the algorithms are nearly identical but involve slightly more bookkeeping).
- Implement augmented maps — adding a cached subtree aggregate value maintained through `join` — and demonstrate a simple application such as range-sum queries.
- Investigate whether alternative parallelism strategies (e.g., `domainslib` task pools vs. raw domain spawning) affect performance characteristics.

### If Work Goes Slowly

- Focus on parallel `union` only as the primary operation, with thorough performance analysis.
- Deliver the comparison with sequential OCaml and C++ PAM on `union` alone, with detailed bottleneck analysis.

### Poster Session

We plan to show speedup graphs across core counts and input sizes, a granularity threshold sensitivity plot, and a comparative chart of OCaml vs. C++ PAM performance. The key narrative is: here is a well-studied parallel algorithm with known good performance in C++, and here is what happens when you implement it in a language with a very different runtime model.

## Platform Choice

OCaml 5 on shared-memory multicore CPUs is the right platform for this project for several reasons:

1. **OCaml's standard library already uses AVL trees for sets and maps**, so we have a natural sequential baseline to compare against. We're parallelizing something that OCaml programmers actually use.

2. **OCaml 5 introduced multicore support** via domains and effect handlers, making shared-memory parallelism possible for the first time in OCaml. This is a relatively new capability (stable since 2022), and there is genuine open interest in understanding how well parallel algorithms perform under OCaml's runtime model.

3. **The contrast with C++ is the interesting part.** The PAM library demonstrates that these algorithms scale well in C++ with Cilk's lightweight work-stealing. OCaml's runtime has fundamentally different characteristics: a garbage collector that must coordinate across domains, heavier-weight task creation, and a purely functional allocation model that produces many short-lived objects. Running the same algorithm on both platforms isolates the effect of these runtime differences, which is directly relevant to the course theme of understanding how hardware and system characteristics affect parallel performance.

4. **Purely functional trees are natural for fork-join parallelism.** Because the data structure is never mutated (every operation produces a new tree), there are no data races and no need for locks or synchronization. The parallelism comes entirely from the algorithmic structure — forking independent recursive calls. This fits cleanly into the fork-join model discussed in the course.

## Rough Schedule

### Week of 3/30 – 4/5
- Complete literature review: read the Joinable Trees paper (algorithmic details for AVL join/split/union) and the PPoPP tutorial (high-level framework overview).
- Study OCaml's standard library `Set` source code. Understand how their existing `union`, `inter`, `diff`, `split` work.
- Set up OCaml 5 development environment and benchmark harness. Verify we can create domains and measure wall-clock time reliably.
- Implement sequential `join` and `split` for AVL trees following the paper's algorithm (replacing OCaml's existing approach with the join-based structure). Test for correctness.

### Week of 4/6 – 4/12
- Implement parallel `union` using OCaml 5 domains, with a configurable granularity threshold.
- Implement parallel `intersection`, `difference`, and `filter` following the same pattern.
- Begin preliminary benchmarking: measure speedup on a few input sizes and core counts. Identify obvious performance problems.
- Prepare milestone report with initial results.

### Week of 4/13 – 4/19
- Systematic performance evaluation: sweep across input sizes, core counts, and granularity thresholds.
- Profile and diagnose scaling bottlenecks (GC time, allocation rates, task overhead). Use OCaml's runtime statistics and profiling tools.
- Run comparison benchmarks against C++ PAM on the same workloads.
- Begin writing the analysis sections of the final report.

### Week of 4/20 – 4/26
- If ahead of schedule: implement maps and/or augmented maps (stretch goals).
- Finalize all benchmarks and analysis. Produce final graphs and figures.
- Write final report and prepare poster materials. -->
</div>