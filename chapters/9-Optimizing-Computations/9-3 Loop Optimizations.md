## Loop Optimizations {#sec:LoopOpts}

Loops are at the heart of nearly all high-performance programs. Since loops represent a piece of code that is executed a large number of times, they account for the majority of the execution time. Small changes in such a critical piece of code may have a large impact on the performance of a program. That's why it is so important to carefully analyze the performance of hot loops in a program and know possible ways to improve them.

In this section, we will take a look at the most well-known loop optimizations that address the types of bottlenecks mentioned above. We first discuss low-level optimizations, in [@sec:LowLevelLoopOpts], that only move code around in a single loop. Next, in [@sec:LoopOptsHighLevel], we will take a look at high-level optimizations that restructure loops, which often affect multiple loops. Note, that what I present here is not a complete list of all known loop transformations. For more detailed information on each of the transformations discussed below, readers can refer to [@EngineeringACompilerBook] and [@OptimizingCompilersForModernArchs].

### Low-level Optimizations. {#sec:LowLevelLoopOpts}

Let's start with simple loop optimizations that transform the code inside a single loop: Loop Invariant Code Motion, Loop Unrolling, Loop Strength Reduction, and Loop Unswitching. Generally, compilers are good at doing such transformations; however, there are still cases when a compiler might need a developer's support. We will talk about that in subsequent sections.

**Loop Invariant Code Motion (LICM)**: an expression whose value never changes across iterations of a loop is called a *loop invariant*. Since its value doesn't change across loop iterations, we can move a loop invariant expression outside of the loop. We do so by storing the result in a temporary variable and using it inside the loop (see [@lst:LICM]).[^17] All decent compilers nowadays successfully perform LICM in the majority of cases.

Listing: Loop Invariant Code Motion

~~~~ {#lst:LICM .cpp}
for (int i = 0; i < N; ++i)             for (int i = 0; i < N; ++i) {
  for (int j = 0; j < N; ++j)    =>       auto temp = c[i];
    a[j] = b[j] * c[i];                   for (int j = 0; j < N; ++j)
                                            a[j] = b[j] * temp;
                                        }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Loop Unrolling**: an _induction variable_ is a variable in a loop, whose value is a function of the loop iteration number. For example, `v = f(i)`, where `i` is an iteration number. Instead of modifying an induction variable on each iteration, we can unroll a loop and perform multiple iterations for each increment of the induction variable (see [@lst:Unrol]).

Listing: Loop Unrolling

~~~~ {#lst:Unrol .cpp}
                                        int i = 0;
for (int i = 0; i < N; ++i)             for (; i+1 < N; i+=2) {
  a[i] = b[i] * c[i];          =>         a[i]   = b[i]   * c[i];
                                          a[i+1] = b[i+1] * c[i+1];
                                        }
                                        // remainder (when N is odd)
                                        for (; i < N; ++i)
                                          a[i] = b[i] * c[i];
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The primary benefit of loop unrolling is to perform more computations per iteration. At the end of each iteration, the index value must be incremented and tested, then we go back to the top of the loop if it has more iterations to process. This work is commonly referred to as "loop overhead" or "loop tax", and it can be reduced with loop unrolling. For example, by unrolling the loop in [@lst:Unrol] by a factor of 2, we reduce the number of executed compare and branch instructions by 2x.

I do not recommend unrolling loops manually except in cases when you need to break loop-carry dependencies as shown in [@lst:DepChain]. First, because compilers are very good at doing this automatically and usually can choose the optimal unrolling factor. The second reason is that processors have an "embedded unroller" thanks to their out-of-order speculative execution engine (see [@sec:uarch]). While the processor is waiting for long-latency instructions from the first iteration to finish (e.g. loads, divisions, microcoded instructions, long dependency chains), it will speculatively start executing instructions from the second iteration and only wait on loop-carried dependencies. This spans multiple iterations ahead, effectively unrolling the loop in the instruction Reorder Buffer (ROB). The third reason is that unrolling too much could lead to negative consequences due to code bloat.

**Loop Strength Reduction (LSR)**: replace expensive instructions with cheaper ones. Such transformation can be applied to all expressions that use an induction variable. Strength reduction is often applied to array indexing. Compilers perform LSR by analyzing how the value of a variable evolves across the loop iterations. In LLVM, it is known as Scalar Evolution (SCEV). In [@lst:LSR], it is relatively easy for a compiler to prove that the memory location `b[i*10]` is a linear function of the loop iteration number `i`, thus it can replace the expensive multiplication with a cheaper addition.

Listing: Loop Strength Reduction

~~~~ {#lst:LSR .cpp}
for (int i = 0; i < N; ++i)             int j = 0;
  a[i] = b[i * 10] * c[i];      =>      for (int i = 0; i < N; ++i) {
                                          a[i] = b[j] * c[i];
                                          j += 10;
                                        }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Loop Unswitching**: if a loop has a conditional statement inside and it is invariant, we can move it outside of the loop. We do so by duplicating the body of the loop and placing a version of it inside each of the `if` and `else` clauses of the conditional statement (see [@lst:Unswitch]). While loop unswitching may double the amount of code written, each of these new loops may now be optimized separately.

Listing: Loop Unswitching

~~~~ {#lst:Unswitch .cpp}
for (i = 0; i < N; i++) {               if (c)
  a[i] += b[i];                           for (i = 0; i < N; i++) {
  if (c)                       =>           a[i] += b[i];
    b[i] = 0;                               b[i] = 0;
}                                         }
                                        else
                                          for (i = 0; i < N; i++) {
                                            a[i] += b[i];
                                          }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### High-level Optimizations. {#sec:LoopOptsHighLevel}

High-level loop transformations change the structure of loops and often affect multiple nested loops. In this section, we will take a look at Loop Interchange, Loop Blocking (Tiling), Loop Fusion and Distribution (Fission), and Loop Unroll and Jam. This set of transformations aims at improving memory access and eliminating memory bandwidth and memory latency bottlenecks. From a compiler perspective, it is very difficult to prove the legality of such transformations and justify their performance benefit. In that sense, developers are in a better position since they only have to care about the legality of the transformation in their particular piece of code. Unfortunately, this means usually we have to do such transformations manually.

**Loop Interchange**: is a process of exchanging the loop order of nested loops. The induction variable used in the inner loop switches to the outer loop, and vice versa. [@lst:Interchange] shows an example of interchanging nested loops for `i` and `j`. The main purpose of this loop interchange is to perform sequential memory accesses to the elements of a multi-dimensional array. By following the order in which elements are laid out in memory, we can improve the spatial locality of memory accesses and make our code more cache-friendly. This transformation helps to eliminate memory bandwidth and memory latency bottlenecks.

Listing: Loop Interchange

~~~~ {#lst:Interchange .cpp}
for (i = 0; i < N; i++)                 for (j = 0; j < N; j++)
  for (j = 0; j < N; j++)          =>     for (i = 0; i < N; i++)
    a[j][i] += b[j][i] * c[j][i];           a[j][i] += b[j][i] * c[j][i];
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Interchange is only legal if loops are *perfectly nested*. A perfectly nested loop is one wherein all the statements are in the innermost loop. Interchanging imperfect loop nests is harder to do but still possible; check an example in the [Codee](https://www.codee.com/catalog/glossary-perfect-loop-nesting/)[^1] catalog.

**Loop Blocking (Tiling)**: the idea of this transformation is to split the multi-dimensional execution range into smaller chunks (blocks or tiles) so that each block will fit in the CPU caches. If an algorithm works with large multi-dimensional arrays and performs strided accesses to their elements, there is a high chance of poor cache utilization. Every such access may push the data that will be requested by future accesses out of the cache (cache eviction). By partitioning an algorithm into smaller multi-dimensional blocks, we ensure the data used in a loop stays in the cache until it is reused.

In the example shown in [@lst:blocking], an algorithm performs row-major traversal of elements of array `a` while doing column-major traversal of array `b`. The loop nest can be partitioned into smaller blocks to maximize the reuse of elements in array `b`.

Listing: Loop Blocking

~~~~ {#lst:blocking .cpp}
// linear traversal                     // traverse in 8*8 blocks
for (int i = 0; i < N; i++)             for (int ii = 0; ii < N; ii+=8)
  for (int j = 0; j < N; j++)    =>      for (int jj = 0; jj < N; jj+=8)
    a[i][j] += b[j][i];                   for (int i = ii; i < ii+8; i++)
                                           for (int j = jj; j < jj+8; j++)
                                            a[i][j] += b[j][i];
                                        // remainder (not shown)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Blocking is a widely known method of optimizing GEneral Matrix Multiplication (GEMM) algorithms. It enhances the cache reuse of memory accesses and improves the memory bandwidth utilization and memory latency.

Typically, engineers optimize a tiled algorithm for the size of caches that are private to each CPU core (L1 or L2 for Intel and AMD, L1 for Apple). However, the sizes of private caches are changing from generation to generation, so hardcoding a block size presents its own set of challenges. As an alternative solution, you can use [cache-oblivious](https://en.wikipedia.org/wiki/Cache-oblivious_algorithm)[^2] algorithms whose goal is to work reasonably well for any size of the cache.

**Loop Fusion and Distribution (Fission)**: separate loops can be fused when they iterate over the same range and do not reference each other's data. An example of a Loop Fusion is shown in [@lst:fusion]. The opposite procedure is called Loop Distribution (Fission) when the loop is split into separate loops.

Listing: Loop Fusion and Distribution

~~~~ {#lst:fusion .cpp}
for (int i = 0; i < N; i++)             for (int i = 0; i < N; i++) {
  a[i].x = b[i].x;                        a[i].x = b[i].x;
                               =>         a[i].y = b[i].y;
for (int i = 0; i < N; i++)             }
  a[i].y = b[i].y;                      
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Fusion helps to reduce the loop overhead (similar to Loop Unrolling) since both loops can use the same induction variable. Also, loop fusion can help to improve the temporal locality of memory accesses. In [@lst:fusion], if both `x` and `y` members of a structure happen to reside on the same cache line, it is better to fuse the two loops since we can avoid loading the same cache line twice. This will reduce the cache footprint and improve memory bandwidth utilization.

However, loop fusion does not always improve performance. Sometimes it is better to split a loop into multiple passes, pre-filter the data, sort and reorganize it, etc. By distributing the large loop into multiple smaller ones, we limit the amount of data required for each iteration of the loop and increase the temporal locality of memory accesses. This helps in situations with a high cache contention, which typically happens in large loops. Loop distribution also reduces register pressure since, again, fewer operations are being done within each iteration of the loop. Also, breaking a big loop into multiple smaller ones will likely be beneficial for the performance of the CPU Frontend because of better instruction cache utilization. Finally, when distributed, each small loop can be further optimized separately by the compiler.

**Loop Unroll and Jam**: to perform this transformation, you need to unroll the outer loop first, then jam (fuse) multiple inner loops together as shown in [@lst:unrolljam]. This transformation increases the ILP (Instruction-Level Parallelism) of the inner loop since more independent instructions are executed inside the inner loop. In the code example, the inner loop is a reduction operation that accumulates the deltas between elements of arrays `a` and `b`. When we unroll and jam the loop nest by a factor of two, we effectively execute two iterations of the original outer loop simultaneously. This is emphasized by having two independent accumulators. This breaks the dependency chain over `diffs` in the initial variant.

Listing: Loop Unroll and Jam

~~~~ {#lst:unrolljam .cpp}
for (int i = 0; i < N; i++)           for (int i = 0; i+1 < N; i+=2)
  for (int j = 0; j < M; j++)           for (int j = 0; j < M; j++) {
    diffs += a[i][j] - b[i][j];   =>      diffs1 += a[i][j]   - b[i][j];
                                          diffs2 += a[i+1][j] - b[i+1][j];
                                        }
                                      diffs = diffs1 + diffs2;
                                      // remainder (not shown)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Unroll and Jam can be performed as long as there are no cross-iteration dependencies on the outer loops, in other words, two iterations of the inner loop can be executed in parallel. Also, this transformation makes sense if the inner loop has memory accesses that are strided on the outer loop index (`i` in this case), otherwise, other transformations likely apply better. The Unroll and Jam technique is especially useful when the trip count of the inner loop is low, e.g., less than 4. By doing the transformation, we pack more independent operations into the inner loop, which increases the ILP.

The Unroll and Jam transformation sometimes is very useful for outer loop vectorization, which, at the time of writing, compilers cannot do automatically. In a situation when the trip count of the inner loop is not visible to a compiler, the compiler could still vectorize the original inner loop, hoping that it will execute enough iterations to hit the vectorized code (more on vectorization in the next section). But if the trip count is low, the program will use a slow scalar version of the loop. By performing Unroll and Jam, we enable the compiler to vectorize the code differently: now "gluing" the independent instructions in the inner loop together. This technique is also known as Superword-Level Parallelism (SLP) vectorization.

### Discovering Loop Optimization Opportunities. {#sec:DiscoverLoopOpts}

As we discussed at the beginning of this section, compilers will do the heavy-lifting part of optimizing your loops. You can count on them to make all the obvious improvements in the code of your loops, like eliminating unnecessary work, doing various peephole optimizations, etc. Sometimes a compiler is clever enough to generate the fast version of a loop automatically, but other times we have to do some rewriting to help the compiler. As we said earlier, from a compiler's perspective, doing loop transformations legally and automatically is very difficult. Often, compilers have to be conservative when they cannot prove the legality of a transformation. 

As a first step, you can check compiler optimization reports or examine the machine code[^16] of the loop to search for easy improvements. Sometimes, it's possible to adjust compiler transformations using user directives. There are compiler pragmas that control certain transformations, like loop vectorization, loop unrolling, loop distribution, and others. For a complete list of user directives, check your compiler's manual.

Next, you should identify the bottlenecks in the loop and assess performance against the hardware theoretical maximum. The Top-down Microarchitecture Analysis (TMA, see [@sec:TMA]) methodology and the Roofline performance model ([@sec:roofline]) both help with that. The performance of a loop is limited by one or many of the following factors: memory latency, memory bandwidth, or the computing capabilities of a machine. Once the performance bottlenecks in a loop have been identified, try applying the required transformations that we discussed in a few previous sections.

Even though there are well-known optimization techniques for a particular set of computational problems, loop optimizations remain a "black art" that comes with experience. I recommend that you rely on your compiler and complement it with manually transforming code when necessary. Above all, keep the code as simple as possible and do not introduce unreasonably complicated changes if the performance benefits are negligible.

[^1]: Codee: perfect loop nesting - [https://www.codee.com/catalog/glossary-perfect-loop-nesting/](https://www.codee.com/catalog/glossary-perfect-loop-nesting/)
[^2]: Cache-oblivious algorithm - [https://en.wikipedia.org/wiki/Cache-oblivious_algorithm](https://en.wikipedia.org/wiki/Cache-oblivious_algorithm)
[^16]: Sometimes difficult, but always a rewarding activity.
[^17]: The compiler will perform the transformation only if it can prove that `a` and `c` don’t alias.
