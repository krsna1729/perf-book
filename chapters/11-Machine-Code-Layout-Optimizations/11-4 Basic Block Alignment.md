

## Basic Block Alignment

Sometimes performance can significantly change depending on the offset at which instructions are laid out in memory. Consider a simple function presented in [@lst:LoopAlignment] along with the corresponding machine code when compiled with `-O3 -march=core-avx2 -fno-unroll-loops` (loop unrolling is disabled to illustrate the idea).

Listing: Basic block alignment

~~~~ {#lst:LoopAlignment .cpp}
void benchmark_func(int* a) {    │ 00000000004046a0 <_Z14benchmark_funcPi>:
  for (int i = 0; i < 32; ++i)   │ 4046a0: mov rax,0xffffffffffffff80
    a[i] += 1;                   │ 4046a7: vpcmpeqd ymm0,ymm0,ymm0
}                                │ 4046ab: nop DWORD [rax+rax+0x0]
                                 │ 4046b0: vmovdqu ymm1,[rdi+rax+0x80] # loop begins
                                 │ 4046b9: vpsubd ymm1,ymm1,ymm0
                                 │ 4046bd: vmovdqu [rdi+rax+0x80],ymm1
                                 │ 4046c6: add rax,0x20
                                 │ 4046ca: jne 4046b0                  # loop ends
                                 │ 4046cc: vzeroupper 
                                 │ 4046cf: ret 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The code itself is reasonable, but its layout is not perfect (see Figure @fig:Loop_default). Instructions that correspond to the loop are highlighted in yellow. Thick boxes denote cache line borders. Cache lines are 64 bytes long. 

<div id="fig:LoopLayout">
![default layout](../../img/cpu_fe_opts/LoopAlignment_Default.png){#fig:Loop_default width=100%}

![improved layout](../../img/cpu_fe_opts/LoopAlignment_Better.png){#fig:Loop_better width=100%}

Two different code layouts for the loop in [@lst:LoopAlignment].
</div>

Notice that the loop spans multiple cache lines: it begins on the cache line `0x80-0xBF` and ends in the cache line `0xC0-0xFF`. To fetch instructions that are executed in the loop, a processor needs to read two cache lines. These kinds of situations sometimes cause performance problems for the CPU Frontend, especially for the small loops like those presented in [@lst:LoopAlignment].

To fix this, we can shift the loop instructions forward by 16 bytes using a single NOP instruction so that the whole loop will reside in one cache line. Figure @fig:Loop_better shows the effect of doing this with the NOP instruction highlighted in blue. 

Interestingly, the performance impact is visible even if you run nothing but this hot loop in a microbenchmark. It is somewhat puzzling since the amount of code is tiny and it shouldn't saturate the L1 I-cache size on any modern CPU. The reason for the better performance of the layout in Figure @fig:Loop_better is not trivial to explain and will involve a fair amount of microarchitectural details, which we don't discuss in this book. Interested readers can find more information in the related article on the Easyperf blog.[^1]

By default, the LLVM compiler recognizes loops and aligns them at 16B boundaries, as we saw in Figure @fig:Loop_default. To reach the desired code placement for our example, as shown in Figure @fig:Loop_better, you can use the `-mllvm -align-all-blocks=5` option that will align every basic block in an object file at a 32 bytes boundary. However, I do not recommend using this and similar options as they affect the code layout of all the functions in the translation unit. There are other less intrusive options.

A recent addition to the LLVM compiler is the new `[[clang::code_align()]]` loop attribute, which allows developers to specify the alignment of a loop in the source code. This gives a very fine-grained control over machine code layout. The following code shows how the new Clang attribute can be used to align a loop at a 64 bytes boundary: 

```cpp
void benchmark_func(int* a) {
  [[clang::code_align(64)]]
  for (int i = 0; i < 32; ++i)
    a[i] += 1;
}
```

Before this attribute was introduced, developers had to resort to some less practical solutions like injecting `asm(".align 64;")` statements of inline assembly in the source code.

Even though CPU architects work hard to minimize the impact of machine code layout, there are still cases when code placement (alignment) can make a difference in performance. Machine code layout is also one of the main sources of noise in performance measurements. It makes it harder to distinguish a real performance improvement or regression from the accidental one, that was caused by the change in the code layout.

[^1]: "Code alignment issues" - [https://easyperf.net/blog/2018/01/18/Code_alignment_issues](https://easyperf.net/blog/2018/01/18/Code_alignment_issues)
