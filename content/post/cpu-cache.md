+++
author = "Andrei Pöhlmann"
title = "A Practical Dive into CPU Cache Performance"
date = "2025-09-14"
description = "An empirical journey to measure the performance impact of CPU cache misses, detailing the process of benchmarking, encountering virtualization and compiler obstacles, and quantifying the final cost in nanoseconds."
tags = [
    "linux",
    "performance",
    "cpu",
    "cache",
    "benchmarking",
    "perf"
]
categories = [
    "Systems Programming"
]
+++

Reading LWN.net's article, "[Improving Linux networking performance](https://lwn.net/Articles/629155/)", highlights the immense pressure high-speed networking puts on a system. The article notes the time budget to process a single packet shrinks dramatically.

A quick calculation confirms this:
* **Link Speed:** 100 Gbps = 12.5 Gigabytes/sec
* **Packet Size (MTU):** 1538 bytes
* **Packets/sec:** `12,500,000,000 / 1538 ≈ 8.1 million`
* **Time/Packet:** `1 second / 8,100,000 ≈` **123 nanoseconds**

The article notes that to address this problem, one must scrutinize the cost of every single operation. The author gives a specific, motivating example:

> ...a cache miss on Jesper's 3GHz processor takes about 32ns to resolve.

This concrete number motivated me to run some experiments myself and to design a test to empirically measure this nanosecond penalty on a standard cloud VM.

## The Experiment

We can measure this using Linux's `perf` utility. The experiment uses a C program that allocates a large array and iterates through it with different access patterns.

The idea is based on the CPU's memory hierarchy. According to official ARM documentation, the Neoverse N1 CPU fetches data from RAM in **64-byte chunks** called **cache lines**. Accessing data already in a cache is a "hit"; accessing data that isn't is a "miss," which forces the CPU to stall and wait.

To exploit this, the C program allocates a **256MB** array, a size chosen to be orders of magnitude larger than the N1's L3 cache (which tops out at 4MB). It then uses a `STRIDE` macro to control the access pattern:
* **`STRIDE = 1`**: This sequential access pattern is designed to be highly efficient, as the CPU's hardware prefetcher can easily predict and load the next 64-byte cache line before it's needed.
* **`STRIDE = 256`**: This strided pattern jumps 256 bytes (4 cache lines) at a time, designed to defeat the prefetcher and cause a cache miss on almost every access.

```c
#include <stdio.h>
#include <stdlib.h>

#define ARRAY_SIZE (256 * 1024 * 1024)
#define STRIDE 256 // Or 1 for the friendly test

int main() {
    char *array = malloc(ARRAY_SIZE);
    if (!array) return 1;

    // Adjust outer loop to equalize total logical operations
    // For STRIDE=256: j < 5000
    // For STRIDE=1:   j < 20
    for (int j = 0; j < 5000; j++) {
        for (long i = 0; i < ARRAY_SIZE; i += STRIDE) {
            array[i] *= 3;
        }
    }

    // Add a checksum to prevent dead code elimination
    long sum = 0;
    for (long i = 0; i < ARRAY_SIZE; i += 1024) { 
        sum += array[i];
    }
    printf("Result sum: %ld\n", sum);

    free(array);
    return 0;
}
````

## Measurement Obstacles
Initial perf runs failed to collect hardware-specific data.
```Bash
$ perf stat -e cycles,instructions,cache-references,cache-misses,LLC-loads,LLC-load-misses ./cache_test
 Performance counter stats for './cache_test':
   <not supported>      LLC-loads
   <not supported>      LLC-load-misses
```
The first issue is a kernel security restriction, `perf_event_paranoid`, which must be lowered.

```Bash
$ sudo sysctl -w kernel.perf_event_paranoid=1
```
The second issue was revealed by lscpu.


```Bash
$ lscpu
Architecture:             aarch64
Model name:               Neoverse-N1
```
The test machine is an ARM-based AWS Graviton instance, not x86. Performance counter names are architecture-specific. `perf list` shows the correct event names for this platform, such as `l1d_cache_refill`.

## The Compiler Optimizer

An early version of the code without the final `printf` checksum was still producing counterintuitive results. The `-O2` flag (`gcc -O2 ...`) enables aggressive compiler optimizations. The compiler correctly identified that the main loop's results were never used and performed dead code elimination.

This can be proven by inspecting the generated assembly. The `-S` flag tells gcc to output assembly code.

```Bash
# Compile the version without the checksum/printf
gcc -O2 -S test.c -o cache_test_O2.s
```

The resulting assembly for the main function contains a call to malloc followed immediately by free. The loops are gone.

```ARM assembler

main:
...
	bl	malloc
	cbz	x0, .L3
	bl	free
	mov	w0, 0
...
```

Adding the checksum loop and printf creates a data dependency that the optimizer cannot ignore, forcing it to execute the main workload.


## The Definitive Result

To get a true apples-to-apples comparison, the workload is equalized. The `Stride=256` test performs `5000 * (256MB / 256) = ~5.24 billion` logical operations. To match this, the `Stride=1` test's outer loop is scaled down to `20` iterations (`20 * (256MB / 1) ≈ ~5.36 billion` ops).

#### Stride = 256 (Cache-Unfriendly)

```bash
$ sudo perf stat -e cycles,instructions,l3d_cache_refill ./cache_test
Result sum: 0

 Performance counter stats for './cache_test':

      126082994775      cycles
       37670101621      instructions      #    0.30  insn per cycle
        5281000266      l3d_cache_refill

      42.982450146 seconds time elapsed
```

#### Stride = 1 (Cache-Friendly, Equal Workload)

```bash
$ sudo perf stat -e cycles,instructions,l3d_cache_refill ./cache_test
Result sum: 0

 Performance counter stats for './cache_test':

        2030187345      cycles
        2696274808      instructions      #    1.33  insn per cycle
          85780773      l3d_cache_refill

       0.706392486 seconds time elapsed
```

For the same amount of logical work, the cache-friendly code was **60 times faster**.

| Metric | Stride = 256 (Unfriendly) | Stride = 1 (Friendly, Equal Work) | Result |
| :--- | :--- | :--- | :--- |
| **Logical Workload** | **\~5.24 Billion Ops** | **\~5.24 Billion Ops** | **Equal** |
| `Time Elapsed` | 42.98 seconds | **0.71 seconds** | **60x Faster** |
| `Instructions` | 37.7 billion | **2.7 billion** | **14x Fewer Instructions** |
| `IPC (Efficiency)`| 0.30 (Terrible) | **1.33 (Excellent)** | **4x More Efficient**|
| `L3 Cache Refills`| 5.3 billion | **85.8 million** | **98.4% Fewer Misses** |

The **IPC (Instructions Per Cycle)** tells the story of hardware efficiency. A high IPC means the CPU is working efficiently; a low IPC means it is frequently stalled. E.g. the CPU is stalled waiting for data from memory, its processing pipeline is effectively frozen.

  * The unfriendly code achieved a dismal **IPC of 0.30**. The CPU was stalled over 70% of the time.
  * The friendly code achieved an excellent **IPC of 1.33**. The hardware prefetcher kept the CPU pipeline full.

## Quantifying the Cost

We can now calculate the average cost of a last-level miss. The penalty is the difference between the cost of an operation that misses the cache and one that hits it.

**1. Cost of an Unfriendly Operation (a Miss)**

This calculation finds the average cost of a single operation when it's almost guaranteed to miss all caches. We use the data from the `Stride=256` test, which we proved had a nearly 100% miss rate.

  * **Formula:** `Total Cycles / Total Operations`
  * **Calculation:** `126.1B cycles / 5.24B ops ≈` **24.0 cycles**.

**2. Cost of a Friendly Operation (a Hit)**

This calculation finds the average cost when an operation is almost guaranteed to be a cache hit. We use the data from the `Stride=1` test, which had a \>98% hit rate.

  * **Formula:** `Total Cycles / Total Operations`
  * **Calculation:** `2.03B cycles / 5.36B ops ≈` **0.38 cycles**.

**3. The Penalty**
The final cost of a miss is the difference between these two scenarios.

  * **Calculation:** `24.0 cycles (miss) - 0.38 cycles (hit) =` **23.62 cycles**.

The internet suggests a typical clock speed of 2.5 GHz for the Neoverse-N1. Using this value:
`23.62 cycles / 2.5 GHz =` **\~9.4 nanoseconds**.

That's the price. Every trip to main memory costs roughly 9.4 nanoseconds, during which a multi-gigahertz processor is stalled. This journey demonstrates that cache performance isn't a theoretical concern; it's a concrete factor that can alter application performance by orders of magnitude.