---
title: "Inanis 1.5.0 released"
date: "2024-11-01"
description: "The next update of the Inanis chess engine: aspiration windows, improved performance and multithreading."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Releases"
  - "Inanis"
sidebar: "right"
---

A new update of the [Inanis](https://github.com/Tearth/Inanis) chess engine: aspiration windows, improved performance and multithreading.

**Strength**: 3000 Elo, **Link**: https://github.com/Tearth/Inanis/releases/tag/v1.5.0

<!--more-->

### Changelog

 - Added aspiration windows
 - Added support for tuning based on evaluation
 - Added "k" and "wdl_ratio" parameters to "tuner" command
 - Added prefetching of transposition table entries
 - Added packed evaluation
 - Added pawn structure to fast evaluation
 - Removed "lock material" parameter from "tuner" command
 - Removed "Crash Files" UCI option from the release version
 - Removed Lazy SMP artificial noise
 - Removed allocator, pawn hashtable has fixed size now
 - Improved tuner performance and excessive memory usage
 - Improved tuner output by setting unused parameters to zero
 - Improved indexing of transposition, pawn and perft tables
 - Improved rand algorithm (change from xorshift to xorshift*)
 - Improved engine strength when using multiple threads
 - Improved evaluation parameters
 - Improved overall performance
 - Incorporated piece values in PST
 - Fixed endless search in fixed-nodes mode
 - Fixed tuner output filename
 - Fixed crash when parsing certain FENs
 - Fixed incorrect workload between threads in tuner
 - Fixed incorrect evaluation of black pawn chains

### History version

| Version                                                       | Release date | Elo  | Description  |
|---------------------------------------------------------------|--------------|------|--------------|
| [1.5.0](https://github.com/Tearth/Inanis/releases/tag/v1.5.0) | 01-11-2024   | 3000 | Aspiration windows, improved performance and multithreading |
| [1.4.0](https://github.com/Tearth/Inanis/releases/tag/v1.4.0) | 03-08-2024   | 2950 | Check extensions, relative PST, countermove heuristic |
| [1.3.0](https://github.com/Tearth/Inanis/releases/tag/v1.3.0) | 14-06-2024   | 2900 | Gradient descent tuner, improved SEE and evaluation |
| [1.2.0](https://github.com/Tearth/Inanis/releases/tag/v1.2.0) | 15-01-2023   | 2850 | Better Syzygy support, performance and stability improvement |
| [1.1.1](https://github.com/Tearth/Inanis/releases/tag/v1.1.1) | 14-08-2022   | 2800 | A bunch of fixes for reported issues, stability improvement |
| [1.1.0](https://github.com/Tearth/Inanis/releases/tag/v1.1.0) | 31-07-2022   | 2800 | Syzygy tablebases, MultiPV, adjusted evaluation |
| [1.0.1](https://github.com/Tearth/Inanis/releases/tag/v1.0.1) | 05-04-2022   | 2750 | A bunch of fixes for reported issues, stability improvement |
| [1.0.0](https://github.com/Tearth/Inanis/releases/tag/v1.0.0) | 02-04-2022   | 2750 | Initial release |