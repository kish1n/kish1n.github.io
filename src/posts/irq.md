---
title: Is Core 0 Sabotaging Your Performance?
description: An investigation into the discrepancy in observed performance between cores on a modern CPU
permalink: posts/{{ title | slug }}/index.html
date: '2025-12-16'
updated: '2025-12-16'
tags: [cpu, code, vcache, amd, intel]
---

**TL;DR:** Not all CPU cores are equal—even beyond architectural differences like 3D V-Cache. On an AMD Ryzen 9 9950X3D, core 0 shows significantly worse tail latency due to the OS preferentially routing interrupts to it. For latency-sensitive applications, this hidden asymmetry can cause significant performance degradation.

## Introduction

It is commonly known that CPUs these days often have different types of cores. For example, Intel has "performance" P-cores and "efficiency" E-cores. AMD's high core count CPUs advertised with 3D V-Cache only have some cores with access to the additional L3 cache, with the rest having the standard amount of L3 cache.

But are there significant differences between cores beyond what is advertised? By complete accident, I found the answer to this is _most definitely_.

## Background

I was working on my tail latency microbenchmarking tool [cpptail](https://github.com/kish1n/cpptail) and added functionality to pin the program to a CPU core of the user's choice. This enables consistent benchmarking as different types of cores are ... well ... different.

To sanity-check my implementation, I made a test script to run a simple mathematical benchmark for each of the 32 logical cores on my AMD Ryzen 9 9950X3D. My expectation was for cores 0-7 and 16-23 to perform slightly slower than the others, as they belong to the CCD with 3D V-Cache stacked on top. The extra cache layers add thermal constraints that reduce boost clocks—a primary factor in simple numerical benchmarks. While the prediction proved mostly true, there was something in the data that piqued my interest.

### Test Configuration

- **CPU:** AMD Ryzen 9 9950X3D
- **OS:** Linux (CachyOS 6.18)
- **Benchmark Tool:** [cpptail](https://github.com/kish1n/cpptail)
- **Iterations:** 1,000,000 per core

## The Data

Time it takes to run the benchmark for each core. Units in **nanoseconds**.

| Core | Mean | p50 | p99 | p99.9 | | Core | Mean | p50 | p99 | p99.9 |
|------|------|-----|-----|-------|---|------|------|-----|-----|-------|
| 0    | 272  | 261 | 271 | 271   | | 16   | 263  | 261 | 271 | 301   |
| 1    | 262  | 261 | 271 | 271   | | 17   | 262  | 261 | 271 | 271   |
| 2    | 262  | 261 | 271 | 271   | | 18   | 262  | 261 | 271 | 271   |
| 3    | 262  | 261 | 271 | 271   | | 19   | 262  | 261 | 271 | 271   |
| 4    | 262  | 261 | 271 | 271   | | 20   | 262  | 261 | 271 | 271   |
| 5    | 262  | 261 | 271 | 271   | | 21   | 262  | 261 | 271 | 271   |
| 6    | 262  | 261 | 271 | 271   | | 22   | 262  | 261 | 271 | 271   |
| 7    | 262  | 261 | 271 | 271   | | 23   | 262  | 261 | 271 | 271   |
| 8    | 255  | 251 | 261 | 261   | | 24   | 255  | 251 | 261 | 261   |
| 9    | 255  | 251 | 261 | 261   | | 25   | 255  | 251 | 261 | 261   |
| 10   | 255  | 251 | 261 | 261   | | 26   | 255  | 251 | 261 | 261   |
| 11   | 255  | 251 | 261 | 290   | | 27   | 255  | 251 | 261 | 261   |
| 12   | 255  | 251 | 261 | 280   | | 28   | 255  | 251 | 261 | 261   |
| 13   | 255  | 251 | 261 | 261   | | 29   | 255  | 251 | 261 | 261   |
| 14   | 255  | 251 | 261 | 261   | | 30   | 255  | 251 | 261 | 261   |
| 15   | 255  | 251 | 261 | 261   | | 31   | 255  | 251 | 261 | 261   |

The 3D V-Cache cores (0-7, 16-23) consistently show ~262ns mean latency, while non-V-Cache cores (8-15, 24-31) achieve ~255ns—a **~2.7% performance difference**. This aligns with expectations: the 3D V-Cache adds thermal constraints that reduce boost clocks.

One core stands out dramatically: **core 0** with 272ns mean latency. Even more suspicious, its mean latency exceeds its p99.9 latency, suggesting the distribution is heavily skewed by outliers. Compared to the non-V-Cache cores, core 0 shows a **17ns (6.7%) latency penalty**—and compared to other V-Cache cores, it's still 10ns slower.

Repeated runs showed identical patterns. Something systematic was happening on core 0.

## Effective Frequency Of Each Core

After analyzing the results from a million iterations of the core 0 benchmark, I saw hundreds of execution times ranging from 1,000 to 23,000 nanoseconds—outliers that were completely absent from benchmarks of other cores. My immediate suspicion was context switches. Perhaps some stray process on my system was competing for core 0 during the benchmark. However, using the Linux `perf` utility, I found there were zero context switches, so that hypothesis was incorrect.

I then created a script to divide the cycles reported by `perf` by time to calculate what could be called an _effective frequency_ for each core. This metric represents only the cycles spent on the benchmarking process, not the total cycles the core executed. I also collected IRQ counts (interrupt requests from hardware and the kernel) for each core.

| Core | Freq (GHz) | IRQs | | Core | Freq (GHz) | IRQs |
|------|------------|------|-|------|------------|------|
| 0    | 5.33       | 111  | | 16   | 5.54       | 35   |
| 1    | 5.54       | 50   | | 17   | 5.53       | 46   |
| 2    | 5.54       | 29   | | 18   | 5.54       | 58   |
| 3    | 5.55       | 31   | | 19   | 5.54       | 45   |
| 4    | 5.54       | 33   | | 20   | 5.55       | 49   |
| 5    | 5.54       | 35   | | 21   | 5.54       | 57   |
| 6    | 5.53       | 62   | | 22   | 5.54       | 49   |
| 7    | 5.53       | 33   | | 23   | 5.54       | 43   |
| 8    | 5.68       | 35   | | 24   | 5.69       | 34   |
| 9    | 5.69       | 26   | | 25   | 5.69       | 27   |
| 10   | 5.69       | 27   | | 26   | 5.69       | 33   |
| 11   | 5.69       | 43   | | 27   | 5.69       | 31   |
| 12   | 5.69       | 36   | | 28   | 5.69       | 31   |
| 13   | 5.69       | 35   | | 29   | 5.70       | 40   |
| 14   | 5.69       | 34   | | 30   | 5.69       | 37   |
| 15   | 5.69       | 31   | | 31   | 5.69       | 28   |

## IRQs: Source of Tail Latency?

IRQs (Interrupt Requests) are treated specially by `perf`. Despite triggering a context switch to kernel interrupt handlers, saving processor state and suspending user-space execution, they don't increment the context switch counter. This explains why `perf` reported zero context switches!

The data reveals the root cause:

- **Core 0**: 111 IRQs during benchmark execution, 5.33 GHz effective frequency
- **Other 3D V-Cache cores**: ~30-60 IRQs, 5.53-5.55 GHz effective frequency  
- **Non-V-Cache cores**: ~25-45 IRQs, 5.68-5.70 GHz effective frequency

Core 0 receives **2-3× more interrupts** than other cores. These interrupts include:
- Timer ticks
- NIC (network interface) interrupts
- Disk I/O completions
- System management interrupts

Each interrupt suspends user-space execution for hundreds of nanoseconds to microseconds. The Linux kernel defaults to preferring the first core for a significant amount of its heavier IRQs, making core 0 the primary victim. Interestingly, the interrupts that did occur on other cores also seemed to have less of an impact on their tail latency suggesting *heavier* IRQs on core 0.

This is validated by examining perf timing breakdown:
- **User time**: Comparable across all cores (~1M operations worth)
- **System time**: Orders of magnitude higher on core 0 than other cores

The tail latency spikes (1,000-23,000ns) occur precisely when IRQs fire during benchmark execution.

## Conclusion

CPU cores that may be commonly assumed to be identical can behave very differently in practice. Operating system decisions—especially how interrupts are distributed—introduce measurable performance gaps between supposedly equal cores. For applications where nanoseconds matter, understanding and controlling CPU affinity and interrupt distribution is not optional. Linux gamers on dual-CCD AMD CPUs already often use `taskset -c 0-7,16-23 %command%` to pin a game to the 3D V-Cache cores. Perhaps it is time to experiment with `taskset -c 1-7,16-23 %command%`?

Further reading:
- [Linux IRQ Affinity Documentation](https://www.kernel.org/doc/html/latest/core-api/irq/irq-affinity.html)
- [cpptail - Tail Latency Microbenchmarking Tool](https://github.com/kish1n/cpptail)