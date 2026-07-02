+++
title = "What a variant really weighs: BANDWIDTH versus the wire"
date = 2026-07-02
description = "The BANDWIDTH attribute in an HLS master playlist is a claim, not a measurement. Downloading real segments and weighing them is cheap — and it caught my own tool reporting 780 Mbps for an 8 Mbps stream."
+++

Every HLS master playlist opens with a series of promises. Each `EXT-X-STREAM-INF` line declares a `BANDWIDTH` — the bits per second a player should budget for that variant — and the player takes it at its word: it picks the highest rung whose declared bandwidth fits under its throughput estimate. The whole adaptive-bitrate ladder is built on those numbers being honest.

But `BANDWIDTH` is a claim, typed into a packager configuration by someone, somewhere, possibly years ago. The truth is on the wire: the segments themselves, each one a known number of bytes covering a known number of seconds. Dividing one by the other is not sophisticated. It is, however, surprisingly rarely done — so I added a `--measure` mode to [`hls-probe`](https://github.com/benjaminboisseau/hls-probe) that downloads the newest few segments of a variant and weighs them.

This post is about what that weighing tells you, and about the moment my own tool told me an 8 Mbps stream was running at 780 Mbps. We will get to that. It is educational.

## Three things a downloaded segment tells you

Once you hold real segment bytes, three facts come for free.

**The container.** The first bytes of a segment identify it. An MPEG-TS segment is a train of 188-byte packets, each starting with the sync byte `0x47` — check byte 0 and byte 188 and you know. An fMP4/CMAF segment opens with an ISO-BMFF box, so bytes 4–8 read `ftyp`, `styp`, `moof` or friends. No parsing framework required; this is the file format equivalent of reading the label on the tin:

```
measure: 3 segment(s) sampled, container MPEG-TS
```

**The real bitrate.** Segment payload bits divided by the declared `EXTINF` duration is the actual media bitrate — the thing `BANDWIDTH` claims to bound. RFC 8216 §4.3.4.2 is precise about the contract: `BANDWIDTH` must be an upper bound of the peak segment bit rate of the variant. If a sampled segment exceeds it, that is not a rounding error, it is a conformance violation with consequences: a player that budgeted for the declared figure will meet a segment it did not plan for, and its buffer pays the difference.

**The delivery timing.** Time-to-first-byte and full-download throughput per segment, which tell you how the path is behaving right now — and which fed the [cache-miss work](/posts/cache-miss-live-edge/) from the previous post.

Pointed at Unified Streaming's public live demo, the top variant declares `BANDWIDTH` 1,316 kbps (average 1,196 kbps), and the tool reports:

```
measure: 3 segment(s) sampled, container MPEG-TS
  measured bitrate 653 kbps (peak segment 653 kbps), avg TTFB 31 ms
  153 KiB in 42 ms (ttfb 27 ms) -> 29903 kbps throughput
```

Measured content comfortably under the declared ceiling — the correct relationship, since the declaration is a peak bound and this particular content moment was undemanding. Nothing dramatic. Dramatic came later.

## The confession: 780 Mbps of pure fiction

To exercise the fMP4 detection I pointed `--measure` at Apple's own public example stream — the venerable BipBop, in its fMP4 incarnation. The top rung of that ladder declares about 8,224 kbps. My tool reported, with total confidence:

```
measured bitrate 780666 kbps (peak segment 780666 kbps)
  571777 KiB in 2235 ms -> 2095390 kbps throughput  main.mp4
[warning] peak segment bitrate 780666 kbps exceeds declared BANDWIDTH 8224 kbps
```

780 Mbps. Per segment: 571,777 KiB — that is 558 megabytes — for six seconds of standard-definition-era test footage. My tool had discovered the most inefficient encoder in history, and it even raised a conformance warning against Apple for it. The prosecution rests.

The actual culprit, of course, was me. That stream uses **byte-range addressing**: instead of one file per segment, the whole variant lives in a single `main.mp4`, and each playlist entry carves out its slice with an `EXT-X-BYTERANGE` tag:

```
#EXTINF:6.00000,
#EXT-X-BYTERANGE:5874288@721
main.mp4
```

Read: this segment is 5,874,288 bytes starting at offset 721 of `main.mp4`. My tool ignored the tag and downloaded the URI — the entire 558 MB movie, once per sampled segment, three times in a row. Then it divided 558 MB by six seconds and reported the result without blinking. Garbage in, confident garbage out: a measurement tool with a bug does not fail loudly, it just lies with excellent posture.

## The fix, and the trap inside the fix

The obvious fix is to honour the tag: send an HTTP `Range: bytes=721-5875008` header and fetch only the slice. The subtle part hides in RFC 8216 §4.3.2.2: the offset in `EXT-X-BYTERANGE:<n>[@<o>]` is *optional*. When it is omitted, the sub-range begins **where the previous sub-range of the same URI ended**. The playlist is not a list of independent facts; it is a running ledger, and you must replay it from the top to know where any entry with an omitted offset actually starts. Resolve only the tail of the playlist — the newest segments, the very ones you want to sample — and every omitted offset silently resolves to zero, which puts you back in the fiction business with extra steps.

So the resolver walks the entire playlist in order, keeping a per-URI cursor, and only then hands the last few segments their absolute ranges. After the fix, on the same Apple stream:

```
measure: 3 segment(s) sampled, container fMP4/CMAF (init segment present)
  measured bitrate 7803 kbps (peak segment 7806 kbps), avg TTFB 24 ms
  5717 KiB in 104 ms (ttfb 64 ms)
```

7,803 kbps measured against a declared 8,224 kbps ceiling. Content just under its declared peak bound — exactly the relationship the RFC prescribes, recovered from a hundredfold error by reading one section of the spec more carefully.

## Why bother weighing at all

Because the declared numbers steer real decisions. A `BANDWIDTH` set too low makes players over-commit to a rung the network cannot actually sustain; set too high, it wastes quality the viewer's link could have carried. A peak segment genuinely exceeding its declaration — the warning my tool raised falsely, but which it can now raise truthfully — is the kind of defect that surfaces as intermittent rebuffering on the highest rung and resists diagnosis precisely because every dashboard shows the *declared* ladder, not the delivered one.

And the byte-range episode carries its own moral, worth more to me than the feature: a measurement is only as good as the request behind it. The numbers looked plausible enough to publish right up until they were absurd, and it took a stream I did not build — with an addressing scheme my happy path never met — to expose it. Test your instruments on material you did not produce. They will thank you by embarrassing you early, in private, which is the good timing.

## Running it yourself

```
hls-probe --measure -p 5 https://example.com/live/master.m3u8
```

`-p` picks how many of the newest segments to sample per variant; add `--json` for machine-readable output (bitrates in raw bps there; the human output speaks kbps, as one does). Byte-range playlists, init segments and both container families are handled. The tool is open source and MIT-licensed: [github.com/benjaminboisseau/hls-probe](https://github.com/benjaminboisseau/hls-probe). Issues and pull requests welcome — especially the embarrassing kind.
