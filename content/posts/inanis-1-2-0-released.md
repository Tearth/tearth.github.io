---
title: "Inanis 1.2.0 released"
date: "2023-01-15"
description: "Another major update of the Inanis chess engine: improved Syzygy support, general performance and stability improvement."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Releases"
  - "Inanis"
sidebar: "right"
---

Another major update of the [Inanis](https://github.com/Tearth/Inanis) chess engine: better Syzygy support, performance and stability improvement.

**Strength**: 2850 Elo, **Link**: https://github.com/Tearth/Inanis/releases/tag/v1.2.0

<!--more-->

### Changelog

 - Added integration with Fathom library to better support Syzygy tablebases
 - Added "tbhits" to the search output
 - Added "avg_game_phase" parameter to "tunerset" command
 - Added "syzygy" and "bindgen" as switchable Cargo features
 - Added information about captures, en passants, castles, promotions and checks in perft's output
 - Added attackers/defenders cache
 - Added killer moves as separate move generator phase
 - Removed unnecessary check detection in null move pruning
 - Removed redundant abort flag check
 - Removed underpromotions in qsearch
 - Reduced binary size by removing dependencies and replacing them with custom implementation
 - Renamed "test" command to "testset"
 - Simplified evaluation by reducing the number of score taperings 
 - Improved build process
 - Improved benchmark output
 - Improved allocation of all hashtables, now their size will always be a power of 2 for better performance
 - Improved king safety evaluation by taking a number of attacked adjacent fields more serious
 - Improved overall performance by a lot of minor refactors and adjustments
 - Improved game phase evaluation
 - Improved killer heuristic
 - Improved history table aging
 - Improved reduction formula in null move pruning
 - Fixed a few "tunerset" command bugs related to the game phase
 - Fixed PGN parser when there were no spaces between dots and moves
 - Fixed invalid evaluation of doubled passing pawns
 - Fixed invalid cut-offs statistics
 - Fixed qsearch failing hard instead of failing soft

### History version

| Version                                                       | Release date | Elo  | Description  |
|---------------------------------------------------------------|--------------|------|--------------|
| [1.2.0](https://github.com/Tearth/Inanis/releases/tag/v1.2.0) | 15-01-2023   | 2850 | Better Syzygy support, performance and stability improvement |
| [1.1.1](https://github.com/Tearth/Inanis/releases/tag/v1.1.1) | 14-08-2022   | 2800 | A bunch of fixes for reported issues, stability improvement |
| [1.1.0](https://github.com/Tearth/Inanis/releases/tag/v1.1.0) | 31-07-2022   | 2800 | Syzygy tablebases, MultiPV, adjusted evaluation |
| [1.0.1](https://github.com/Tearth/Inanis/releases/tag/v1.0.1) | 05-04-2022   | 2750 | A bunch of fixes for reported issues, stability improvement |
| [1.0.0](https://github.com/Tearth/Inanis/releases/tag/v1.0.0) | 02-04-2022   | 2750 | Initial release |