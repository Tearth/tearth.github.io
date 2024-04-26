---
title: "A few thoughts about testing chess engines"
date: "2022-07-22"
description: "Short article about a few methodologies of testing chess engines based on my 4 years of experience."
categories:
  - "Chess engines"
tags:
  - "Chess"
  - "Testing"
sidebar: "right"
---

My adventure with chess engine development started a good few years ago, and one of the issues I've definitely struggled with a lot was: how to test if a new build with a very sound feature is actually performing better than the old one. This article is a summary of the experience I gained after four engines, where each of them had a different methodology.

<!--more-->

## Manual testing

[Proxima b](https://github.com/Tearth/Proxima-b), my first chess engine and the simplest one at the same time, wasn't implementing any useful protocol for chess GUIs like Arena. It had its own graphical interface with different modes (Player vs AI, AI vs AI, AI vs FICS), so the only way to test it (I skip AI vs FICS mode for now) was manually playing games. As you can imagine, it was very unreliable because depended not only on the engine performance but also on mine, so while it might work for very simple programs where introducing major algorithms like alpha-beta can indeed show a very noticeable difference, it's pretty much useless for a more fine-tuning.

## Internet chess servers

Both [Proxima b](https://github.com/Tearth/Proxima-b) and [Proxima b 2.0](https://github.com/Tearth/Proxima-b-2.0) had the ability to connect to the [FICS](https://www.freechess.org/) (Free Internet Chess Server) - that was the time when [lichess.org](https://lichess.org/) didn't have open API yet, so it was the only way known for me to play through a network. While it was a great way to observe playing them live, you can easily observe on the chart below how dynamic and unpredictable ratings can be:

{{< image
src="/img/14/fics.jpg"
alt="Proxima b rating on FICS between Dec 2017 and Aug 2018"
caption="Proxima b rating on FICS between Dec 2017 and Aug 2018" >}}

Something very similar can be observed when looking at [Cosette's](https://github.com/Tearth/Cosette) lichess chart below. There's about 600 Elo difference between the first and the last version [according to CCRL](https://ccrl.chessdom.com/ccrl/404/cgi/compare_engines.cgi?family=Cosette&print=Rating+list&print=Results+table&print=LOS+table&print=Ponder+hit+table&print=Eval+difference+table&print=Comopp+gamenum+table&print=Overlap+table&print=Score+with+common+opponents), which is somehow also visible here, but the increase was visible after a long time and far from being stable (unless we define "stable" as ± 100 Elo).

{{< image
src="/img/14/lichess.jpg"
alt="Cosette rating on lichess between Aug 2020 and Dec 2021"
caption="Cosette rating on lichess between Aug 2020 and Dec 2021" >}}

Another problem is that the pool (so all opponents) is also constantly changing, which can lead to inflating or deflating the engine's rating, even when there were no changes. This can be also seen on the image above since Cosette wasn't really updated in a long time before the bot was shut down permanently, and still, Elo was slowly rising.

## Self-play caveats

Testing based on a self-playing, so running the engine against its older version, is one of the more popular ways to test changes. This was also my main approach during [Cosette](https://github.com/Tearth/Cosette) development (where I even made my own application called Arbiter), and while it worked, it had also two main disadvantages:
 - rating differences are often very exaggerated: because both players are the same engine (usually with some minor difference), they are prone to exploit weaknesses in the evaluation function or search itself
 - a change which worked here, might not perform well at all when playing with other opponents which have a different evaluation and search features

## Gauntlet tournaments

First of all, I use [cutechess-cli](https://github.com/cutechess/cutechess) which is probably one of the best testing applications - it's running without any GUI, so I can leave it on my VPS for multiple hours. The command below is my main way to test changes in [Inanis](https://github.com/Tearth/Inanis), so I will explain below what every option does and why it's important.

```
./cutechess-cli \
    -concurrency 4 \
    -tournament gauntlet -rounds 5000 -games 2 -repeat -ratinginterval 1 -recover \
    -engine cmd="./engines/inanis" name="Inanis DEV" proto=uci option."Crash Files"=true \
    -engine cmd="./engines/2500-2800/k2_095" proto=uci \
    -engine cmd="./engines/2500-2800/Counter-v2.9-linux-64" proto=uci \
    -engine cmd="./engines/2500-2800/weiss" proto=uci \
    -engine cmd="./engines/2500-2800/scorpio_2_84" proto=xboard \
    -engine cmd="./engines/2500-2800/GreKo-Linux-64" proto=uci \
    -engine cmd="./engines/2500-2800/ethereal_823" proto=uci \
    -engine cmd="./engines/2500-2800/inanis_1_0_1" proto=uci option."Crash Files"=true \
    -each tc=inf/5+0.1 book=Perfect2019.bin bookdepth=8 option.Hash=32 \
    -resign movecount=3 score=500 \
    -draw movenumber=50 movecount=5 score=20 \
    -tb /home/ubuntu/syzygy/ -tbpieces 6
```

 - `-concurrency 4` - the more available cores you have, the more parallel games can be played. I have about 6 cores free, but to ensure that the operating system's scheduler is not messing with the results (especially important in very short games, where every millisecond matter), I always have a very safe margin
 - `-tournament gauntlet` - we're not interested in the exact rating of other engines than Inanis, so to save time they will not play matches between each others
 - `-rounds 5000` - I'm always manually stopping tests, so the value here is just big enough to never do it automatically
 - `-games 2 -repeat` - this ensures that every opening will be played two times, with switched colors - this reduced the risk of making the result unstable due to unbalanced opening book positions
 - `-ratinginterval 1` - display information about the results after every game
 - `-recover` - some engines like to crash and we can't do much with it, so try to restart them instead of aborting the tournament
 - `-each tc=inf/5+0.1 book=Perfect2019.bin bookdepth=8 option.Hash=32` - use time control 5+0.1 with no incrementation after n moves, [Perfect2019](https://sites.google.com/site/computerschess/perfect2019books) opening book with depth restriction set to 8, and 32 MB Hash size
 - `-resign movecount=3 score=500` - stop the game if both engines indicated that the score is equal or greater than 500 for at least three moves - games like that are probably almost won anyway, so no point to waste time here
 - `-draw movenumber=50 movecount=5 score=20` - prevent games from being too long by adjudicating a draw when after 50 moves there are at least 5 ones with the score less than 20
 - `-tb /home/ubuntu/syzygy/ -tbpieces 6` - use 6-men Syzygy tablebases to end the game when the result is already known, so a bit of time can be saved

This configuration allows playing about 20000 super-fast games in 24 hours. While their quality might be not the best due to time and depth limitations, the amount of them makes the result pretty stable, with an uncertainty of measurement of about ± 4 Elo (which seems to be true based on my experience).

{{< image
src="/img/14/cutechess.jpg"
alt="Example test done after 20204 games"
caption="Example test done after 20204 games" >}}

One more important thing is how to select good sparring partners. As you can see on the image above, all opponents of Inanis DEV are weaker at this point, due to the constant improving my engine. The general rule is, the difference between both engines should be less than 200 Elo, ideally less than 100 Elo, to get the most reliable results. I'm usually just going through [CCRL Blitz list](https://ccrl.chessdom.com/ccrl/404/) to pick the best ones, with the help of [sdchess](http://www.sdchess.ru/download_engines.htm).

## Sequential Probability Ratio Test

[WORK IN PROGRESS]

## Summary

Hopefully, this short article will help some of you to make your testing procedures better and more reliable - while no one is perfect, sometimes you have to trust your intuition to predict if the change you just made really worked.