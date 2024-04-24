---
title: "Performance of chess engines written in C#, part 2"
date: "2021-04-12"
description: "Second part of the series about chess programming and engines written using C# and .NET Core platform."
categories:
  - "Chess engines"
tags:
  - ".NET"
  - ".NET Core"
  - ".NET Framework"
  - "Chess"
issue: 11
sidebar: "right"
---

Half-year ago I did a small text about [writing chess engines in C# and performance issues related to it](https://tearth.dev/posts/performance-of-chess-engines-written-in-csharp-part-1/), where I presented a few interesting methods of optimizing the engine. Today, I want to extend it a bit by new elements, some of them related to the lastly released .NET 5 - they aren't game-changers, but can nicely improve some parts of code.

<!--more-->

## .NET 5 update

Five months ago we had [a release of .NET 5](https://devblogs.microsoft.com/dotnet/announcing-net-5-0/), which gives us nice performance improvements, mainly in JIT and garbage collector areas. One of the most interesting features I found is a new attribute called [SkipLocalsInit](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.skiplocalsinitattribute?view=net-5.0), which prevents runtime from clearing our local variables every time when we have a method call (which was usually done for safety purposes). Let's say we have our NegaMax search function which contains this little piece of code:

{{< highlight csharp "linenos=table" >}}
Span<Move> moves = stackalloc Move[SearchConstants.MaxMovesCount];
Span<short> moveValues = stackalloc short[SearchConstants.MaxMovesCount];
{{< / highlight >}}

If we assume that `SearchConstants.MaxMovesCount` = 218 (value I set in my engine), `sizeof(Move)` = 2 and `sizeof(short)` = 2, then total size of memory needed to clear is almost 1 kilobyte! Of curse we have to adjust our code to not rely on values which weren't set, but it's definitely worth - my benchmarks showed that search speed has improved by a few percents, thanks to this one line:

{{< highlight csharp "linenos=table" >}}
[module: SkipLocalsInit]
{{< / highlight >}}

## Improved intrinsics functions

This is an extension to the chapter made in the previous part, where I told about the usage of intrinsic functions. I said there, that we can split our method to support both versions of CPU (with and without BMI support) by code like this:

{{< highlight csharp "linenos=table" >}}
public static class BitOperations
{
    public static int BitScan(ulong value)
    {
#if BMI
        return (int) Bmi1.X64.TrailingZeroCount(value);
#else
        return BitScanValues[((ulong)((long)value & -(long)value) * 0x03f79d71b4cb0a89) >> 58];
#endif
    }
}
{{< / highlight >}}

But as I found last time (and described [in this article](https://tearth.dev/posts/inlining-of-intrinsic-functions-in-dot-net-5/)), there is an even better way to do this, thanks to JIT's ability to remove dead branches of code considering current processor features:

{{< highlight csharp "linenos=table" >}}
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static int BitScan(ulong value)
{
    if (Bmi1.X64.IsSupported)
    {
        return (int)Bmi1.X64.TrailingZeroCount(value);
    }

    return BitScanValues[((ulong)((long)value & -(long)value) * 0x03f79d71b4cb0a89) >> 58];
}
{{< / highlight >}}

It's crucial to remember about `MethodIpml` attribute, as JIT is not that smart yet to inline this method in some cases.

## Dictionary vs Array

It's surprisingly frequent when I see transposition tables based on some standard library element like [map](https://www.cplusplus.com/reference/map/map/) in C++ or [Dictionary](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2?view=net-5.0) in C#. Of course, they work as intended, but their performance isn't great due to overhead in implementation. Way better solution is to make own hashmap, where we have a plain array with indexes acting as key and a set of methods to read and write. The code below presents a simple transposition table used in my engine.

{{< highlight csharp "linenos=table" >}}
public static class TranspositionTable
{
    private static TranspositionTableEntry[] _table;
    private static ulong _size;

    public static unsafe void Init(int sizeMegabytes)
    {
        var entrySize = sizeof(TranspositionTableEntry);

        _size = (ulong)sizeMegabytes * 1024ul * 1024ul / (ulong)entrySize;
        _table = new TranspositionTableEntry[_size];
    }

    public static void Add(ulong hash, TranspositionTableEntry entry)
    {
        _table[hash % _size] = entry;
    }

    public static TranspositionTableEntry Get(ulong hash)
    {
        return _table[hash % _size];
    }

    public static void Clear()
    {
        Array.Clear(_table, 0, (int)_size);
    }
}
{{< / highlight >}}

## Summary

This is the second part of performance tricks in C# chess engines - I'm not sure if a third part will be ever made, but I hope some of these tips will help to improve your programs. 