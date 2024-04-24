---
title: "Inanis 1.1.0 released"
date: "2022-07-31"
description: "Major update of the Inanis chess engine: Syzygy tablebases, MultiPV, adjusted evaluation and more."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Releases"
  - "Inanis"
issue: 15
sidebar: "right"
---

Major update of the [Inanis](https://github.com/Tearth/Inanis) chess engine, introducing a lot of new features and improvements: Syzygy tablebases, MultiPV, adjusted evaluation and more. Full changelog below.

**Strength**: 2800 Elo, **Link**: https://github.com/Tearth/Inanis/releases/tag/v1.1.0

<!--more-->

### Changelog

 - Added support for Syzygy tablebases
 - Added support for "MultiPV" UCI option
 - Added support for "searchmoves" in "go" UCI command
 - Added "hashfull" in the UCI search output
 - Added "tunerset" command
 - Added "transposition_table_size" and "threads_count" parameters to "test" command
 - Added instant move when there is only one possible in the position
 - Added new benchmarks
 - Added tuner dataset generator
 - Added information about the compiler and a list of target features at the startup
 - Added diagnostic mode in search functions to gather statistics only if necessary
 - Added a simple PGN parser
 - Removed "tries_to_confirm" parameter from "test" command
 - Removed arr_macro crate from dependencies
 - Improved mobility evaluation, now the parameters are defined per piece instead of one value for all
 - Improved null move reduction formula, now should be more aggressive
 - Improved null move pruning, now it shouldn't be tried for hopeless positions
 - Improved make-undo scheme performance
 - Improved release script, now it's shorter and more flexible
 - Improved error messages and made them more detailed
 - Improved repetition draw detection
 - Increased late move pruning max depth
 - Increased amount of memory allocated for pawn hashtable
 - Adjusted evaluation parameters
 - Made LMR less aggressive in PV nodes
 - Made aging in the transposition table faster and more reliable
 - Merged reduction pruning with late move pruning
 - Decreased memory usage during tuner work
 - Deferred evaluation of evasion mask
 - Reduced amount of lazy evaluations
 - Reduced amount of locks in the UCI interface
 - Removed duplicated search calls in the PVS framework
 - Fixed crash when "tuner" command had not enough parameters
 - Fixed crash when FEN didn't have information about halfmove clock and move number
 - Fixed crash when search in ponder mode was trying to be started in already checkmated position
 - Fixed tuner and tester not being able to examine all positions when multithreading is enabled
 - Fixed draw detection issue caused by transposition table
 - Fixed undefined behaviors and reduced the amount of unsafe code
 - Fixed incorrect benchmark statistics
 - Fixed a few edge cases in the short algebraic notation parser

### History version

| Version                                                       | Release date | Elo  | Description  |
|---------------------------------------------------------------|--------------|------|--------------|
| [1.1.0](https://github.com/Tearth/Inanis/releases/tag/v1.1.0) | 31-07-2022   | 2800 | Syzygy tablebases, MultiPV, adjusted evaluation |
| [1.0.1](https://github.com/Tearth/Inanis/releases/tag/v1.0.1) | 05-04-2022   | 2750 | A bunch of fixes for reported issues, stability improvement |
| [1.0.0](https://github.com/Tearth/Inanis/releases/tag/v1.0.0) | 02-04-2022   | 2750 | Initial release |