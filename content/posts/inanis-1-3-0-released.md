---
title: "Inanis 1.3.0 released"
date: "2024-06-14"
description: "The next update of the Inanis chess engine: gradient descent tuner, improved SEE and evaluation."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Releases"
  - "Inanis"
sidebar: "right"
---

The next update of the [Inanis](https://github.com/Tearth/Inanis) chess engine: gradient descent tuner, improved SEE and evaluation.

**Strength**: 2900 Elo, **Link**: https://github.com/Tearth/Inanis/releases/tag/v1.3.0

<!--more-->

### Changelog

 - Added search parameters as UCI options (only if the dev feature is present)
 - Added gradient descent tuner in place of local search
 - Added internal iterative reduction
 - Added bishop pair evaluation
 - Removed "avg_game_phase" in "tunerset" command
 - Removed "magic", "testset", "tuner" and "tunerset" commands from the release builds
 - Improved king safety evaluation
 - Improved quality of tunerset output
 - Improved search parameters
 - Improved pawn structure evaluation
 - Improved mobility evaluation by excluding squares attacked by enemy pawns
 - Fixed invalid position score when both kings are checked
 - Fixed incorrect SEE results for sliding pieces 

### History version

| Version                                                       | Release date | Elo  | Description  |
|---------------------------------------------------------------|--------------|------|--------------|
| [1.3.0](https://github.com/Tearth/Inanis/releases/tag/v1.3.0) | 14-06-2024   | 2900 | Gradient descent tuner, improved SEE and evaluation |
| [1.2.0](https://github.com/Tearth/Inanis/releases/tag/v1.2.0) | 15-01-2023   | 2850 | Better Syzygy support, performance and stability improvement |
| [1.1.1](https://github.com/Tearth/Inanis/releases/tag/v1.1.1) | 14-08-2022   | 2800 | A bunch of fixes for reported issues, stability improvement |
| [1.1.0](https://github.com/Tearth/Inanis/releases/tag/v1.1.0) | 31-07-2022   | 2800 | Syzygy tablebases, MultiPV, adjusted evaluation |
| [1.0.1](https://github.com/Tearth/Inanis/releases/tag/v1.0.1) | 05-04-2022   | 2750 | A bunch of fixes for reported issues, stability improvement |
| [1.0.0](https://github.com/Tearth/Inanis/releases/tag/v1.0.0) | 02-04-2022   | 2750 | Initial release |