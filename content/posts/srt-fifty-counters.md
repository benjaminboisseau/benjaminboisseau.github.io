+++
title = "Fifty counters and no diagnosis: reading SRT statistics like a doctor"
date = 2026-07-06
description = "libsrt tells you everything about a dying stream except what is killing it. I built a small tool that samples the stats at 100 ms and names the disease — then let a lab full of broken networks prove its first three diagnoses wrong."
+++

My previous posts lived in the HLS world, where observability is almost a solved problem: everything is HTTP, every hop writes a log line, and when something breaks you follow the request. Contribution is the opposite country. An SRT feed is one encrypted UDP flow between two boxes, and when it breaks up there is no access log to read. What you get instead is `srt_bstats()`: a C struct with more than fifty counters, updated live, documented tersely, and — this is the maddening part — all of them *true* without any of them telling you what is actually wrong.

Everyone who has operated SRT links knows the support call. The picture macroblocked at 14:32. The counters show loss, retransmissions, drops, belated packets, buffer levels, an RTT estimate and a bandwidth estimate. Which one is the disease and which ones are symptoms? Is the network bad? Is the latency configured too low? Is the link simply too small? Those three problems produce superficially similar walls of numbers and demand three *incompatible* fixes.

So I built [`srt-doctor`](https://github.com/benjaminboisseau/srt-doctor), a small Rust CLI that sits at either end of a live SRT session, samples the full stats structure every 100 ms, and tries to name the condition instead of printing the wall. This post is about the three diseases, the stats signatures that separate them, and — in the honest tradition of this blog — the three times the doctor itself confidently named the wrong disease on a lab-controlled network and had to be corrected by its own data.

## Three diseases, three incompatible cures

**Disease one: the network is lossy, and SRT is coping.** Packets are lost (`pktRcvLoss` climbs), NAKs fly back, retransmissions arrive (`pktRcvRetrans` climbs), and every gap is filled before its play deadline: `pktRcvDrop` stays at zero. This looks alarming on a dashboard and is in fact the system working exactly as designed. The cure is to do nothing.

**Disease two: the latency budget is too small.** Same lossy network — but now recovery loses the race. A retransmission takes at least one round trip to arrive; if `SRTO_LATENCY` doesn't buy enough time for that round trip (plus the next one, when the retransmission itself is lost), the receiver gives up and skips: `pktRcvDrop` climbs, and the viewer sees it. Nothing is wrong with the network that wasn't wrong in disease one. The cure is one settings change, and no amount of network engineering will substitute for it.

**Disease three: the link cannot carry the stream.** This is the sneaky one, because its loudest symptom is *silence*: often zero loss reported. The sender's own buffer backs up (`msSndBuf` grows), and the sender itself starts discarding packets it never got to send (`pktSndDrop`). More latency — the reflex fix — makes this strictly worse, because it just gives the doomed queue more room to grow.

The receiving end sees these three as: loss with retransmissions and no drops; loss with retransmissions *and* drops; and drops with **no** retransmissions at all — that last signature meaning the missing packets were never sent, not lost in flight. Three different columns in the same stats struct. Once you know to read them as a differential diagnosis, the wall of counters collapses into one sentence.

## A lab that cannot lie to me

Field anecdotes are not evidence, and I don't write about my employer's links. So the data in this post comes from a reproducible lab that fits in a [shell script](https://github.com/benjaminboisseau/srt-doctor/blob/main/lab/run-scenarios.sh): a veth pair with one end in a Linux network namespace, `tc netem` degrading the link in controlled ways, ffmpeg producing a constant 3.5 Mbps transport stream, and `srt-doctor` instrumenting **both** ends of the SRT session — because, as disease three shows, the sender knows things the receiver can only infer.

The core experiment is a pair of runs on the *identical* impaired network — 80 ms RTT, 2% random loss — differing only in configured latency.

With `latency=500ms` (about six round trips of budget):

```
packets: 20681 unique | lost 398 (1.888%) | retransmitted 411 | dropped 0 (0.000%)

diagnosis: lossy but recovering
  - The link lost 398 packets (1.89%) but every one was retransmitted in time:
    the network is imperfect and the configuration is absorbing it.
  - configured 500 ms covers 4 retransmission round(s) at 1.89% loss — adequate
```

With `latency=80ms` — equal to the RTT, so no retransmission can *ever* arrive in time:

```
packets: 20415 unique | lost 452 (2.166%) | retransmitted 448 | dropped 272 (1.332%)

diagnosis: LATENCY-STARVED
  - 272 packets were dropped unrecovered: the latency budget loses the race
    against loss recovery.
  - configured 80 ms is too small: 2.17% loss at 80.2 ms RTT (p95) needs
    ~482 ms to cover 4 retransmission round(s)

timeline (1 char = 1s): rLLLLLLLLLLLLLLLLLLLLLLLLLLLLrLLLLLLLLLLLLLLLLLLLLLLLLLLLLLL
```

Same network. Same loss. Zero drops versus 272 drops — visible artifacts roughly every 200 ms for a minute. The difference between "nothing to fix" and "unwatchable" was one integer in a configuration file.

## Where the recommended number comes from

The rule of thumb passed from control room to control room says "set SRT latency to 4× RTT". It works until the day it doesn't, because it compresses a piece of reasoning worth doing in full — and the full version fits on a napkin.

Follow one lost packet. The receiver notices a hole in the sequence numbers and asks for that packet again (a NAK). The request crosses the network one way, the retransmission comes back the other: roughly one round trip, plus a little slack — call it 1.5× RTT to be safe. The SRT latency is simply how long the receiver agrees to wait for a packet before playing on without it. So the latency budget is really a number of *attempts*: how many times can this one packet be requested and re-sent before its deadline expires?

One attempt is enough — if the retransmission arrives. But on a link that loses 2% of packets, the retransmission itself dies one time in fifty. Then you need a second attempt, occasionally a third. How many should the budget cover? Enough that repeated bad luck becomes invisible: I target one packet in a million still missing after all attempts, which on a typical stream means about one visible glitch per hour. At 2% loss, the odds of the same packet dying four times in a row are about one in six million — below the target. So: four attempts × 1.5 × 80 ms ≈ 480 ms. That is exactly what the tool recommended above, and what the 500 ms run confirmed empirically: 398 losses, zero drops.

The same arithmetic explains both ways the 4× RTT shortcut fails: at 0.1% loss, two attempts suffice and 4× RTT wastes delay you could give back to the viewer; at 5% loss it takes five attempts and 4× RTT quietly ruins your evening.

## The disease that reports no loss

The bandwidth-starvation run caps the link at 2.5 Mbps under a 3.6 Mbps stream. The sender's report is unambiguous — and note the RTT line:

```
RTT min/mean/p95/max: 100.0/2888.2/3388.0/3449.2 ms
packets: 20688 unique | lost 0 (0.000%) | retransmitted 0 | dropped 0 | sender-dropped 19988

diagnosis: BANDWIDTH-STARVED
  - The sender dropped 19988 packets with a backed-up send buffer: the link
    cannot carry the stream rate. Reduce the encoder bitrate, raise MAXBW,
    or fix the link — extra latency will not help.
```

Zero loss, zero retransmissions, and the sender discarded *half the stream* before it ever touched the wire. Meanwhile the RTT — 20 ms on this link when idle (the 100.0 in the min column is just libsrt's starting estimate, reported before the first real measurement could ever complete) — inflated to 3.4 **seconds**, because the rate cap's queue filled and every ACK waited behind a quarter-second of doomed video. That queue growth is bufferbloat doing to a contribution feed exactly what it does to your home upload during a backup, and it is the receiver's best clue: sequence gaps arriving with **zero** retransmissions while the RTT balloons means the missing packets were never sent at all.

The tool's first diagnosis of this case was exactly wrong, and it's worth confessing how. The receiver saw 31% "loss" and dutifully computed a latency recommendation: *increase latency to roughly 40745 ms*. Forty seconds. The math was internally correct and the medicine would have been poison — the queue does not care how long you are willing to wait for packets the sender already threw away. The current version recognizes the no-retransmissions-plus-inflated-RTT signature and refuses to give latency advice at all, because the only honest advice is: your link is too small.

## The disease that isn't one

The best find of the lab was an accident. I wanted a "jitter" scenario and gave netem `delay 30ms 15ms` — a 30 ms transit time, plus or minus 15 ms of random variation. Jitter is exactly that: not how long the trip takes, but how much the trip time *varies* from one packet to the next. And varying trip times have a consequence that is easy to miss: they shuffle the deck. Take two packets sent 5 ms apart. If the first one draws 44 ms of delay and the second draws 18 ms, the second arrives 21 ms *before* its elder. The network lost nothing — it reordered.

Now watch it from the receiver's chair. Packets arrive numbered 1, 2, 5 — and SRT concludes, instantly, that 3 and 4 are missing. That instant reaction is a deliberate design choice, and against real loss it is SRT's greatest quality: not a millisecond wasted before requesting the missing packet again. Against reordering it becomes a hair trigger: 3 and 4 were never lost, they are 20 ms behind, already in flight. The NAK goes out anyway, the sender re-sends both, and a moment later the receiver holds two copies of each — the original, arriving fashionably late, and a retransmission nobody needed. Multiply by every shuffled packet on the link:

```
packets: 20688 unique | lost 15816 (43.327%) | retransmitted 15797 |
         dropped 0 (0.000%) | belated 15796

diagnosis: jittery
  - Jitter is reordering packets: 15816 "lost" packets came back belated (15796)
    with zero unrecovered drops. Not real loss — but the false NAKs cost 76%
    extra bandwidth in spurious retransmissions. Consider SRTO_LOSSMAXTTL
    to tolerate reordering.
```

Forty-three percent reported loss. Zero drops. A perfect picture — and a wire carrying 6.4 Mbps for a 3.6 Mbps stream, three-quarters of it retransmissions nobody needed. This failure mode is not a lab curiosity: bonded cellular links reorder by construction (two halves of the stream racing down two carriers), WiFi reorders under retry, and per-packet load balancing in carrier cores does it under congestion. It terrifies dashboards, and it has a one-line cure almost nobody deploys: `SRTO_LOSSMAXTTL N` tells the receiver to wait until N more packets have arrived before declaring a gap a loss — tolerance for shuffling instead of a hair trigger. It ships disabled (N = 0), because tolerance costs reaction time when the loss is real; but on a link known to reorder, a small N converts 76% overhead into none. The counter that unmasks the whole charade is `pktRcvBelated` — "lost" packets that arrived after being given up on. When belated ≈ lost and drops = 0, you do not have a loss problem; you have an ordering problem and a bandwidth bill.

That `pktRcvBelated` counter also delivered my second confession: an earlier version of the diagnosis treated *any* belated packet as proof of latency starvation. The recovered-loss run then showed 24 belated packets with zero drops — late *duplicate* retransmissions, costing nothing — and the tool cried "latency-starved" over a perfectly healthy stream. The counters are subtler than they look: belated with drops means starvation; belated without drops means duplicates or reordering. One column, two diseases, distinguished only by its neighbor.

## What 100 ms sampling buys

All of the above used one-second verdicts built from 100 ms samples, and the resolution matters: a per-minute average of the latency-starved run reads "2.2% loss" — indistinguishable from the healthy run's "1.9%". The damage lives in the structure, not the average.

A fair objection: can a burst shorter than 100 ms slip *between* two samples? No — and the reason is worth spelling out, because it is why polling works at all. libsrt has no event stream for statistics; the only interface is `srt_bstats()`, which copies out *cumulative counters* that the library updates on every single packet. Sampling therefore never misses an event: a 30 ms burst that kills 15 packets deposits all 15 into whichever sample covers it. What the interval sets is the timing *precision* — at 100 ms you know which tenth of a second hurt, not which millisecond. And since the call is nothing more than a locked struct copy, the dial turns when you need it to: `--interval-ms 10` runs happily (989 samples over a ten-second session on this same lab) and resolves burst structure below frame duration. 100 ms is the default because it separates every disease in this post while producing logs a human can still read.

The last lab scenario shows what the default already catches: a Gilbert-Elliott loss model (rare bad states that eat ~10 consecutive packets) produced a polite 1.28% run average, and the tool's view of the fine structure:

```
  - loss is concentrated: only 4.3% of 100 ms samples saw any loss, but each
    hit lost ~10 packets — micro-bursts a coarse average would smooth away.
  - configured 300 ms is too small: 1.28% loss at 60.1 ms RTT (p95) needs
    ~361 ms to cover 4 retransmission round(s)
```

That advisor verdict earned its keep here: 300 ms configured against a computed need of ~361 ms predicts a stream that *almost* copes — and indeed 269 packets were lost, 263 recovered, and exactly 6 dropped. A margin problem you would never find in a weekly average. Averages are how a stream dies with a green dashboard.

Everything here is reproducible from the repo — the tool, the lab script, and the raw JSONL sample logs behind every quoted report. If you run contribution links and have a war story where the stats lied to you, I want to hear it.

*Tools used: [srt-doctor](https://github.com/benjaminboisseau/srt-doctor) (Rust, MIT), libsrt 1.5.5, ffmpeg, tc/netem on Alpine Linux.*
