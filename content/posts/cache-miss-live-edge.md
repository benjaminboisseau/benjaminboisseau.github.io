+++
title = "Measuring the cache-miss penalty at the HLS live edge"
date = 2026-06-15
description = "Every HLS player chases the newest segment — the one least likely to be in the CDN edge cache. A small Rust tool measures the penalty that hides from every dashboard."
+++

Every HLS player spends its life chasing the same file: the newest segment. After each playlist reload it asks the CDN for the segment that was published a moment ago, plays it, and comes back for the next one. That newest segment is also the one least likely to be sitting in the CDN edge cache — and when it isn't, the player pays a penalty that almost no dashboard will show you.

So I wrote a small Rust tool, [`hls-probe`](https://github.com/benjaminboisseau/hls-probe), to measure that penalty directly — half because it matters to playback, half because the nagging suspicion that it was happening would not leave me alone. Here is what it looks for, and why it matters.

## Why the live edge is a cache's worst case

A CDN edge node caches objects on demand. The first time anyone at a given point-of-presence requests an object, the edge has nothing to serve: it forwards the request upstream — to a parent tier, and if needed all the way to your origin — fetches the bytes, caches them, and only then answers. That is a *cache miss*. Every later request for the same object, until it expires, is a *cache hit* served straight from the edge in single-digit milliseconds.

For a VOD catalogue this is a non-event: popular files are warm within seconds of publication and stay warm. The live edge breaks the assumption. The segment a player wants is, by definition, a few seconds old. If it is the first viewer to reach a particular edge node after that segment was published, its request is a guaranteed cache miss. Someone has to be first to every edge, and the CDN hands out no medal for it: the time-to-first-byte simply balloons while the edge walks the request all the way back to the origin.

The penalty is real but easy to miss, because it hides from the metrics operators usually watch. Cache-offload ratio and average edge latency are both dominated by the overwhelming majority of requests, which are warm hits. The handful of cold first-requests per segment, per edge, disappear into the average — even though they land on exactly the requests players are most time-sensitive about.

## A way to measure it: fresh versus warmed

You cannot see a cache decision from playback alone, but you can isolate it with a simple comparison. For each newly published segment:

1. Request it **immediately**, the moment it appears in the playlist — this is the *fresh* request, the one a cache miss would punish.
2. Wait one target duration, then request the **same URL again** — by now your own first request has warmed the edge, so this is the *warmed* request, a near-certain hit.

The difference in time-to-first-byte between the two is the penalty, measured on the very object a player would have fetched. Repeat it over several segments and the noise averages out. To turn the inference into proof, read the CDN's cache-status headers on both requests: a clean *miss → hit* transition confirms you measured caching and not network jitter.

`hls-probe --edge-test` automates exactly this. It watches the live playlist, catches each segment as it is published, runs the fresh/warmed pair, and reads the cache headers.

## A control: a direct origin shows no penalty

Before trusting any positive result, it is worth checking that the method does not manufacture one. Pointed at Unified Streaming's public live demo — a packager served by nginx, with no CDN cache in front of it — the tool reports honestly:

```
$ hls-probe --edge-test -p 6 https://demo.unified-streaming.com/k8s/live/stable/live.isml/.m3u8
  avg: fresh TTFB 53 ms, warmed 80 ms, penalty -27 ms
  reading: no meaningful fresh-segment penalty observed (cache already warm, or origin very close to the edge)
```

The fresh and warmed requests are statistically indistinguishable; the average "penalty" is negative, which is just noise. There is no edge cache to miss, so there is no penalty — and the tool says so rather than inventing one. That is the result you want from a control: a measurement tool that always finds something is not a measurement tool, it is a horoscope.

## The CDN signature: reading the cache headers

On a real CDN the cache decision is not a guess — the edge tells you, if you ask. The two big networks expose it directly.

CloudFront answers on every object:

```
X-Cache: Hit from cloudfront
X-Amz-Cf-Pop: FRA56-P4
Age: 7
```

The `Age` header is the tell: it counts the seconds since the edge fetched the object from origin. Scanning the same public test asset, I saw objects ranging from `Age: 73414` (warm for twenty hours) down to `Age: 7` (pulled to the edge seven seconds ago) — proof that objects genuinely cycle in and out of the cache rather than living there forever.

Akamai is even more explicit, naming the edge server and the decision:

```
X-Cache: TCP_MISS from a2-19-125-146.deploy.akamaitechnologies.com (AkamaiGHost/22.5.3-...)
```

`TCP_MISS` versus `TCP_HIT` is the cache decision in plain text. When a fresh request returns `TCP_MISS` at around a hundred milliseconds and the same segment, one segment-duration later, returns `TCP_HIT` in single-digit milliseconds, you are looking at the cache-miss penalty in the act — the first viewer per edge paying for everyone who follows. That *miss → hit* transition, locked to the freshly published segment, is the signature `--edge-test` is built to surface.

## What the penalty costs a player

A hundred milliseconds sounds harmless until you place it in the player's timeline. The player requests the newest segment the instant the playlist advances, and it is working against a buffer it is trying to keep small for latency. Two things go wrong when that request is slow.

First, the obvious one: with a tight buffer — anything tuned toward low latency — a slow first byte on every new segment eats into the safety margin and, in the worst case, stalls playback. A 100 ms penalty on a 2-second segment is five percent of the budget gone before a single media byte arrives, repeated on every segment.

Second, the sneaky one: adaptive-bitrate logic estimates available bandwidth from how fast segments download. A cache miss makes a segment arrive slowly for a reason that has nothing to do with the viewer's connection. The ABR estimator cannot tell the difference, reads the slow download as a congested network, and may switch the viewer *down* a quality level — degrading a stream whose link was perfectly healthy.

Neither failure shows up as a CDN error. The bytes were delivered; the dashboard is green; the viewer simply got a worse experience than the network could support.

## The structural fix

This is one of the problems Low-Latency HLS was designed around. With blocking playlist reloads and preload hints, the player can tell the CDN *in advance* which part is coming next, and the edge can hold the request open and stream bytes as the packager produces them — collapsing the miss-then-fetch round trip into a single pipelined exchange. The cache-miss penalty does not disappear so much as it stops being a separate, serial round trip on the critical path. Measuring the penalty first tells you how much there is to gain before you invest in the migration.

## Running it yourself

`--edge-test` needs only a live playlist URL:

```
hls-probe --edge-test -p 8 https://your-cdn.example/live/master.m3u8
```

Two honest caveats. The measurement is statistical — a single pair can be pure network jitter, so collect several and trust the cache-header transitions over any individual timing. And a probe on an unwatched staging or test channel is a pessimistic case: with no real audience, every fresh segment is a guaranteed first-request miss, whereas on a popular production channel some other viewer warms each edge within milliseconds and request coalescing spares everyone behind them. That gap between staging and production is itself worth measuring — it tells you how much of your tail latency is real and how much is an artifact of testing in an empty room.

The tool is open source and MIT-licensed: [github.com/benjaminboisseau/hls-probe](https://github.com/benjaminboisseau/hls-probe). Issues and pull requests welcome.
