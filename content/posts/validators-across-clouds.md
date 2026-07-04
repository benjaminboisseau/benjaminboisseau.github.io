+++
title = "One stream, two CDNs: the validators Google wants and MediaPackage doesn't send"
date = 2026-07-04
description = "Google's CDNs refuse to cache objects over 1 MiB unless the origin sends ETag or Last-Modified. AWS MediaPackage sends neither, because CloudFront never asks. Cross the clouds and your cache offload quietly collapses."
+++

Here is a failure mode with no error message. Take an HLS origin that has served happily behind one CDN for years, point a different CDN at it — a perfectly normal thing to do in a multi-CDN world — and watch your cache offload collapse. Playback still works. Every dashboard is green. The only symptom is that your origin is suddenly doing the work the CDN was hired for, and billing you for the privilege.

The specific pair I want to talk about is AWS Elemental MediaPackage as the origin and Google Cloud CDN as the delivery layer, because the mismatch between them is documented, sharp, and invisible until you know where to look. It comes down to two response headers: `ETag` and `Last-Modified`.

## What Google demands before caching big objects

Both of Google's CDNs put conditions on caching large objects, and "large" starts at exactly 1 MiB.

[Cloud CDN](https://cloud.google.com/cdn/docs/caching) caches objects bigger than 1 MiB through *chunked cache fill*: it fetches them from the origin as a series of byte-range requests. But it will only do that once the origin has proven it can be trusted with range requests, and the proof is a response carrying **all** of:

- `Accept-Ranges: bytes`
- a valid `Content-Length` (or `Content-Range` on a 206)
- `Last-Modified` **and** a strong `ETag`

The validators are not decoration. When the cache assembles one object out of several range responses, the `ETag` and `Last-Modified` are how it checks all the pieces came from the same version of the resource — nobody wants a segment whose first megabyte is from one encoder restart and whose second is from another. An origin that fails the test is treated as not supporting byte ranges at all: no chunked fill, cacheable size capped at 10 MiB, and a response over 1 MB is declined for caching on first request.

[Media CDN](https://cloud.google.com/media-cdn/docs/caching) — the one Google actually aims at video delivery — is stricter and blunter. To cache origin responses larger than 1 MiB, the origin must send a validator: `Last-Modified` *or* `ETag`, plus a valid `Date` and `Content-Length`. And since objects over 1 MiB are fetched as byte-range requests of up to 2 MiB each, the documentation adds, without softening the blow: "Objects larger than 1 MiB aren't served if byte ranges are not supported on the origin."

Now do the arithmetic on a video segment. Four seconds of 1080p at 5,000 kbps is about 2.4 MiB. Even a two-second segment at that bitrate crosses the 1 MiB line. Essentially every video segment above SD quality lives in exactly the range where Google's caching rules have opinions. Audio renditions sneak under the bar; your video does not.

## What MediaPackage actually sends

AWS Elemental MediaPackage is a just-in-time packager: it originates HLS and DASH on request, and it is explicitly designed to sit behind a CDN. Its cache guidance is genuinely good — [AWS documents](https://docs.aws.amazon.com/mediatailor/latest/ug/cdn-emp-caching.html) the `Cache-Control` values its origin endpoints set, and they are exemplary: fourteen days on media segments (immutable once created), half a segment duration on manifests (always changing), one second on LL-HLS playlists.

What its responses do not carry is a validator. No `ETag`, no `Last-Modified`. And you can read the reason straight out of the same documentation: every word of AWS's CDN optimization guidance for MediaPackage is written for CloudFront, and **CloudFront doesn't need validators to cache**. It caches on `Cache-Control` alone, whatever the object size. For a just-in-time packager the omission is even understandable — what is the honest `Last-Modified` of a segment that is generated on demand? The origin is not broken. It is specialized: a purpose-built origin for a CDN that never asks the question Google asks.

Put the two halves together and the trap closes. Behind CloudFront: fourteen-day TTLs, beautiful offload, case closed. Put Google Cloud CDN or Media CDN in front of the same endpoint — multi-CDN, cloud migration, a delivery contract that changed hands — and every video segment is an object over 1 MiB from an origin with no validator. Cloud CDN declines them; Media CDN's documentation says they aren't served at all. The stream that cached perfectly on one cloud doesn't cache on the other, and nothing anywhere reports an error, because delivering every byte from the origin is not an error. It is just an invoice.

## Making the audit automatic

This is a property you can check from the outside in one request, so I taught [`hls-probe`](https://github.com/benjaminboisseau/hls-probe) to check it. As of v0.4, `--measure` reports the cache-relevant headers on the segments it samples and warns when they collide with the documented rules:

```
$ hls-probe --measure https://demo.unified-streaming.com/k8s/live/stable/live.isml/.m3u8
  origin headers: ETag "usp-84377CE5", Last-Modified Sat, 04 Jul 2026 13:26:37 GMT,
                  Accept-Ranges bytes, Cache-Control absent
  [warning] no Cache-Control on segments: CDNs in honor-origin mode
            will not cache them at all
```

That is a plain nginx-fronted packager: validators for free (file-backed servers can hardly avoid them), but no `Cache-Control` — the exact mirror image of MediaPackage's profile, and a different way to lose your cache. Apple's Akamai-fronted test stream, by contrast, walks in with everything: strong `ETag`, `Last-Modified`, `Accept-Ranges: bytes`, `Cache-Control: max-age=600, public` — segments of 5.7 MiB that any CDN on any cloud will happily cache. And an origin with the MediaPackage profile — `Cache-Control` present, validators absent, segments over 1 MiB — gets the warning this feature exists for:

```
  [warning] segments up to 2456 KiB carry no ETag or Last-Modified: Google Media CDN
            will not cache objects over 1 MiB without a validator
  [warning] origin fails Google Cloud CDN's byte-range test (needs Accept-Ranges:
            bytes + strong ETag + Last-Modified): no chunked cache fill,
            cacheable size capped at 10 MiB
```

The pattern that emerges from probing around is telling: origins that serve files — nginx, Apache, S3 — emit validators without anyone asking, because the file system hands them a modification time and a hash costs nothing. Origins that *synthesize* content just in time have nothing convenient to put in those headers, so they don't. Which means the absence goes unnoticed for exactly as long as your CDN doesn't care, and becomes load-bearing the day you switch.

## What to do about it

Three honest options, in the order I would try them.

**Interpose something that adds validators.** A thin proxy between Google's CDN and MediaPackage can synthesize them. For media segments this is safe *because* they are immutable: a strong `ETag` derived from the segment URL and a `Last-Modified` set at first sight will never wrongly validate a changed object, since the object never changes. Manifests are the opposite — they change constantly under the same URL, so either exempt them (they are tiny and short-TTL anyway; the 1 MiB rules don't touch them) or derive the `ETag` from the manifest body. An incorrect validator is worse than no validator; immutability is what makes this one correct.

**Chain the CDNs.** Keep CloudFront directly in front of MediaPackage — the pairing AWS designed — and let the other CDN treat CloudFront as its origin. You pay a double hop, but CloudFront's responses toward the second CDN can carry what MediaPackage's don't.

**Check before you sign.** If you are evaluating a CDN change, one `hls-probe --measure` against your origin tells you in ten seconds whether your segments carry what the destination CDN requires. It is the cheapest due diligence you will do all year.

The general lesson generalizes past these two vendors: *cacheable* is not a property of your stream, it is a property of your stream *and* the CDN reading it. Every origin-CDN pair renegotiates the contract, the terms are buried in each CDN's caching documentation, and the penalty clause is silent. Read the headers your origin actually sends — they are one request away, and they are the contract.

The tool is open source and MIT-licensed: [github.com/benjaminboisseau/hls-probe](https://github.com/benjaminboisseau/hls-probe).
