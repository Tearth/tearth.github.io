---
title: "Inanis 1.4.0 released"
date: "2024-08-03"
description: "The next update of the Inanis chess engine: check extensions, relative PST, countermove heuristic."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Releases"
  - "Inanis"
sidebar: "right"
---

A new update of the [Inanis](https://github.com/Tearth/Inanis) chess engine: check extensions, relative PST, countermove heuristic.

**Strength**: 2950 Elo, **Link**: https://github.com/Tearth/Inanis/releases/tag/v1.4.0

<!--more-->

### Changelog

 - Added relative PST
 - Added check extensions
 - Added countermove heuristic
 - Added simplified benchmark when "dev" feature is not enabled
 - Added history table penalties and reduced aging divisor
 - Added non-standard "fen" command to UCI
 - Added crash when the best move is invalid (only in dev version)
 - Added pawn attacks cache
 - Added support for LZCNT instruction
 - Improved evaluation parameters by using a new dataset for tuning
 - Improved search parameters
 - Improved header, now it also includes LLVM version, target, profile and enabled features
 - Renamed "tunerset" command to "dataset"
 - Merged "bindgen" and "syzygy" features
 - Fixed rare bug with invalid moves when a search was aborted
 - Fixed crash when ply is larger than the killer table size
 - Fixed performance overhead of setting a new position

### History version

| Version                                                       | Release date | Elo  | Description  |
|---------------------------------------------------------------|--------------|------|--------------|
| [1.4.0](https://github.com/Tearth/Inanis/releases/tag/v1.4.0) | 03-08-2024   | 2950 | Check extensions, relative PST, countermove heuristic |
| [1.3.0](https://github.com/Tearth/Inanis/releases/tag/v1.3.0) | 14-06-2024   | 2900 | Gradient descent tuner, improved SEE and evaluation |
| [1.2.0](https://github.com/Tearth/Inanis/releases/tag/v1.2.0) | 15-01-2023   | 2850 | Better Syzygy support, performance and stability improvement |
| [1.1.1](https://github.com/Tearth/Inanis/releases/tag/v1.1.1) | 14-08-2022   | 2800 | A bunch of fixes for reported issues, stability improvement |
| [1.1.0](https://github.com/Tearth/Inanis/releases/tag/v1.1.0) | 31-07-2022   | 2800 | Syzygy tablebases, MultiPV, adjusted evaluation |
| [1.0.1](https://github.com/Tearth/Inanis/releases/tag/v1.0.1) | 05-04-2022   | 2750 | A bunch of fixes for reported issues, stability improvement |
| [1.0.0](https://github.com/Tearth/Inanis/releases/tag/v1.0.0) | 02-04-2022   | 2750 | Initial release |