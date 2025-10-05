+++
author = "Andrei Pöhlmann"
title = "Measuring Kernel Memory Latency with Pointer Chasing"
date = "2025-09-21"
description = "Following up on the CPU cache throughput experiment: using a pointer-chasing benchmark to measure the true serialized cost of a DRAM access, without the masking effect of memory-level parallelism."
tags = [
    "linux",
    "performance",
    "cpu",
    "cache",
    "latency",
    "benchmarking",
    "perf"
]
categories = [
    "System Performance"
]
+++

In the [previous post](/post/cpu-cache/), we measured the amortized throughput cost of cache misses using a stride-based benchmark. That test showed a dramatic 60x slowdown when cache-unfriendly access patterns forced the CPU to fetch data from main memory. The final number was **8 nanoseconds** per miss.

But there was an important caveat: that 8ns figure represents throughput, not latency. Modern CPUs hide memory latency through out-of-order execution and by handling many independent memory requests simultaneously. The stride test saturated the memory system with parallel requests, so the 8ns was the amortized cost per operation when dozens of memory accesses are in flight.

The true latency of a single DRAM access is much higher. To measure it, we need a different kind of test—one where each memory access depends on the result of the previous one, creating a serial dependency chain that defeats all the CPU's tricks for hiding latency.

## The Pointer-Chasing Benchmark

The classic technique for measuring true memory latency is pointer chasing. Instead of accessing memory in a predictable pattern, we build a linked structure where each node points to the next. The CPU must load one pointer before it knows where to look next, creating a true dependency that prevents overlapping requests.

Here's the implementation:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

typedef struct Node {
    unsigned next;
    unsigned char pad[64 - sizeof(unsigned)];
} Node;

#define ARRAY_SIZE_MB 256
#define STEPS 20000000

// Fisher-Yates shuffle to create a random permutation
void shuffle(unsigned *indices, unsigned n) {
    for (unsigned i = n - 1; i > 0; i--) {
        unsigned j = rand() % (i + 1);
        unsigned temp = indices[i];
        indices[i] = indices[j];
        indices[j] = temp;
    }
}

int main(int argc, char *argv[]) {
    int random_pattern = (argc > 1 && strcmp(argv[1], "random") == 0);
    int warmup = (argc > 2 && strcmp(argv[2], "warmup") == 0);
    
    unsigned long num_nodes = (ARRAY_SIZE_MB * 1024UL * 1024UL) / sizeof(Node);
    Node *nodes = malloc(num_nodes * sizeof(Node));
    if (!nodes) return 1;
    
    unsigned *indices = malloc(num_nodes * sizeof(unsigned));
    if (!indices) return 1;
    
    // Initialize indices
    for (unsigned i = 0; i < num_nodes; i++) {
        indices[i] = i;
    }
    
    if (random_pattern) {
        srand(42);
        shuffle(indices, num_nodes);
    }
    
    // Build the pointer chain
    for (unsigned i = 0; i < num_nodes - 1; i++) {
        nodes[indices[i]].next = indices[i + 1];
    }
    nodes[indices[num_nodes - 1]].next = indices[0];
    
    // Warmup pass
    if (warmup) {
        unsigned idx = 0;
        for (unsigned i = 0; i < num_nodes; i++) {
            idx = nodes[idx].next;
        }
    }
    
    // Timed pointer chase
    unsigned idx = 0;
    for (unsigned i = 0; i < STEPS; i++) {
        idx = nodes[idx].next;
    }
    
    // Prevent optimization
    printf("Final index: %u\n", idx);
    
    free(indices);
    free(nodes);
    return 0;
}
```

The code allocates 256MB of memory, far larger than any on-chip cache. Each `Node` is exactly 64 bytes—one cache line—so every access forces a new cache line fetch. The nodes are linked into a single cycle using either:
- A random permutation (unfriendly): forces unpredictable jumps through memory
- Sequential order (friendly): allows the prefetcher to help

The critical loop is `idx = nodes[idx].next`. Each iteration must wait for the previous load to complete before it can issue the next one. This serial dependency is what exposes true latency.

## Running the Test

First, compile with optimizations but without vectorization (which would inflate the instruction count without changing the memory behavior):

```bash
$ gcc -O2 -fno-tree-vectorize pointer_chase.c -o pointer_chase
```

### Unfriendly Pattern: Random Access

```bash
$ sudo perf stat -e cycles,instructions,l1d_cache_refill,l3d_cache_refill \
    ./pointer_chase random

Final index: 3831491

 Performance counter stats for './pointer_chase random':

     9,627,514,830      cycles                                                                
     1,112,317,747      instructions                     #    0.12  insn per cycle            
        31,471,765      l1d_cache_refill                                                      
        46,771,688      l3d_cache_refill                                                      

       3.348841440 seconds time elapsed

       3.189412000 seconds user
       0.151066000 seconds sys
```

### Friendly Pattern: Sequential Access with Warmup

```bash
$ sudo perf stat -e cycles,instructions,l1d_cache_refill,l3d_cache_refill \
    ./pointer_chase sequential warmup

Final index: 3222784

 Performance counter stats for './pointer_chase sequential warmup':

       734,540,411      cycles                                                                
       704,159,796      instructions                     #    0.96  insn per cycle            
        11,361,840      l1d_cache_refill                                                      
        25,794,903      l3d_cache_refill                                                      

       0.256324908 seconds time elapsed

       0.123884000 seconds user
       0.131877000 seconds sys
```

## The Results

The difference is stark. Both tests perform exactly 20 million pointer chases, but their performance is radically different.

| Metric | Random (Cold) | Sequential (Warm) | Ratio |
| :--- | :--- | :--- | :--- |
| **Time** | 3.35 seconds | 0.26 seconds | **13.1x faster** |
| **Cycles** | 9.63 billion | 735 million | **13.1x** |
| **Cycles/Hop** | **481** | **37** | **13.0x** |
| **L3 Refills** | 46.8 million | 25.8 million | **1.8x fewer** |
| **IPC** | 0.12 | 0.96 | **8x** |

The random pattern forces the CPU to wait **481 cycles** per hop. Using the effective clock frequency (9.63B cycles / 3.35s ≈ 2.87 GHz), that's:

**481 cycles / 2.87 GHz = 167 nanoseconds per memory access**

The sequential pattern, even though it's traversing the same 256MB structure, benefits enormously from hardware prefetching. Each hop costs only **37 cycles (12.8 ns)** because the prefetcher is loading the next several cache lines before they're needed.

## Throughput vs. Latency

This 167ns figure is the true cost of a serialized DRAM access on this system. It's **21 times higher** than the 8ns amortized cost measured in the stride experiment.

The difference illustrates a fundamental aspect of modern CPU memory systems:
- **Throughput** measures how much total work can be done when many independent operations are in flight. Modern CPUs excel at this through out-of-order execution and memory-level parallelism.
- **Latency** measures how long a single dependent operation takes when nothing else can proceed until it completes.

Real applications fall somewhere between these extremes. Code with many independent memory accesses (think: parallel loops over different arrays) operates closer to the throughput model. Code with long dependency chains (think: linked list traversals, tree walks) operates closer to the latency model.

The 0.12 IPC in the random pointer chase shows the CPU is stalled 88% of the time, waiting for memory. The hardware is capable of retiring over 4 instructions per cycle when fed properly, but here it's starved, executing roughly one instruction every 8 cycles. The sequential case achieves 0.96 IPC—the prefetcher keeps the pipeline full.

This is why pointer-heavy data structures can be so devastating to performance. A binary search tree traversal pays this 167ns penalty at every level. A hash table with chaining pays it on every collision. The same amount of data stored in a simple array and accessed sequentially might be 13x faster, not because the CPU is faster, but because the prefetcher makes the memory system faster.

## What This Means

It's worth noting that the 8ns cost from the stride benchmark and this 167ns figure measure different things. The stride test had cache-unfriendly access (STRIDE=256, IPC of 0.30), but the CPU could still overlap many independent memory requests through out-of-order execution. That test measured **throughput**: total cycles divided by total operations when the memory system is saturated with parallel requests.

This pointer-chasing test removes that parallelism entirely. Each access depends on the previous one, so the CPU can't issue the next load until the current one completes. That's why we see 167ns here versus 8ns there—it's not that this test is worse, it's that we're measuring the **latency** of a single serialized hop instead of amortized throughput across many parallel accesses.

Most real code falls between these extremes. A loop iterating through an array and doing independent work on each element? The CPU can overlap dozens of memory requests, so it operates closer to the throughput model. Walking a linked list or traversing a tree where each step depends on the last? Those are serialized chains, closer to the latency model.

The 21x gap between these numbers (8ns vs 167ns) shows how much modern CPUs can hide memory delays through parallelism—but only when the code structure allows it. Write algorithms with long dependency chains and all that hardware sophistication sits idle.



