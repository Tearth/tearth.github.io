---
title: "Inanis 1.1.1 released"
date: "2022-08-14"
description: "A small patch for the Inanis chess engine with improved compatibility with some GUIs."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Releases"
  - "Inanis"
sidebar: "right"
---

A small patch for the [Inanis](https://github.com/Tearth/Inanis) chess engine with improved compatibility with some GUIs.

**Strength**: 2800 Elo, **Link**: https://github.com/Tearth/Inanis/releases/tag/v1.1.1

<!--more-->

### Changelog

 - Added support for FEN property in PGN parser and "tunerset" command
 - Replaced crossbeam package with native scoped threads
 - Fixed invalid handling of "isready" UCI command during a search
 - Fixed engine crash when trying to search invalid position
 - Fixed incorrect version of toolchain used in GitHub Actions

### History version

| Version                                                       | Release date | Elo  | Description  |
|---------------------------------------------------------------|--------------|------|--------------|
| [1.1.1](https://github.com/Tearth/Inanis/releases/tag/v1.1.1) | 14-08-2022   | 2800 | A bunch of fixes for reported issues, stability improvement |
| [1.1.0](https://github.com/Tearth/Inanis/releases/tag/v1.1.0) | 31-07-2022   | 2800 | Syzygy tablebases, MultiPV, adjusted evaluation |
| [1.0.1](https://github.com/Tearth/Inanis/releases/tag/v1.0.1) | 05-04-2022   | 2750 | A bunch of fixes for reported issues, stability improvement |
| [1.0.0](https://github.com/Tearth/Inanis/releases/tag/v1.0.0) | 02-04-2022   | 2750 | Initial release |