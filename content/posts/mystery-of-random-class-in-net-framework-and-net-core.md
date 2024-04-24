---
title: "Mystery of Random class in .NET Framework and .NET Core"
date: "2020-07-09"
description: "What are differences between Random class in .NET Framework and .NET Core - why sometimes it returns the same \"random\" values?."
categories:
  - ".NET internals"
  - "Performance comparisons"
tags:
  - ".NET"
  - ".NET Framework"
  - ".NET Core"
  - "Random"
issue: 6
sidebar: "right"
---

[Random](https://docs.microsoft.com/en-us/dotnet/api/system.random.next?view=netcore-3.1) class is one of the most used parts of the .NET library, which contains a few methods to generate pseudo-random numbers. They are extremely simple to use, but even with this, there are still some traps waiting for a programmer. In this article, I will focus on differences in implementation of this class between .NET Framework and .NET Core, especially seed generation which sometimes leads to interesting bugs.

<!--more-->

## Not that random values

Let's say we are writing some application using old good .NET Framework - our goal is to generate an infinity sequence of random values displayed on the console. Very simple task:

{{< highlight csharp "linenos=table" >}}
using System;

namespace RandomTest
{
    class Program
    {
        static void Main(string[] args)
        {
            while (true)
            {
                var random = new Random();
                Console.WriteLine(random.Next());
            }
        }
    }
}
{{< / highlight >}}

At first glance, everything looks ok (**except that Garbage Collector will have very hard times, but that's not the point of this article**). Someone can think that's hard to break something in that short code. It turns out, however, that we have one serious bug here which you can see at the program result below:

```
1577235169
1577235169
1577235169
806707241
806707241
806707241
1375326785
1375326785
1375326785
```

We have duplicated values, see? For some reason, [Next](https://docs.microsoft.com/en-us/dotnet/api/system.random.next?view=netframework-4.7.2#System_Random_Next) method doesn't generate new pseudo-random values but instead of it, we get the same numbers for a short period. It's even more interesting if we will try to run this code on .NET Core platform:

```
1614888221
1022877457
963858083
500339468
633473342
1408994225
1749934098
991519111
1535277395
```

Here everything looks normal. So what's the difference between both implementation in this class? This issue is quite common and it's not hard to find a solution on StackOverflow, but they usually don't explain exactly what are the difference between both platforms. Let's see it in the next chapter.

## Constructors of Random classes

We will start with examining the implementation of [Random](https://docs.microsoft.com/en-us/dotnet/api/system.random.next?view=netframework-4.7.2) class on the .NET Framework platform. As we can see on the code below, initial seed is generated using a value from the [Environment.TickCount](https://docs.microsoft.com/en-us/dotnet/api/system.environment.tickcount?view=netframework-4.7.2) property.

{{< highlight csharp "linenos=table" >}}
/// <summary>Initializes a new instance of the <see cref="T:System.Random" /> class, using a time-dependent default seed value.</summary>
[__DynamicallyInvokable]
public Random() : this(Environment.TickCount)
{
}
{{< / highlight >}}

What does it mean for us? This property returns a number of milliseconds elapsed since the system started. The problem is, if we create an instance of the [Random](https://docs.microsoft.com/en-us/dotnet/api/system.random.next?view=netframework-4.7.2) class too frequently (more than once per millisecond), we will initialize it with the same number! Therefore, if the seed is the same, then the generated pseudo-random numbers are also the same as we saw on the example in the previous chapter.

There are two possible solutions to this issue. First, we can provide our seed which ensures that will be unique for every [Random](https://docs.microsoft.com/en-us/dotnet/api/system.random.next?view=netframework-4.7.2) instance. This is quite problematic, so the better way is to create one instance (at the start of the program on example) and use it whenever you need it inside your class methods.

The same program run on the .NET Core platform behaves differently - every instance of [Random](https://docs.microsoft.com/en-us/dotnet/api/system.random.next?view=netcore-3.1) class had a unique seed that lead to unique pseudo-random numbers every iteration. Let's see what's inside the constructor:

{{< highlight csharp "linenos=table" >}}
/*=========================================================================================
**Action: Initializes a new instance of the Random class, using a default seed value
===========================================================================================*/
public Random() : this(GenerateSeed())
{

}
{{< / highlight >}}

The seed is generated inside of the `GenerateSeed` method:

{{< highlight csharp "linenos=table" >}}
[ThreadStatic]
private static Random? t_threadRandom;

/*=====================================GenerateSeed=====================================
**Returns: An integer that can be used as seed values for consecutively
           creating lots of instances on the same thread within a short period of time.
========================================================================================*/
private static int GenerateSeed()
{
    Random? rnd = t_threadRandom;
    if (rnd == null)
    {
        int seed;
        lock (s_globalRandom)
        {
            seed = s_globalRandom.Next();
        }
        rnd = new Random(seed);
        t_threadRandom = rnd;
    }
    return rnd.Next();
}
{{< / highlight >}}

We can see that every thread has its unique instance of the [Random](https://docs.microsoft.com/en-us/dotnet/api/system.random.next?view=netcore-3.1) class - this is done by using [ThreadStatic](https://docs.microsoft.com/en-us/dotnet/api/system.threadstaticattribute?view=netframework-4.7.2) attribute. The `t_threadRandom` field is initialized when the `GenerateSeed` method is called the first time, and next, it's used to generate a pseudo-random value which will be used as the seed. This explains why it behaves differently than on .NET Framework platform - we are no longer depend on the [Environment.TickCount](https://docs.microsoft.com/en-us/dotnet/api/system.environment.tickcount?view=netframework-4.7.2) property, but we create our value.

Someone can say: .NET Core maybe did it safer, but now it's slower than the implementation on .NET Framework - simple access to the property is for sure faster than messing with some additional random generator. Well, it's not entirely true. It's crucial to know that [Environment.TickCount](https://docs.microsoft.com/en-us/dotnet/api/system.environment.tickcount?view=netframework-4.7.2) isn't just simple variable read, but an entire API request which internally uses [GetTickCount](https://docs.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-gettickcount) function on Windows.

{{< highlight csharp "linenos=table" >}}
/// <summary>Gets the number of milliseconds elapsed since the system started.</summary>
/// <returns>A 32-bit signed integer containing the amount of time in milliseconds that has passed since the last time the computer was started. </returns>
[__DynamicallyInvokable]
public static extern int TickCount { [SecuritySafeCritical, __DynamicallyInvokable, MethodImpl(MethodImplOptions.InternalCall)] get; }
{{< / highlight >}}

To check which method works faster, I did a small benchmark (using [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) library) which measures how much time will take a specified .NET platform to initialize `Random` class instance and get pseudo-random value.

{{< highlight csharp "linenos=table" >}}
using System;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Running;

namespace RandomBenchmark
{
    [DisassemblyDiagnoser]
    [SimpleJob(RuntimeMoniker.NetCoreApp31)]
    [SimpleJob(RuntimeMoniker.Net472)]
    public class RandomTest
    {
        [Benchmark]
        public void Test()
        {
            new Random();
        }
    }

    class Program
    {
        public static void Main(string[] args)
        {
            BenchmarkRunner.Run<RandomTest>();
        }
    }
}
{{< / highlight >}}

```
BenchmarkDotNet=v0.12.1, OS=Windows 10.0.18363.959 (1909/November2018Update/19H2)
Intel Core i5-8300H CPU 2.30GHz (Coffee Lake), 1 CPU, 8 logical and 4 physical cores
.NET Core SDK=5.0.100-preview.5.20279.10
  [Host]        : .NET Core 3.1.6 (CoreCLR 4.700.20.26901, CoreFX 4.700.20.31603), X64 RyuJIT
  .NET 4.7.2    : .NET Framework 4.8 (4.8.4180.0), X64 RyuJIT
  .NET Core 3.1 : .NET Core 3.1.6 (CoreCLR 4.700.20.26901, CoreFX 4.700.20.31603), X64 RyuJIT
```

| Method |           Job |       Runtime |       Mean |   Error |  StdDev | Code Size |
|------- |-------------- |-------------- |-----------:|--------:|--------:|----------:|
|   Test |    .NET 4.7.2 |    .NET 4.7.2 |   556.7 ns | 3.15 ns | 2.63 ns |     413 B |
|   Test | .NET Core 3.1 | .NET Core 3.1 | 1,503.5 ns | 7.16 ns | 6.70 ns |     621 B |

The results are interesting: initialization of `Random` class instance on .NET Framework takes about three times faster than on .NET Core. It implies that reading [Environment.TickCount](https://docs.microsoft.com/en-us/dotnet/api/system.environment.tickcount?view=netframework-4.7.2) is still faster than using random generators to get the seed.

## Summary

Both platforms, .NET Framework and .NET Core, have different ways to implement `Random` class. The first one is faster and uses the system clock to generate a seed for generator - it's also more dangerous because it doesn't ensure that seeds will be different when creating frequently. The second platform depends on its internal random generators which are assigned to the threads - it's a bit slower, but guarantees that seeds will be well distributed.