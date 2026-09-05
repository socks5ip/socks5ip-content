---
title: "How to Test Proxy Quality: Uptime, Speed, Purity in One Pass"
description: "A practical proxy test routine: real protocol handshake, latency/jitter/uptime over 50-100 attempts, ASN/residential flags and blacklist checks, and what pass/fail thresholds to use."
date: 2026-09-05
tags: [proxy, testing, quality, uptime]
canonical: https://socks5ip.com.cn/proxy-check/
---

# How to Test Proxy Quality: Uptime, Speed, Purity in One Pass

## Bottom line

**Test three dimensions before you trust a line: real connectivity (protocol handshake), stability (50–100 connection attempts), and purity (owner type + blacklist).** A line that passes all three is worth paying for; a line that fails any is a gamble.

## 1. Connectivity: does it actually connect?

Many proxies answer pings but fail real handshakes, or vice versa. Test with a **real protocol handshake** — for SOCKS5 that means the RFC 1928 negotiation, for HTTP the CONNECT method — not just an ICMP ping or a page fetch. Wrong credentials, wrong protocol type and dead ports all show up here.

## 2. Stability: is it consistent?

Run **50–100 connection attempts** across different times of day and record:

- **Uptime rate**: ≥98% is good; below 95% is a problem for any serious work.
- **Latency**: measure a real round trip through the proxy, not ping to the server. Domestic China lines typically 20–80 ms; cross-border lines 100–300 ms.
- **Jitter & packet loss**: a line with high jitter or loss is unreliable even when the average looks fine.
- **Exit consistency**: does the exit address stay in the same range, or jump around?

One burst of 100 tests in a minute tells you less than a handful of sessions spread over a day — schedule a small test over 24 hours before committing.

## 3. Purity: does it look clean?

Run the exit through an IP check:

- **Owner type**: ISP name = residential/mobile; cloud/hosting name = datacenter.
- **Blacklist hits**: near-zero hits is clean; multiple hits means the range is commonly abused.
- **Risk score**: any serious checker aggregates the above into a risk score — use it as the tiebreaker.

For account-sensitive work, residential + zero hits is the bar. For scraping, datacenter with low hits is acceptable.

## Recommended thresholds

| Metric | Pass | Caution |
|---|---|---|
| Protocol handshake | Succeeds first try | Fails intermittently |
| Uptime (50–100 tests) | ≥98% | <95% |
| Latency (domestic) | 20–80 ms | >150 ms |
| Packet loss | <1% | >3% |
| Blacklist hits | 0–1 | Multiple |

## Where to test

The main site hosts a proxy line checker (canonical link above) that runs real handshakes, grades latency and batch-checks lines — paste your trial lines there before paying, and re-check periodically because IP pools change.
