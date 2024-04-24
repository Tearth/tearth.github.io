---
title: "Performance of Bit Manipulation Instructions (BMI)"
date: "2020-06-01"
description: "Small discussion about Bit Manipulation Instructions (BMI) set, with a comparison of assembly code and performance between typical computations and intrinsic functions."
categories:
  - "Performance comparisons"
tags:
  - ".NET"
  - ".NET Core"
  - "C++"
  - "Assembly"
  - "BMI"
sidebar: "right"
---

**Bit Manipulation Instructions** (BMI) is an interesting extension for the x86-64 architecture, introduced by Intel in Haswell processors (early 2010s). Its main purpose is, as the name suggests, increasing the speed of the most common bit operations by replacing manual calculation with dedicated instructions (which means hardware support). This article will focus on the performance of three example instructions: [BLSI](https://www.felixcloutier.com/x86/BLSI) (reads the lowest bit), [BLSR](https://www.felixcloutier.com/x86/BLSR) (resets the lowest bit) and [TZCNT](https://www.felixcloutier.com/x86/TZCNT) (counts the number of trailing non-set bits).

<!--more-->

## Theory

The first time I met with Bit Manipulation Instructions was during developing a chess engine [Proxima b 2.0](https://github.com/Tearth/Proxima-b-2.0) for my thesis. It is important to know, that nearly every modern engine represents the chessboard in memory as [bitboards](https://www.chessprogramming.org/Bitboards) - a collection of unsigned 64-bit variables, where every bit represents one field. Thanks to it, a lot of things (like generating moves or checking which pieces are currently attacked) can be done really fast using bit operations. The correct implementation is crucial for whole engine performance, and that's where hardware support for more complex expressions can be really useful. BMI isn't one solid pack of instructions -  there are two main sets (which are identified separately by [CPUID](https://www.felixcloutier.com/x86/CPUID) instruction):
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

As you can see, there are many ways to operate on bits using hardware support. In the next chapter, we will focus on performance of three instructions: [BLSI](https://www.felixcloutier.com/x86/BLSI), [BLSR](https://www.felixcloutier.com/x86/BLSR), [TZCNT](https://www.felixcloutier.com/x86/TZCNT). First, we will write a program that performs operations using traditional mathematical formulas and then compares their performance with BMI instructions described above.

## Tests

Let's start with a generic template in C++, that will be used to perform tests. I've decided to use [chrono library](https://en.cppreference.com/w/cpp/chrono) which is amazing if you need accurate time measuring. There are also two lines from WinAPI ([SetPriorityClass](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setpriorityclass) and [SetThreadPriority](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadpriority)) which set a priority of the process and thread to the highest possible (except realtime) - Linux users have to use [setpriority](https://linux.die.net/man/3/setpriority) function. I think that the rest of the code is quite self-explaining, so let's go further.

{{< highlight cpp "linenos=table" >}}
#include <iostream>
#include <chrono>
#include <Windows.h>

int IterationsPerTest = 10;
unsigned long long SamplesCount = 4000000000;

int main()
{
    // Set priorities
    SetPriorityClass(GetCurrentProcess(), HIGH_PRIORITY_CLASS);
    SetThreadPriority(GetCurrentThread(), THREAD_MODE_BACKGROUND_BEGIN);

    // Test
    for (int i = 0; i < IterationsPerTest; i++)
    {
        auto start = steady_clock::now();
        // ...
        // Insert test function here
        // ...
        auto end = steady_clock::now();

        auto elapsed = end - start;
        auto time = duration_cast<milliseconds>(elapsed).count();
        cout << "Iteration " << i << ": " << time << " ms" << endl;
    }
}
{{< / highlight >}}

Now, we will write a few functions and look at the assembly code generated by the compiler.

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

Having this knowledge, let's see at the example implementation in C++. It's a simple for-loop which reads the lowest set bit from the iterator and then saves it to the variable. It's incredibly important to mark Result as volatile - without it, the compiler sees that all results except the last one will be overridden and it's high chance that it will optimize this to a single operation instead of a loop.

{{< highlight cpp "linenos=table" >}}
volatile unsigned long long Result;

void TestGetLsb()
{
    for (unsigned long long i = 0; i < SamplesCount; i++)
    {
        Result = (unsigned long long)((long long)i & -(long long)i);
    }
}
{{< / highlight >}}

Now let's check assembly code generated by the compiler for the loop. An optimization done here is a bit tricky, because there is not any negation performed here, what would be expected when we do `-(long long)i` operation. Instead of it, the compiler did it only once (for the initial value of `i`, so 0) before the loop and assigned a result to the `rdx` register. Every iteration, this register is decrementing which gives the same effect as for negating iterator and adding 1 every time. The rest of the assembly is quite straightforward - we can see `and` instruction with `rcx` (value of `rdx` just before decrementation) and `rax` (iterator) arguments. Then, the iterator is incremented and our result is saved to the variable. Last `cmp` and `jb` instructions just check if the loop should iterate more or not.

{{< highlight nasm "linenos=table" >}}
mov rcx,rdx
dec rdx
and rcx,rax
inc rax
mov qword ptr ds:[<unsigned __int64 volatile Result>],rcx
cmp rax,rbp
jb cpptest.7FF64C3B1080
{{< / highlight >}}

We know how does assembly code looks like for using formula directly, so let's check what we will see after using [BLSI](https://www.felixcloutier.com/x86/BLSI) instruction. We have two ways to do it: first, we can change compiler properties to use AVX2 instruction set (Project -> Properties -> C/C++ -> Code Generation -> Enable Enhanced Instruction Set). Because this option includes also BMI sets, then it's a high chance that the compiler will detect a pattern and use specialized instruction directly (and results of my experiments confirm that it does indeed).

{{< image
src="/img/1/instruction_set_properties.jpg"
alt="Values available for Enable Enchanced Instruction Set option"
caption="Values available for Enable Enchanced Instruction Set option" >}}

The second option is to use [intrinsic functions](https://docs.microsoft.com/en-us/cpp/intrinsics/x64-amd64-intrinsics-list?view=vs-2019), which are the explicit way to tell the compiler that we want to use the specified instruction. For our experiments, we will use the second one. [BLSI](https://www.felixcloutier.com/x86/BLSI) instruction is supported by `_blsi_u64` functions, which takes a number to process as the parameter, and returns the lowest set bit.

{{< highlight cpp "linenos=table" >}}
void TestGetLsb()
{
    for (unsigned long long i = 0; i < SamplesCount; i++)
    {
        Result = _blsi_u64(i);
    }
}
{{< / highlight >}}

Now the assembly code. As expected, a few instructions has been replaced by single `blsi` call with `rax` as the source, and `rcx` as destination register - so no more MOVing, ANDing and DECrementing. The rest of the code related to loop control remained the same.

{{< highlight nasm "linenos=table" >}}
blsi rcx,rax
inc rax
mov qword ptr ds:[<unsigned __int64 volatile Result>],rcx
cmp rax,rbp
jb cpptest.7FF71FDD1080
{{< / highlight >}}

One of the biggest concerns during performing a benchmark was overhead generated by the instructions related to loop (like controlling iterator or jumping). To minimize its effect, I did an additional test run called "Zero", which contains the loop with the single assign of 0 to `Result` variable inside. Of course, it's far from ideal, but at least shows more or less the time needed by for-loop to do the most basic operation.

Tests were run for 4,000,000,000 samples in 10 independent iterations. As we can see at the table below, "Zero" pass takes about 1053 ms, so we can assume that it's the time necessary for loop and variable assignment. The intrinsic function has a very similar result, which differs by several dozen milliseconds. As expected, the longest run was during executing the "Manual" pass, where all calculations were performed using direct formula. Subtracting time got from "Zero" pass, we can assume that direct formula performs about 500 ms longer than intrinsic function.

|         | Zero [ms]   | Manual [ms]      | Intrinsic [ms]      |
|---------|:------------|:-----------------|:--------------------|
| 0       | 1112        | 1692             | 1089                |
| 1       | 1077        | 1715             | 1078                |
| 2       | 1039        | 1634             | 1109                |
| 3       | 1042        | 1646             | 1083                |
| 4       | 1041        | 1612             | 1064                |
| 5       | 1050        | 1568             | 1080                |
| 6       | 1038        | 1558             | 1101                |
| 7       | 1040        | 1554             | 1076                |
| 8       | 1049        | 1568             | 1066                |
| 9       | 1042        | 1560             | 1062                |
| **Avg** | **1053**    | **1611**         | **1081**            |

#### BLSR

[BLSR](https://www.felixcloutier.com/x86/BLSR) instruction resets the lowest set bit and returns the new value. Equivalent formula for this operations is `x & (x - 1)` - ANDing variable with its copy decreased by 1. Let's see at the example for 8-bit number 108 (similar as in previous chapter):

```
    x = 01101100
x - 1 = 01101011

        01101100
    AND 01101011
    ------------
        01101000
```

{{< highlight cpp "linenos=table" >}}
volatile unsigned long long Result;

void TestPopLsb()
{
    for (unsigned long long i = 0; i < SamplesCount; i++)
    {
        Result = i & (i - 1);
    }
}
{{< / highlight >}}

Assembly code generated by the compiler is pretty simple - `lea` instruction decrements `rax` register (where the iterator is stored), and stores result in the `rcx` register. Both `rax` and `rcx` are then ANDed and the result is stored to the `rcx` register. Next, `rax` (iterator) is incremented, the result of the `and` is stored to the `Result` variable and at the end, there is `cmp` and `jb` which are responsible for proper loop work.

{{< highlight nasm "linenos=table" >}}
lea rcx,qword ptr ds:[rax-1]
and rcx,rax
inc rax
mov qword ptr ds:[<unsigned __int64 volatile Result>],rcx
cmp rax,rbp
jb cpptest.7FF6ABAF1076
{{< / highlight >}}

The implementation using the intrinsic function is very familiar with the one from the previous chapter. Here we use `_blsr_u64` function, which does the same operation as in the previous code.

{{< highlight cpp "linenos=table" >}}
#include <immintrin.h>
volatile unsigned long long Result;

void TestPopLsb()
{
    for (unsigned long long i = 0; i < SamplesCount; i++)
    {
        Result = _blsr_u64(i);
    }
}
{{< / highlight >}}

The assembly output for the intrinsic function is nearly the same as when using `_blsi_u64` - the only difference is the used instruction `blsr` instead of `blsi`.

{{< highlight nasm "linenos=table" >}}
blsr rcx,rax
inc rax
mov qword ptr ds:[<unsigned __int64 volatile Result>],rcx
cmp rax,rbp
jb cpptest.7FF79B791080
{{< / highlight >}}

|         | Zero [ms]     | Manual [ms]      | Intrinsic [ms]      |
|---------|:--------------|:-----------------|:--------------------|
| 0       | 1083          | 2314             | 1101                |
| 1       | 1045          | 2216             | 1073                |
| 2       | 1056          | 2178             | 1075                |
| 3       | 1058          | 2098             | 1071                |
| 4       | 1061          | 2093             | 1073                |
| 5       | 1060          | 2097             | 1081                |
| 6       | 1057          | 2088             | 1085                |
| 7       | 1069          | 2092             | 1074                |
| 8       | 1041          | 2084             | 1063                |
| 9       | 1052          | 2091             | 1068                |
| **Avg** | **1058**      | **2135**         | **1076**            |

#### TZCNT

The last tested instruction, [TZCNT](https://www.felixcloutier.com/x86/TZCNT), counts trailing non-set bits. The algorithm is very simple: increment counter until the first bit in the source variable is not set, and shift it every iteration. If the first bit is set, or the counter is bigger than 64 (the size of type) then stop the whole loop.

{{< highlight cpp "linenos=table" >}}
volatile unsigned long long Result;

void TestTrailingZeros()
{
    for (unsigned long long i = 0; i < SamplesCount; i++)
    {
        unsigned long long x = i;
        int count = 0;

        while ((x & 1) == 0 && count < 64)
        {
            x >>= 1;
            count++;
        }

        Result = (unsigned long long)count;
    }
}
{{< / highlight >}}

The output assembly is a bit more complex, but the general idea is the same as for previous examples. You can say only by looking at this, that there are a lot of instructions, which will for sure consume processor time during computation.

{{< highlight nasm "linenos=table" >}}
xor eax,eax
mov rcx,rdx
test dl,1
jne cpptest.7FF760DC108F
cmp eax,40
jge cpptest.7FF760DC108F
shr rcx,1
inc eax
test cl,1
je cpptest.7FF760DC1080
inc rdx
movsxd rcx,eax
mov qword ptr ds:[<unsigned __int64 volatile Result>],rcx
cmp rdx,rbp
jb cpptest.7FF760DC1076
{{< / highlight >}}

Now let's see at the intrinsic function. The equivalent of the manual counting zeros is `_tzcnt_u64`, which takes some number or variable, and then returns the number of trailing zeros.

{{< highlight cpp "linenos=table" >}}
#include <immintrin.h>
volatile unsigned long long Result;

void TestTrailingZeros()
{
    for (unsigned long long i = 0; i < SamplesCount; i++)
    {
        Result = _tzcnt_u64(i);
    }
}
{{< / highlight >}}

The output assembly code is without surprises - just `tzcnt` call with some instruction for a loop.

{{< highlight nasm "linenos=table" >}}
tzcnt rcx,rax
inc rax
mov qword ptr ds:[<unsigned __int64 volatile Result>],rcx
cmp rax,rbp
jb cpptest.7FF6AE1A1080
{{< / highlight >}}

Tests indicate, that difference between manual computation and using the intrinsic function is really significant. 

|         | Zero [ms]   | Manual [ms]      | Intrinsic [ms]      |
|---------|:------------|:-----------------|:--------------------|
| 0       | 1068        | 5020             | 1133                |
| 1       | 1042        | 4868             | 1090                |
| 2       | 1056        | 4674             | 1076                |
| 3       | 1044        | 4676             | 1048                |
| 4       | 1064        | 4660             | 1093                |
| 5       | 1056        | 4665             | 1143                |
| 6       | 1049        | 4660             | 1059                |
| 7       | 1040        | 4673             | 1052                |
| 8       | 1084        | 4655             | 1045                |
| 9       | 1077        | 4655             | 1051                |
| **Avg** | **1058**    | **4720**         | **1079**            |

## Comparison

Let's piece together our results. Values in columns "Zero", "Manual" and "Intrinsic" are taken directly from the Avg row in all previous tables. Column "Gain" is calculated by the formula `((Manual - Zero) / (Intrinsic - Zero)) * 100%` which theoretically should give us information about how much better is the intrinsic function over manual calculating. 

|                                                  | Zero [ms]   | Manual [ms]      | Intrinsic [ms]      | Gain [%] |
|--------------------------------------------------|:------------|:-----------------|:--------------------|:---------|
| [BLSI](https://www.felixcloutier.com/x86/BLSI)   | 1053        | 1611             | 1081                | 2006     |
| [BLSR](https://www.felixcloutier.com/x86/BLSR)   | 1058        | 2135             | 1076                | 5917     |
| [TZCNT](https://www.felixcloutier.com/x86/TZCNT) | 1058        | 4721             | 1079                | 17441    |

The final results are as follows: [BLSI](https://www.felixcloutier.com/x86/BLSI) instruction is about 20 times faster than `i & (i - 1)`, [BLSR](https://www.felixcloutier.com/x86/BLSR) is about 60 times faster than `i & (i - 1)`, and [TZCNT](https://www.felixcloutier.com/x86/TZCNT) is (with absolute record) about 175 times faster than manual computing based on the loop and shifting.

## A few words about C#

Until recently, all responsibility for generating machine code was on the JIT (Just-in-time). It means you couldn't tell the compiler that you want to use some specific instruction, eg. [BLSI](https://www.felixcloutier.com/x86/BLSI). But according to the [recent news](https://devblogs.microsoft.com/dotnet/hardware-intrinsics-in-net-core/), .NET Core 3 adds an amazing feature which allows using intrinsic functions directly in the C# code. You can find them in the [System.Runtime.Intrinsics.X86](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86?view=netcore-3.1) namespace - don't be suggested by the "X86" prefix, there are also versions for 64-bit functions but for some reason, they are inside X86 namespace.

Equivalents of functions in C# for intrinsic functions used in the previous chapters:
 - **_blsi_u64** - [System.Runtime.Intrinsics.X86.Bmi1.X64.ExtractLowestSetBit](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.bmi1.x64.extractlowestsetbit?view=netcore-3.1)
 - **_blsr_u64** - [System.Runtime.Intrinsics.X86.Bmi1.X64.ResetLowestSetBit](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.bmi1.x64.resetlowestsetbit?view=netcore-3.1)
 - **_tzcnt_u64** - [System.Numerics.BitOperations.TrailingZeroCount](https://docs.microsoft.com/en-us/dotnet/api/system.numerics.bitoperations.trailingzerocount?view=netcore-3.1)

## Summary

Today's compilers have very sophisticated mechanisms to optimize machine code - in 99% of cases, you will not use intrinsic functions directly by calling methods like `_blsr_u64` because the compiler will detect that programmer uses a formula to reset the lowest set bit and use [BLSR](https://www.felixcloutier.com/x86/BLSR) instruction. After all, it's just good to know that things like this exist and are ready to improve the performance of the most critical parts of our applications - it's just a matter of enabling them in settings or use directly.