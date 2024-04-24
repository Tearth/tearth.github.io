---
title: "Performance of chess engines written in C#, part 1"
date: "2020-09-20"
description: "First part of the series about chess programming and engines written using C# and .NET Core platform."
categories:
  - "Chess engines"
tags:
  - ".NET"
  - ".NET Core"
  - ".NET Framework"
  - "Chess"
sidebar: "right"
---

Last month was quite busy - I've started a new project called Cosette, which is a brand new chess engine written in C# for .NET Core platform. It's not my first project of this kind (a few years ago I made [Proxima b 2.0](https://github.com/Tearth/Proxima-b-2.0) (C#), together with even older [Proxima b](https://github.com/Tearth/Proxima-b) (C++)), so using the gained experience I can finally write a few words about performance tips and tricks, especially in C# language.

<!--more-->

## Bitboards and Magic Bitboards

My very first engine, [Proxima b](https://github.com/Tearth/Proxima-b), used array-based board representation - basically, there was a [Board](https://github.com/Tearth/Proxima-b/blob/master/Source/Board.cpp) class which contained a two-dimensional char array field. If the specified cell contained a letter, it was a sign that we have some piece on this field. This is probably the simplest way to represent board state, but at the same time, very inefficient when generating moves. Let's see at [this method](https://github.com/Tearth/Proxima-b/blob/master/Source/PieceBase.cpp#L14):

{{< highlight cpp "linenos=table" >}}
vector<Move> PieceBase::getMovesAtSide(Position currentPos, Board board, int x, int y) 
{
    Position currentMove;
    vector<Move> availableMoves;

    for (int i = 1; i < 8; i++)
    {
        currentMove = currentPos + Position(x*i, y*i);
        if (board.IsPositionValid(currentMove))
        {
            bool canKill = board.CanKill(currentPos, currentMove);
            if (board.GetFieldAtPosition(currentMove) == '0' || canKill)
            {
                availableMoves.push_back(Move(currentPos, currentMove));
            }
            else
                break;

            if (canKill)
                break;
        }
    }

    return availableMoves;
}
{{< / highlight >}}

This is the code used for sliding pieces when we want to generate available moves. Let's look at a [queen moves generator](https://github.com/Tearth/Proxima-b/blob/master/Source/Queen.cpp#L14) now:

{{< highlight cpp "linenos=table" >}}
vector<Move> Queen::GetAvailableMoves(Board board, Position pos, EColor color)
{
    vector<Move> availableMoves;

    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, 1, 0));
    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, -1, 0));
    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, 0, 1));
    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, 0, -1));

    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, 1, 1));
    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, -1, -1));
    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, 1, -1));
    availableMoves = mergeMoves(availableMoves, getMovesAtSide(pos, board, -1, 1));

    return availableMoves;
}
{{< / highlight >}}

It's not hard to find that it takes a lot of time to get these moves - and remember that we talk about processing millions of boards per second. That's why array-based representation is a very unfortunate thing for high-performance solutions. This is where bitboards gain the advantage - we store our state using 64-bit types like `ulong`, which allows using extremally fast bit arithmetic instructions like `and`, `or` and `xor`. What's more, using Magic Bitboards, we are able to precalculate moves for sliding pieces and probe them using [this simple code](https://github.com/Tearth/Cosette/tree/e433c8c9adfe92568f8585217a542db22688ad4/Cosette/Engine/Moves/Magic/MagicBitboards.cs#L26):

{{< highlight csharp "linenos=table" >}}
public static ulong GetRookMoves(ulong board, int fieldIndex)
{
    board &= _rookMagicArray[fieldIndex].Mask;
    board *= _rookMagicArray[fieldIndex].MagicNumber;
    board >>= _rookMagicArray[fieldIndex].Shift;
    return _rookMagicArray[fieldIndex].Attacks[board];
}

public static ulong GetBishopMoves(ulong board, int fieldIndex)
{
    board &= _bishopMagicArray[fieldIndex].Mask;
    board *= _bishopMagicArray[fieldIndex].MagicNumber;
    board >>= _bishopMagicArray[fieldIndex].Shift;
    return _bishopMagicArray[fieldIndex].Attacks[board];
}
{{< / highlight >}}

Way more simple, right? I don't want to explain the whole algorithm, because it's quite complicated to understand and this is not the point of this article. Chess Programming Wiki have excellent articles about [Bitboards](https://www.chessprogramming.org/Bitboards) and [Magic Bitboards](https://www.chessprogramming.org/Magic_Bitboards). A bit similar thing is with king and knight moves, where precalculating and probing moves is trivial (because they have very limited movement). 

## Regular arrays vs. stackalloc expression

C# has an excellent feature called [stackalloc](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc) which allows making an array on the stack instead of the heap. This is a very important difference because we have to remember that Garbage Collector's work is very undesired during a search of the best move (GC has to stop the thread and analyze all heap objects to find which of them are no longer used). In Cosette, I use [stackalloc](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc) expression mainly to make arrays containing generated moves and their values (used later for sorting).

{{< highlight csharp "linenos=table" >}}
Span<Move> moves = stackalloc Move[128];
Span<int> moveValues = stackalloc int[128];

var movesCount = board.GetAvailableMoves(moves);
MoveOrdering.AssignValues(board, moves, moveValues, movesCount, depth, entry);
{{< / highlight >}}

Because it's a part of negamax search, this piece of code is called very frequently - so allocating array on heap every time and cleaning it then would be a real nightmare for overall performance. Don't do it.

## Copy-Make vs. Make-Undo

We have a list of generated moves, so let's think about how to execute them. There are two main approaches:
 - **Copy-Make** - create a full copy of the board state and then apply a move. There is no need for any undo logic because reverting the state is about using the original state instead of the created copy. It's the most simple method, which at the same time is very memory-inefficient and insanely pressures Garbage Collector when writing in C#.
 - **Make-Undo** - create a separate undo logic, which allows reverting board states using information about the move that has been made. Implementation details are heavily dependent on the engine architecture, move structure, etc.

In [Proxima b](https://github.com/Tearth/Proxima-b) and [Proxima b 2.0](https://github.com/Tearth/Proxima-b-2.0) (C#), I was using the first approach due to my small experience with chess engines. It worked quite well and was simple to do, but both memory allocation on heap and copying data was a real bottleneck - even with using fast [Buffer.BlockCopy](https://docs.microsoft.com/en-us/dotnet/api/system.buffer.blockcopy?view=netcore-3.1) in [Proxima b 2.0](https://github.com/Tearth/Proxima-b-2.0). In Cosette, I've decided to try the second approach to reduce Garbage Collector pressure, which is a significant change in the engine architecture compared to the previous ones. First, let's see at the [Move](https://github.com/Tearth/Cosette/tree/e433c8c9adfe92568f8585217a542db22688ad4/Cosette/Engine/Moves/Move.cs) structure:

{{< highlight csharp "linenos=table" >}}
public readonly struct Move
{
    public readonly byte From;
    public readonly byte To;
    public readonly Piece Piece;
    public readonly MoveFlags Flags;

    // Methods not related to the article
    // ...
}

[Flags]
public enum MoveFlags : byte
{
    None = 0,
    Kill = 1,
    Castling = 2,
    DoublePush = 4,
    EnPassant = 8,
    KnightPromotion = 16,
    BishopPromotion = 32,
    RookPromotion = 64,
    QueenPromotion = 128
}
{{< / highlight >}}

The whole structure takes 32 bits, and what's more important, it contains only the most needed information. That's because: 
 - It reduces memory usage (especially in transposition tables where each entry contains the best move).
 - Not every generated move will be used due to pruning, so it doesn't make sense to calculate more things than necessary.
 
So what things are missing? I've decided to not store a piece that has been captured (this can be obtained when necessary during search or evaluation), there is no also an indicator if the move generates check or checkmate. Lack of the first one makes a bit more difficult to do an undo operation, so the engine has to store this information somewhere else. That's why board state in Cosette contains [a set of stacks](https://github.com/Tearth/Cosette/blob/e433c8c9adfe92568f8585217a542db22688ad4b/Cosette/Engine/Board/BoardState.cs#L29) to store information which would be lost when making moves:

{{< highlight csharp "linenos=table" >}}
private readonly FastStack<Piece> _killedPieces;
private readonly FastStack<ulong> _enPassants;
private readonly FastStack<Castling> _castlings;
private readonly FastStack<Piece> _promotedPieces;
private readonly FastStack<ulong> _hashes;
private readonly FastStack<ulong> _pawnHashes;
{{< / highlight >}}

When making or undoing moves, these stacks are pushing or popping values, depending on the situation. [FastStack](https://github.com/Tearth/Cosette/blob/e433c8c9adfe92568f8585217a542db22688ad4b/Cosette/Engine/Common/FastStack.cs) is a fast alternative for [Stack](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.stack-1?view=netcore-3.1) class - it has a fixed size which prevents from reallocating array during a search. Because we know what values are expected and how much, so it's not a problem.

[MakeMove](https://github.com/Tearth/Cosette/blob/e433c8c9adfe92568f8585217a542db22688ad4b/Cosette/Engine/Board/BoardState.cs#L138) and [UndoMove](https://github.com/Tearth/Cosette/blob/e433c8c9adfe92568f8585217a542db22688ad4b/Cosette/Engine/Board/BoardState.cs#L375) in Cosette are a bit complex, so again, I won't go into implementation details. The main point of this chapter is: if possible, go with Make-Undo schema, because of the reduced pressure of Garbage Collector and time saved on the memory allocating/copying.

## Intrinsics functions

Since .NET Core 3.0, we have finally able to use BMI functions directly (a few months ago I wrote an article about them: [Performance of Bit Manipulation Instructions (BMI)](http://localhost:1313/posts/performance-of-bit-manipulation-instructions-bmi/)). Because Bitboards and Magic Bitboards heavily depends on them, we can take an advantage from using them like this:

{{< highlight csharp "linenos=table" >}}
public static class BitOperations
{
#if !BMI
    private static int[] BitScanValues = {
        0,  1,  48,  2, 57, 49, 28,  3,
        61, 58, 50, 42, 38, 29, 17,  4,
        62, 55, 59, 36, 53, 51, 43, 22,
        45, 39, 33, 30, 24, 18, 12,  5,
        63, 47, 56, 27, 60, 41, 37, 16,
        54, 35, 52, 21, 44, 32, 23, 11,
        46, 26, 40, 15, 34, 20, 31, 10,
        25, 14, 19,  9, 13,  8,  7,  6
    };
#endif

    public static ulong GetLsb(ulong value)
    {
#if BMI
        return Bmi1.X64.ExtractLowestSetBit(value);
#else
        return (ulong)((long)value & -(long)value);
#endif
    }

    public static ulong PopLsb(ulong value)
    {
#if BMI
        return Bmi1.X64.ResetLowestSetBit(value);
#else
        return value & (value - 1);
#endif
    }

    public static ulong Count(ulong value)
    {
#if BMI
        return Popcnt.X64.PopCount(value);
#else
        var count = 0ul;
        while (value > 0)
        {
            value = PopLsb(value);
            count++;
        }

        return count;
#endif
    }

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

It's crucial to remember that not every processor supports BMI instructions - so having both versions is a good solution. During the tests, I've got about 10% better performance using BMI so it's worth try at least.

## Summary

This is the first part of the series about chess engines written in C# - next time, I will write something about transposition tables, and hopefully other things. Cosette is still in early development and lacks a lot of optimization, so hopefully there will be a ton of good knowledge to present. And in the meantime, feel free to play with my engine using [lichess platform](https://lichess.org/@/CosetteBot), or visit the [GitHub repository](https://github.com/Tearth/Cosette).