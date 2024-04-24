---
title: "Inanis 1.0.1 released"
date: "2022-04-05"
description: "A small patch for the Inanis, addressing bugs found by TalkChess community."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Releases"
  - "Inanis"
issue: 13
sidebar: "right"
---

A small patch for the [Inanis](https://github.com/Tearth/Inanis), addressing bugs found by the [TalkChess](http://talkchess.com/) community. The most important one fixes random crashes caused by the move legality check, giving false indications in rare cases. In the result, illegal moves were processed which led to the board state irreversibly corrupted. 

**Strength**: 2750 Elo, **Link**: https://github.com/Tearth/Inanis/releases/tag/v1.0.1

<!--more-->

### Changelog

 - Added a new UCI option "Crash Files" (disabled by default)
 - Fixed move legality check which in rare cases was leading to engine crashes
 - Fixed PV lines being too long due to endless repetitions

### History version

| Version                                                       | Release date | Elo  | Description  |
|---------------------------------------------------------------|--------------|------|--------------|
| [1.0.1](https://github.com/Tearth/Inanis/releases/tag/v1.0.1) | 05-04-2022   | 2750 | A bunch of fixes for reported issues, stability improvement |
| [1.0.0](https://github.com/Tearth/Inanis/releases/tag/v1.0.0) | 02-04-2022   | 2750 | Initial release |