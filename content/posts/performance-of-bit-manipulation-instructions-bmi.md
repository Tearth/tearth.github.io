---
title: "Performance of Bit Manipulation Instructions (BMI)"
date: "2024-05-08"
description: "Small discussion about Bit Manipulation Instructions (BMI) set, with a comparison of assembly code and performance between typical computations and intrinsic functions."
categories:
  - "Performance comparisons"
tags:
  - ".NET"
  - ".NET Core"
  - "Rust"
  - "Assembly"
  - "BMI"
sidebar: "right"
---

**Bit Manipulation Instructions** (BMI) is an interesting extension for the x86-64 architecture, introduced by Intel in Haswell processors (early 2010s). Its main purpose is, as the name suggests, increasing the speed of the most common bit operations by replacing manual calculation with dedicated instructions (which means hardware support). This article will focus on the performance of three example instructions: [BLSI](https://www.felixcloutier.com/x86/BLSI) (reads the lowest bit), [BLSR](https://www.felixcloutier.com/x86/BLSR) (resets the lowest bit) and [TZCNT](https://www.felixcloutier.com/x86/TZCNT) (counts the number of trailing non-set bits).

<!--more-->

## Theory

The first time I met with Bit Manipulation Instructions was while developing a chess engine [Proxima b 2.0](https://github.com/Tearth/Proxima-b-2.0) for my thesis. It is important to know, that nearly every modern engine represents the chessboard in memory as [bitboards](https://www.chessprogramming.org/Bitboards) - a collection of unsigned 64-bit variables, where every bit represents one field. Thanks to it, a lot of things (like generating moves or checking which pieces are currently attacked) can be done fast using bit operations. The correct implementation is crucial for whole engine performance, and that's where hardware support for more complex expressions can be really useful. BMI isn't one solid pack of instructions -  there are two main sets (which are identified separately by [CPUID](https://www.felixcloutier.com/x86/CPUID) instruction):
 - **BMI1** contains 6 new instructions to resetting, extracting, counting and comparing
 
| Instruction name                                   | Description                                                    |
|----------------------------------------------------|----------------------------------------------------------------|
| [ANDN](https://www.felixcloutier.com/x86/ANDN)     | Performs logical AND with negated X and Y (`~x & y`)           |
| [BEXTR](https://www.felixcloutier.com/x86/BEXTR)   | Reads n bits starting from the specified index                 |
| [BLSI](https://www.felixcloutier.com/x86/BLSI)     | Reads the lowest bit (`x & ~x`)                                |
| [BLSMSK](https://www.felixcloutier.com/x86/BLSMSK) | Creates mask for all bits up to the lowest bit (`x ^ (x - 1)`) |
| [BLSR](https://www.felixcloutier.com/x86/BLSR)     | Resets the lowest bit (`x & (x - 1)`)                            |
| [TZCNT](https://www.felixcloutier.com/x86/TZCNT)   | Calculates the number of trailing non-set bits                 |

 - **BMI2** contains 8 new instructions mainly for shifting, rotating and parallel operations

| Instruction name                                         | Description                                              |
|----------------------------------------------------------|----------------------------------------------------------|
| [BZHI](https://www.felixcloutier.com/x86/BZHI)           | Zeros all bits from the specified index                  |
| [MULX](https://www.felixcloutier.com/x86/MULX)           | Performs unsigned multiplication without affecting flags |
| [PDEP](https://www.felixcloutier.com/x86/PDEP)           | Stores bits using the mask                               |
| [PEXT](https://www.felixcloutier.com/x86/PEXT)           | Extracts bits using the mask                             |
| [RORX](https://www.felixcloutier.com/x86/RORX)           | Rotates logical right without affecting flags            |
| [SARX](https://www.felixcloutier.com/x86/SARX:SHLX:SHRX) | Performs arithmetic right shift without affecting flags  |
| [SHRX](https://www.felixcloutier.com/x86/SARX:SHLX:SHRX) | Performs logical right shift without affecting flags     |
| [SHLC](https://www.felixcloutier.com/x86/SARX:SHLX:SHRX) | Performs logical left shift without affecting flags      |

What's more, AMD created its own sets:
 - **ABM** (Advanced Bit Manipulation) with two instructions: [POPCNT](https://www.felixcloutier.com/x86/POPCNT) (calculates the number of set bits) and [LZCNT](https://www.felixcloutier.com/x86/LZCNT) (calculates the number of leading non-set bits).
 - **TBM** (Trailing Bit Manipulation) - a set of 10 instructions whose goal was to refill the original BMI set. You can read more about them in the [AMD documentation](https://www.amd.com/system/files/TechDocs/24594.pdf).

As you can see, there are many ways to operate on bits using hardware support. In the next chapter, we will focus on the performance of three instructions: [BLSI](https://www.felixcloutier.com/x86/BLSI), [BLSR](https://www.felixcloutier.com/x86/BLSR), [TZCNT](https://www.felixcloutier.com/x86/TZCNT). First, we will write a program that performs operations using traditional mathematical formulas and then compares their performance with BMI instructions described above.

## Implementation

Each test will be implemented in Rust and benchmarked by [criterion.rs](https://github.com/bheisler/criterion.rs) framework.

#### BLSI

At first, we will test [BLSI](https://www.felixcloutier.com/x86/BLSI) instruction, which reads the lowest set bit and returns it. The equivalent formula for this operation is `x & -x` - ANDing variable with its negation, which works thanks to U2 (two's complement) system, where negating a number is just an inversion of all bits, and then adding 1. Let's see at the example for 8-bit number 108:

```
    x = 01101100
   -x = 10010100

        01101100
    AND 10010100
    ------------
        00000100
```

Having this knowledge, let's see at the example implementation in Rust.

{{< highlight rust "linenos=table" >}}
use criterion::black_box;
use criterion::criterion_group;
use criterion::criterion_main;
use criterion::Criterion;

fn benchmark(criterion: &mut Criterion) {
    criterion.bench_function("get_lsb", |bencher| bencher.iter(|| get_lsb(black_box(45631345))));
}

fn get_lsb(x: u64) -> u64 {
    x & x.wrapping_neg()
}

criterion_group!(benches, benchmark);
criterion_main!(benches);
{{< / highlight >}}

Assembly code generated by the compiler is straightforward - we store `x` in `rax` register, negate it and perform logical AND with `x`.

{{< highlight nasm "linenos=table" >}}
mov rax, rcx
neg rax
and rax, rcx
ret
{{< / highlight >}}

Now let's check what we will see after using [BLSI](https://www.felixcloutier.com/x86/BLSI) instruction. To enable BMI1 instruction set we have to set environmental variable RUSTFLAGS (`RUSTFLAGS = "-C target-feature=+bmi1`) and recompile project - Rust compiler should detect and optimize the formula to `blsi` without any additional action.

Now the assembly code. As expected, a few instructions has been replaced by single `blsi` call with `rcx` as the source, and `rax` as the destination register.

{{< highlight nasm "linenos=table" >}}
blsi rax, rcx
ret
{{< / highlight >}}

Benchmark results:

|        | Lower bound | Estimate      | Upper bound |
|--------|:------------|:--------------|:------------|
| Manual | 356.28 ps   | **356.69 ps** | 357.16 ps   |
| BMI1   | 356.22 ps   | **356.97 ps** | 357.87 ps   |

#### BLSR

[BLSR](https://www.felixcloutier.com/x86/BLSR) instruction resets the lowest set bit and returns the new value. The equivalent formula for this operation is `x & (x - 1)` - ANDing variable with its copy decreased by 1. Let's see at the example for 8-bit number 108 (similar as in previous chapter):

```
    x = 01101100
x - 1 = 01101011

        01101100
    AND 01101011
    ------------
        01101000
```

{{< highlight rust "linenos=table" >}}
use criterion::black_box;
use criterion::criterion_group;
use criterion::criterion_main;
use criterion::Criterion;

fn benchmark(criterion: &mut Criterion) {
    criterion.bench_function("pop_lsb", |bencher| bencher.iter(|| get_lsb(black_box(45631345))));
}

fn pop_lsb(x: u64) -> u64 {
    x & (x - 1)
}

criterion_group!(benches, benchmark);
criterion_main!(benches);
{{< / highlight >}}

Assembly code generated by the compiler is also pretty simple - `lea` instruction decrements `rcx` register (where the iterator is stored), and stores result in the `rax` register. Both `rax` and `rcx` are then ANDed and the result is stored to the `rax` register. 

{{< highlight nasm "linenos=table" >}}
lea rax, [rcx - 1]
and rax, rcx
ret
{{< / highlight >}}

With BMI1 enabled, both instructions are merged into single `blsi` with the result stored in the `rax` register.

{{< highlight nasm "linenos=table" >}}
blsi rax, rcx
ret
{{< / highlight >}}

Benchmark results:

|        | Lower bound | Estimate      | Upper bound |
|--------|:------------|:--------------|:------------|
| Manual | 356.12 ps   | **357.31 ps** | 359.10 ps   |
| BMI1   | 356.65 ps   | **357.58 ps** | 358.92 ps   |

#### TZCNT

The last tested instruction, [TZCNT](https://www.felixcloutier.com/x86/TZCNT), counts trailing non-set bits. The algorithm is based on [De Bruijn multiplication](https://www.chessprogramming.org/BitScan#De_Bruijn_Multiplication) which is considered as one of the fastest alternative to `tzcnt`.

{{< highlight rust "linenos=table" >}}
use criterion::black_box;
use criterion::criterion_group;
use criterion::criterion_main;
use criterion::Criterion;

fn benchmark(criterion: &mut Criterion) {
    criterion.bench_function("bit_scan", |bencher| bencher.iter(|| bit_scan(black_box(45631345))));
}

fn bit_scan(x: u64) -> u64 {
    #[rustfmt::skip]
    const INDEX64: [u32; 64] = [
        0,  1,  48,  2, 57, 49, 28,  3,
        61, 58, 50, 42, 38, 29, 17,  4,
        62, 55, 59, 36, 53, 51, 43, 22,
        45, 39, 33, 30, 24, 18, 12,  5,
        63, 47, 56, 27, 60, 41, 37, 16,
        54, 35, 52, 21, 44, 32, 23, 11,
        46, 26, 40, 15, 34, 20, 31, 10,
        25, 14, 19,  9, 13,  8,  7,  6
    ];

    INDEX64[(((x & x.wrapping_neg()).wrapping_mul(0x03f79d71b4cb0a89)) >> 58) as usize]
}

criterion_group!(benches, benchmark);
criterion_main!(benches);
{{< / highlight >}}

The output assembly is a bit more complex, but the general idea is the same as for previous examples. You can say only by looking at this, that there are a lot of instructions, which will for sure consume processor time during computation.

{{< highlight nasm "linenos=table" >}}
mov rax, rcx
neg rax
and rax, rcx
movabs rcx, 285870213051386505
imul rcx, rax
shr rcx, 58
lea rax, [rip + anon.9e0e815cd4fdb58dcb37887dde8870f8.1020]
mov eax, dword ptr [rax + 4*rcx]
ret
{{< / highlight >}}

Now let's look at the intrinsic function. The BMI1 equivalent of the manual counting zeros can be called by `trailing_zeros()`.

{{< highlight rust "linenos=table" >}}
use criterion::black_box;
use criterion::criterion_group;
use criterion::criterion_main;
use criterion::Criterion;

fn benchmark(criterion: &mut Criterion) {
    criterion.bench_function("bit_scan", |bencher| bencher.iter(|| bit_scan(black_box(45631345))));
}

fn bit_scan(x: u64) -> u64 {
    x.trailing_zeros()
}

criterion_group!(benches, benchmark);
criterion_main!(benches);
{{< / highlight >}}

{{< highlight nasm "linenos=table" >}}
tzcnt rax, rcx
ret
{{< / highlight >}}

Benchmark results:

|        | Lower bound | Estimate      | Upper bound |
|--------|:------------|:--------------|:------------|
| Manual | 532.50 ps   | **533.43 ps** | 534.23 ps   |
| BMI1   | 330.40 ps   | **331.52 ps** | 332.93 ps   |

## Comparison

All benchmark results have been compiled in the following table. It clearly suggests that using `blsi` and `blsr` instructions didn't bring any noticeable performance gain - manual calculation was fast enough. Using `tzcnt` was much more successful since De Bruijn multiplication involved much more assembly that could be replaced by just one instruction.

|        | Estimate (Manual) | Estimate (BMI1) | Difference |
|--------|:------------------|:----------------|:-----------|
| BLSI   | 356.69            | 356.97          | **0,0%**   |
| BLSR   | 357.31 ps         | 357.58 ps       | **0,0%**   |
| TZCNT  | 533.43 ps         | 331.52 ps       | **62,1%**  |

## A few words about C#

Until recently, all responsibility for generating machine code was on the JIT (Just-in-time). It means you couldn't tell the compiler that you want to use some specific instruction, eg. [BLSI](https://www.felixcloutier.com/x86/BLSI). But according to the [recent news](https://devblogs.microsoft.com/dotnet/hardware-intrinsics-in-net-core/), .NET Core 3 adds an amazing feature that allows using intrinsic functions directly in the C# code. You can find them in the [System.Runtime.Intrinsics.X86](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86?view=netcore-3.1) namespace - don't be suggested by the "X86" prefix, there are also versions for 64-bit functions but for some reason, they are inside X86 namespace.

 - **BLSI** - [System.Runtime.Intrinsics.X86.Bmi1.X64.ExtractLowestSetBit](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.bmi1.x64.extractlowestsetbit?view=netcore-3.1)
 - **BLSR** - [System.Runtime.Intrinsics.X86.Bmi1.X64.ResetLowestSetBit](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.bmi1.x64.resetlowestsetbit?view=netcore-3.1)
 - **TZCNT** - [System.Numerics.BitOperations.TrailingZeroCount](https://docs.microsoft.com/en-us/dotnet/api/system.numerics.bitoperations.trailingzerocount?view=netcore-3.1)

I made a good use of them while developing [Cosette](https://github.com/Tearth/Cosette), example can be found on the project repository in [BitOperations.cs](https://github.com/Tearth/Cosette/blob/master/Cosette/Engine/Common/BitOperations.cs). Calls to `X.IsSupported` do not have any additional overhead, since JIT correctly understands the platform on which a program runs on, so the unnecessary parts of the functions are cut off.