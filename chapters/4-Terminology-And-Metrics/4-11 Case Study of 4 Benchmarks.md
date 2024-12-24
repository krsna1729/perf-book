## Case Study: Analyzing Performance Metrics of Four Benchmarks {#sec:PerfMetricsCaseStudy}

To put together everything we discussed so far in this chapter, let's look at some real-world examples. We ran four benchmarks from different domains and calculated their performance metrics. First of all, let's introduce the benchmarks.

1. Blender 3.4 - an open-source 3D creation and modeling software project. This test is of Blender's Cycles performance with the BMW27 blend file. All hardware threads are used. URL: [https://download.blender.org/release](https://download.blender.org/release). Command line: `./blender -b bmw27_cpu.blend -noaudio --enable-autoexec -o output.test -x 1 -F JPEG -f 1`.
2. Stockfish 15 - an advanced open-source chess engine. This test is a stockfish built-in benchmark. A single hardware thread is used. URL: [https://stockfishchess.org](https://stockfishchess.org). Command line: `./stockfish bench 128 1 24 default depth`.
3. Clang 15 self-build - this test uses Clang 15 to build the Clang 15 compiler from sources. All hardware threads are used. URL: [https://www.llvm.org](https://www.llvm.org). Command line: `ninja -j16 clang`.
4. CloverLeaf 2018 - a Lagrangian-Eulerian hydrodynamics benchmark. All hardware threads are used. This test uses the clover_bm.in input file (Problem 5). URL: [http://uk-mac.github.io/CloverLeaf](http://uk-mac.github.io/CloverLeaf). Command line: `./clover_leaf`.

For this exercise, I ran all four benchmarks on the machine with the following characteristics:

* 12th Gen Alder Lake Intel&reg; Core&trade; i7-1260P CPU @ 2.10GHz (4.70GHz Turbo), 4P+8E cores, 18MB L3-cache
* 16 GB RAM, DDR4 @ 2400 MT/s
* 256GB NVMe PCIe M.2 SSD
* 64-bit Ubuntu 22.04.1 LTS (Jammy Jellyfish)
* Clang-15 C++ compiler with the following options: `-O3 -march=core-avx2`

To collect performance metrics, I used the `toplev.py` script from Andi Kleen's [pmu-tools](https://github.com/andikleen/pmu-tools):[^1]

```bash
$ ~/workspace/pmu-tools/toplev.py -m --global --no-desc -v -- <app with args>
```

Table {@tbl:perf_metrics_case_study} provides a side-by-side comparison of performance metrics for our four benchmarks. There is a lot we can learn about the nature of those workloads just by looking at the metrics. 

\small

--------------------------------------------------------------------------
Metric           Core        Blender     Stockfish   Clang15-   CloverLeaf
Name             Type                                selfbuild
---------------- ----------- ----------- ----------- ---------- ----------
Instructions     P-core      6.02E+12    6.59E+11    2.40E+13   1.06E+12

Core Cycles      P-core      4.31E+12    3.65E+11    3.78E+13   5.25E+12

IPC              P-core      1.40        1.80        0.64       0.20

CPI              P-core      0.72        0.55        1.57       4.96

Instructions     E-core      4.97E+12    0           1.43E+13   1.11E+12

Core Cycles      E-core      3.73E+12    0           3.19E+13   4.28E+12

IPC              E-core      1.33        0           0.45       0.26

CPI              E-core      0.75        0           2.23       3.85

L1MPKI           P-core      3.88        21.38       6.01       13.44

L2MPKI           P-core      0.15        1.67        1.09       3.58

L3MPKI           P-core      0.04        0.14        0.56       3.43

Br. Misp. Ratio  P-core      0.02        0.08        0.03       0.01

Code stlb MPKI   P-core      0           0.01        0.35       0.01

Ld stlb MPKI     P-core      0.08        0.04        0.51       0.03

St stlb MPKI     P-core      0           0.01        0.06       0.1

LdMissLat (Clk)  P-core      12.92       10.37       76.7       253.89

ILP              P-core      3.67        3.65        2.93       2.53

MLP              P-core      1.61        2.62        1.57       2.78

Dram Bw (GB/s)   All         1.58        1.42        10.67      24.57

IpCall           All         176.8       153.5       40.9       2,729

IpBranch         All         9.8         10.1        5.1        18.8

IpLoad           All         3.2         3.3         3.6        2.7

IpStore          All         7.2         7.7         5.9        22.0

IpMispredict     All         610.4       214.7       177.7      2,416

IpFLOP           All         1.1         1.82E+06    286,348    1.8

IpArith          All         4.5         7.96E+06    268,637    2.1

IpArith Scal SP  All         22.9        4.07E+09    280,583    2.60E+09

IpArith Scal DP  All         438.2       1.22E+07    4.65E+06   2.2

IpArith AVX128   All         6.9         0.0         1.09E+10   1.62E+09

IpArith AVX256   All         30.3        0.0         0.0        39.6

IpSWPF           All         90.2        2,565       105,933    172,348
--------------------------------------------------------------------------

Table: Performance Metrics of Four Benchmarks. {#tbl:perf_metrics_case_study}

\normalsize

Here are the hypotheses we can make about the performance of the benchmarks:

* __Blender__. The work is split fairly equally between P-cores and E-cores, with a decent IPC on both core types. The number of cache misses per kilo instructions is pretty low (see `L*MPKI`). Branch misprediction presents a minor bottleneck: the `Br. Misp. Ratio` metric is at `2%`; we get 1 misprediction for every `610` instructions (see `IpMispredict` metric), which is quite good. TLB is not a bottleneck as we very rarely miss in STLB. We ignore the `Load Miss Latency` metric since the number of cache misses is very low. The ILP is reasonably high. Golden Cove is a 6-wide architecture; an ILP of `3.67` means that the algorithm utilizes almost `2/3` of the core resources every cycle. Memory bandwidth demand is low (only 1.58 GB/s), far from the theoretical maximum for this machine. Looking at the `Ip*` metrics we can tell that Blender is a floating-point algorithm (see `IpFLOP` metric), a large portion of which is vectorized FP operations (see `IpArith AVX128`). But also, some portions of the algorithm are non-vectorized scalar FP single precision instructions (`IpArith Scal SP`). Also, notice that every 90th instruction is an explicit software memory prefetch (`IpSWPF`); we expect to see those hints in Blender's source code. Preliminary conclusion: Blender's performance is bound by FP compute.

* __Stockfish__. We ran it using only one hardware thread, so there is zero work on E-cores, as expected. The number of L1 misses is relatively high, but then most of them are contained in L2 and L3 caches. The branch misprediction ratio is high; we pay the misprediction penalty every `215` instructions. We can estimate that we get one mispredict every `215 (instructions) / 1.80 (IPC) = 120` cycles, which is very frequent. Similar to the Blender reasoning, we can say that TLB and DRAM bandwidth is not an issue for Stockfish. Going further, we see that there are almost no FP operations in the workload (see `IpFLOP` metric). Preliminary conclusion: Stockfish is an integer compute workload, which is heavily affected by branch mispredictions.

* __Clang 15 selfbuild__. Compilation of C++ code is one of the tasks that has a very flat performance profile, i.e., there are no big hotspots. You will see that the running time is attributed to many different functions. The first thing we spot is that P-cores are doing 68% more work than E-cores and have 42% better IPC. But both P- and E-cores have low IPC. The L*MPKI metrics don't look troubling at first glance; however, in combination with the load miss real latency (`LdMissLat`, in core clocks), we can see that the average cost of a cache miss is quite high (~77 cycles). Now, when we look at the `*STLB_MPKI` metrics, we notice substantial differences with any other benchmark we test. This is due to another aspect of the Clang compiler (and other compilers as well): the size of the binary is relatively big (more than 100 MB). The code constantly jumps to distant places causing high pressure on the TLB subsystem. As you can see the problem exists both for instructions (see `Code stlb MPKI`) and data (see `Ld stlb MPKI`). Let's proceed with our analysis. DRAM bandwidth use is higher than for the two previous benchmarks, but still is not reaching even half of the maximum memory bandwidth on our platform (which is ~34 GB/s). Another concern for us is the very small number of instructions per call (`IpCall`): only ~41 instructions per function call. This is unfortunately the nature of the compiler's codebase: it has thousands of small functions. The compiler needs to be more aggressive with inlining all those functions and wrappers.[^3] Yet, we suspect that the performance overhead associated with making a function call remains an issue for the Clang compiler. Also, one can spot the high `ipBranch` and `IpMispredict` metrics. For Clang compilation, every fifth instruction is a branch and one of every ~35 branches gets mispredicted. There are almost no FP or vector instructions, but this is not surprising. Preliminary conclusion: Clang has a large codebase, flat profile, many small functions, and "branchy" code; performance is affected by data cache misses and TLB misses, and branch mispredictions.

* __CloverLeaf__. As before, we start with analyzing instructions and core cycles. The amount of work done by P- and E-cores is roughly the same, but it takes P-cores more time to do this work, resulting in a lower IPC of one logical thread on P-core compared to one physical E-core.[^2] The `L*MPKI` metrics are high, especially the number of L3 misses per kilo instructions. The load miss latency (`LdMissLat`) is off the charts, suggesting an extremely high price of the average cache miss. Next, we take a look at the `DRAM BW use` metric and see that memory bandwidth consumption is near its limits. That's the problem: all the cores in the system share the same memory bus, so they compete for access to the main memory, which effectively stalls the execution. CPUs are undersupplied with the data that they demand. Going further, we can see that CloverLeaf does not suffer from mispredictions or function call overhead. The instruction mix is dominated by FP double-precision scalar operations with some parts of the code being vectorized. Preliminary conclusion: multi-threaded CloverLeaf is bound by memory bandwidth.

As you can see from this study, there is a lot one can learn about the behavior of a program just by looking at the metrics. It answers the "what?" question, but doesn't tell you the "why?". For that, you will need to collect a performance profile, which we will introduce in later chapters. In Part 2 of this book, we will discuss how to mitigate the performance issues we suspect to exist in the four benchmarks that we have analyzed.

Keep in mind that the summary of performance metrics in Table {@tbl:perf_metrics_case_study} only tells you about the *average* behavior of a program. For example, we might be looking at CloverLeaf's IPC of `0.2`, while in reality, it may never run with such an IPC. Instead, it may have 2 phases of equal duration, one running with an IPC of `0.1`, and the second with an IPC of `0.3`. Performance tools tackle this by reporting statistical data for each metric along with the average value. Usually, having min, max, 95th percentile, and variation (stdev/avg) is enough to understand the distribution. Also, some tools allow plotting the data, so you can see how the value for a certain metric changed during the program running time. As an example, Figure @fig:CloverMetricCharts shows the dynamics of IPC, L*MPKI, DRAM BW, and average frequency for the CloverLeaf benchmark. The `pmu-tools` package can automatically build those charts once you add the `--xlsx` and `--xchart` options. The `-I 10000` option aggregates collected samples with 10-second intervals.

```bash
$ ~/workspace/pmu-tools/toplev.py -m --global --no-desc -v --xlsx workload.xlsx –xchart -I 10000 -- ./clover_leaf
```

![Performance metrics charts for the CloverLeaf benchmark with 10 second intervals.](../../img/terms-and-metrics/CloverMetricCharts2.png){#fig:CloverMetricCharts width=98% }

Even though the deviation from the average values reported in the summary is not very big, we can see that the workload is not stable. After looking at the IPC chart for P-core we can hypothesize that there are no distinct phases in the workload and the variation is caused by multiplexing between performance events (discussed in [@sec:counting]). Yet, this is only a hypothesis that needs to be confirmed or disproved. Possible ways to proceed would be to collect more data points by running collection with higher granularity (in our case it was 10 seconds). The chart that plots L*MPKI suggests that all three metrics hover around their average numbers without much deviation. The DRAM bandwidth utilization chart indicates that there are periods with varying pressure on the main memory. The last chart shows the average frequency of all CPU cores. As you may observe on this chart, throttling starts after the first 10 seconds. I recommend being careful when drawing conclusions just from looking at the aggregate numbers since they may not be a good representation of the workload behavior.

Remember that collecting performance metrics is not a substitute for looking into the code. Always try to find explanation for the numbers that you see by checking relevant parts of the code.

In summary, performance metrics help you build the right mental model about what is and what is *not* happening in a program. Going further into analysis, these data will serve you well.

[^1]: pmu-tools - [https://github.com/andikleen/pmu-tools](https://github.com/andikleen/pmu-tools)
[^2]: A possible explanation for that is because CloverLeaf is very memory-bandwidth bound. All P- and E-cores are equally stalled waiting on memory. Because P-cores have a higher frequency, they waste more CPU clocks than E-cores.
[^3]: Perhaps by using Link Time Optimizations (LTO).
