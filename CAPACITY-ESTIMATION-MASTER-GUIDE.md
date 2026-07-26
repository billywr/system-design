# Capacity Estimation Master Guide — Metrics, Units, and Back-of-Envelope Math

> **Goal:** In any system design interview, spend 5–10 minutes on numbers that **prove you understand scale** — not fantasy precision, but defensible orders of magnitude.

---

## How to Use This Guide

1. **Memorize the metric dictionary** (Part 1) — what MAU, QPS, TPS mean
2. **Learn the 6 core formulas** (Part 2) — storage, bandwidth, QPS, latency, concurrency
3. **Practice the worked examples** (Part 5) — say numbers out loud
4. **Before each case study**, skim **Part 4** — which metrics that system cares about

**Interview flow (recommended):**

```
Assumptions (DAU, actions/user/day)
    -> Daily volume
    -> QPS (avg + peak)
    -> Storage (5-year)
    -> Bandwidth (ingress + egress)
    -> Sanity check ("does this feel like Instagram/Twitter scale?")
```

---

# Part 1 — Metric Dictionary (What Every Term Means)

## 1.1 User and activity metrics

| Metric | Full name | Meaning | Typical use |
|--------|-----------|---------|-------------|
| **MAU** | Monthly Active Users | Unique users active at least once in 30 days | Product size, growth, ad inventory |
| **DAU** | Daily Active Users | Unique users active at least once in 24 hours | Engagement, load planning |
| **WAU** | Weekly Active Users | Unique users active in 7 days | Mid-funnel between DAU and MAU |
| **Stickiness** | DAU / MAU | How often monthly users come back daily | 0.5 = half of MAU use daily; social apps often 0.4–0.6 |
| **Sessions / user / day** | — | How many app opens per DAU | Feed apps: 3–8; messaging: 10–20 |
| **Actions / user / day** | — | Likes, messages, searches, rides, etc. | Drives write QPS |

**Example:**

```
Instagram-like: 1B MAU, 500M DAU
Stickiness = 500M / 1B = 0.5 (50% of monthly users are daily)
```

**When to use which:**

| Question in interview | Metric |
|-----------------------|--------|
| "How big is the product?" | MAU or DAU |
| "How much read traffic?" | DAU x sessions x requests per session |
| "How much write traffic?" | DAU x actions per user per day |

---

## 1.2 Throughput metrics (requests per time)

| Metric | Meaning | Same as | Notes |
|--------|---------|---------|-------|
| **QPS** | Queries per second | RPS (requests/sec) in many contexts | Reads + writes combined unless specified |
| **TPS** | Transactions per second | Often **writes** or DB commits | Payment, booking, inventory |
| **RPM** | Requests per minute | QPS x 60 | Legacy dashboards |
| **EPS** | Events per second | Analytics, Kafka, logs | View events, clicks, location pings |
| **MPS** | Messages per second | Chat, notifications | WhatsApp, Discord |

**Conversion formulas:**

```
QPS (average) = total requests per day / 86,400

QPS (peak)    = QPS (avg) x peak multiplier   (often 2x–5x; flash sales 10x–100x)

Requests/day  = DAU x actions_per_user_per_day
              (or MAU x monthly_actions / 30)
```

**86,400** = seconds in a day (memorize).

**Quick mental math:**

| Per day | Divide by | Approx QPS |
|---------|-----------|------------|
| 8.6M | 86,400 | ~100 |
| 86M | 86,400 | ~1,000 |
| 864M | 86,400 | ~10,000 |
| 8.6B | 86,400 | ~100,000 |

---

## 1.3 Read vs write split

| Pattern | Read:Write | Examples |
|---------|------------|----------|
| Social feed | 100:1 to 1000:1 | Instagram, Twitter |
| Search | 1000:1 | Google Search |
| Chat | 1:1 to 5:1 | WhatsApp (send + sync + receipts) |
| Video stream | 10000:1 (egress heavy) | YouTube, Netflix |
| Payments | 1:10 read:write on ledger | Authorize + capture + settle |
| URL shortener | 100:1 | Redirect reads >> create |

Always **state your ratio** in the interview — it drives cache and DB design.

---

## 1.4 Latency metrics

| Term | Meaning | Interview use |
|------|---------|-----------------|
| **Latency** | Time from request sent to response received | "p99 feed load < 300ms" |
| **RTT** | Round-trip time (network there + back) | Cross-region: 50–150ms |
| **TTFF** | Time to first frame (video) | YouTube: < 2s |
| **p50 (median)** | Half of requests faster than this | Typical user experience |
| **p95 / p99** | 95th / 99th percentile | Tail latency — what SLOs use |
| **SLA / SLO** | Target reliability or latency | 99.9% availability = 8.7h downtime/year |

**Why p99 matters:** Average hides slow users. Interviewers want "p99 read < 100ms from cache."

**Little's Law (concurrency):**

```
Concurrent requests = QPS x average latency (seconds)

Example: 10,000 QPS x 0.050s (50ms) = 500 in-flight requests
```

Use this to size connection pools, thread pools, and server instances.

---

## 1.5 Storage metrics

| Unit | Size | Memory hook |
|------|------|-------------|
| **Byte (B)** | 8 bits | One character |
| **KB** | 1,024 B | Small JSON |
| **MB** | 1,024 KB | Photo thumbnail |
| **GB** | 1,024 MB | High-res photo, short video |
| **TB** | 1,024 GB | DB shard, daily logs |
| **PB** | 1,024 TB | YouTube-scale archive |

**Use decimal (1e9) in interviews** unless asked otherwise — easier on whiteboard:

```
1 GB ≈ 10^9 bytes
1 TB ≈ 10^12 bytes
1 PB ≈ 10^15 bytes
```

| Term | Meaning |
|------|---------|
| **Hot storage** | SSD, low latency, expensive — recent data |
| **Cold storage** | S3 Glacier, tape — archive, cheap |
| **Retention** | How long you keep data — drives total storage |
| **Replication factor** | 3x for durability — multiply storage by 3 |

**Storage formula:**

```
Total storage = records_per_day x record_size x retention_days x replication_factor

5-year storage = daily_new_data x 365 x 5
```

---

## 1.6 Bandwidth metrics

| Unit | Meaning |
|------|---------|
| **bps** | Bits per second (network capacity) |
| **Bps** | Bytes per second (watch capitalization) |
| **Mbps / Gbps / Tbps** | Megabit / Gigabit / Terabit per second |

**Critical:** Storage uses **bytes**; network links are often quoted in **bits**.

```
Bytes to bits: multiply by 8
Gbps to GB/s: divide by 8

Example: 1 Gbps link ≈ 125 MB/s ≈ 0.125 GB/s
```

**Bandwidth formula:**

```
Bandwidth = QPS x payload_size (bytes) x 8 (to bits)

Egress (CDN) = views_per_day x average_object_size / 86400 x 8
```

| Direction | Meaning | Example |
|-----------|---------|---------|
| **Ingress** | Upload into your system | Video upload, photo POST |
| **Egress** | Download out to users | CDN, feed images, streaming |
| **Cross-region** | Between your data centers | Replication, federation |

---

## 1.7 Availability and reliability

| Availability | Downtime per year | Use case |
|--------------|-------------------|----------|
| 99% | 3.65 days | Internal tools |
| 99.9% | 8.76 hours | Standard SaaS |
| 99.99% | 52.6 minutes | Payments, messaging |
| 99.999% | 5.26 minutes | Telco, core infra |

**Error budget:** If SLO is 99.9% and you used half on bad deploys, slow down releases.

---

## 1.8 Other common terms

| Term | Meaning |
|------|---------|
| **Fan-out** | One write triggers N reads/writes (post to followers) |
| **Cardinality** | Number of unique values (high cardinality = many unique keys — bad for indexes) |
| **Working set** | Data actively accessed — fits in cache? |
| **Skew / hot key** | One key gets most traffic (celebrity post, viral video) |
| **Back-of-envelope** | Rough calculation to nearest order of magnitude |
| **Headroom** | Capacity above peak (plan 2x–3x for spikes) |

---

# Part 2 — The Six Core Formulas (Memorize)

## Formula 1 — Average QPS from daily volume

```
QPS_avg = events_per_day / 86,400

Peak QPS = QPS_avg x peak_factor    (typical peak_factor = 2 to 5)
```

**Example:** 1 billion messages/day

```
QPS_avg = 1e9 / 86400 ≈ 11,600 messages/sec
Peak (3x) ≈ 35,000 msg/sec
```

---

## Formula 2 — Storage

```
Daily new bytes = events_per_day x bytes_per_event
Total (retention R days) = daily_new x R x replication

With growth: use yearly sum or round daily x 365 x years
```

**Example:** 100M photos/day, 2 MB average, 5 years, 3x replication

```
Daily = 100M x 2 MB = 200 TB/day new logical
5 years = 200 TB x 365 x 5 ≈ 365 PB logical (before compression)
With 3x replication ≈ 1 EB raw (say "hundreds of PB" in interview)
Compression (JPEG) often 5–10x — mention as follow-up
```

---

## Formula 3 — Bandwidth (egress)

```
Avg egress Gbps = (daily_bytes_egress x 8) / 86400 / 1e9

Or: concurrent_streams x bitrate_per_stream
```

**Example:** YouTube-like — 1B hours watched/day, 5 Mbps average bitrate

```
Bits/day = 1e9 hours x 3600 s x 5 Mbps
         = 1e9 x 3600 x 5e6 bits ≈ 1.8e19 bits/day
Gbps avg = 1.8e19 / 86400 / 1e9 ≈ 200,000 Gbps = 200 Tbps
(Say "low hundreds of Tbps" — matches CDN-scale thinking)
```

---

## Formula 4 — Little's Law (concurrency)

```
N = lambda x W

N = concurrent in-flight requests
lambda = arrival rate (QPS)
W = average time in system (seconds)
```

**Example:** Search service 50K QPS, 80ms latency

```
N = 50000 x 0.08 = 4000 concurrent requests
If each server handles 500 concurrent → ~8 servers minimum (+ headroom)
```

---

## Formula 5 — Fan-out write amplification

```
Write QPS to feed/cache = posts_per_second x average_followers

Celebrity post: 1 post x 100M followers = 100M writes (push model problem)
Hybrid: push if followers < 10K, pull for celebrities
```

---

## Formula 6 — Database connections / shards

```
Shards needed = total_data / max_per_shard
              (e.g. 500 GB per shard → 10 TB / 500 GB = 20 shards)

Connections = QPS x avg_query_time / queries_per_connection
            (or pool size from Little's Law per service)
```

---

# Part 3 — Units Cheat Sheet (Whiteboard)

```
TIME
  1 day = 86,400 seconds
  1 month ≈ 2.6 x 10^6 seconds (rough)
  1 year ≈ 3.15 x 10^7 seconds

THROUGHPUT
  1 KQPS = 1,000 requests/sec
  86.4M requests/day ≈ 1 KQPS

DATA SIZE (decimal, interview-friendly)
  1 KB = 10^3 B
  1 MB = 10^6 B
  1 GB = 10^9 B
  1 TB = 10^12 B
  1 PB = 10^15 B

NETWORK
  1 byte = 8 bits
  1 Gbps = 10^9 bits/sec ≈ 125 MB/sec
  1 Tbps = 10^3 Gbps

LATENCY (typical)
  L1 cache: nanoseconds
  SSD read: 0.1 ms
  Same-region DB: 1–5 ms
  Cross-region: 50–150 ms
  Human perception: 100 ms feels instant; 1 s feels slow
```

---

# Part 4 — Which System Uses Which Metrics

Use this table before opening a case study guide — know what to estimate first.

| System | Primary metrics | Secondary | Typical peak factor |
|--------|-----------------|-----------|---------------------|
| **Instagram** | DAU, feed read QPS, post write QPS, photo size, CDN egress | Fan-out, story TTL | 3x |
| **TikTok** | DAU, FYP read QPS, watch events/sec, video bitrate, transcode CPU | Recommendation latency | 3x |
| **LinkedIn** | DAU, feed QPS, graph query latency, connection degree | Search QPS | 2–3x |
| **WhatsApp** | DAU, messages/sec, connection count (WebSocket), E2E storage | Media upload size | 2x |
| **Discord** | Concurrent users, messages/sec, voice SFU bandwidth | Gateway shards | 2–3x |
| **Zoom** | Concurrent meetings, video bitrate x participants, SFU CPU | Signaling QPS | 2x (scheduled peaks) |
| **Uber** | Active drivers/riders, location updates/sec, match QPS, geospatial index | Surge multipliers | 3–5x (New Year) |
| **Airbnb** | Search QPS, booking TPS, calendar read QPS | Photo CDN | 3x |
| **Ticketmaster** | Queue poll QPS, purchase TPS, inventory lock TPS | Flash sale 100x | 10–100x on sale |
| **Dropbox** | Sync delta size, block dedup ratio, metadata QPS | Egress on restore | 2x |
| **YouTube** | Watch hours/day, upload ingress, transcode queue, CDN Tbps | View count writes | 2–3x |
| **Netflix** | Concurrent streams, segment QPS, Open Connect fill rate, TTFF/rebuffer SLO | Studio ingest (low vs playback) | 4x (prime time) |
| **Google Search** | Queries/sec, crawl pages/day, index size PB | Autocomplete QPS 5x search | 5–10x news events |
| **AI Rec System** | Inference QPS, feature lookup QPS, embedding dimension | Training offline | 2x |
| **URL Shortener** | Redirect read QPS, create write QPS, key space | Cache hit ratio | 5x |
| **Distributed Cache** | Key QPS, memory per key, hit ratio | Hot key skew | 3x |
| **LRU Cache** | Capacity entries/bytes, get+put QPS, hit ratio, evictions/sec | Scan thrashing | 2x |
| **Notification System** | Push QPS, device tokens, provider rate limits | Batch vs real-time | 5x (campaigns) |
| **Rate Limiter** | Check QPS per API, Redis memory for windows | Global vs per-user | Peak = burst |
| **Payment Gateway** | TPS, idempotency store, PCI latency p99 | Reconciliation batch | 3x (Black Friday) |
| **Distributed Lock** | Lock acquire QPS, hold time, fencing tokens | Contention hotspots | 2x |
| **Metrics / Datadog** | Ingest events/sec, cardinality, retention | Query QPS | 2x |
| **Elevator / Parking Lot** | Requests/sec per building, state machine transitions | Low scale — focus latency | 1.5x |
| **Kafka / MQ** | Messages/sec, partition count, retention GB | Consumer lag | 2x |
| **K8s / CI/CD** | Deploy frequency, pod count, build minutes | Not user QPS | — |
| **DNS** | Queries/sec globally, TTL, propagation delay | DDoS 100x | 10x |
| **CDN / S3** | Egress Gbps, object size, request rate | Origin shield | 3x |

---

# Part 5 — Worked Examples (Say These Aloud)

## 5.1 Instagram-scale (photo + feed)

**Assumptions:**

```
500M DAU, 1B MAU
Each DAU: 5 feed refreshes/day, 2 posts/week ≈ 0.3 posts/day average
Average photo: 2 MB stored; 200 KB CDN variant served per view
200 followers average; hybrid fan-out
```

**Feed read QPS:**

```
Feed reads/day = 500M x 5 = 2.5B
QPS_avg = 2.5e9 / 86400 ≈ 29,000
Peak (3x) ≈ 87,000 QPS
```

**Post write QPS:**

```
Posts/day = 500M x 0.3 = 150M posts/day
QPS_avg ≈ 1,700 posts/sec
```

**CDN egress (rough):**

```
Views/day ≈ 2.5B feed loads x 10 images x 200 KB = 5 PB/day served
Avg Gbps ≈ (5e15 x 8) / 86400 / 1e9 ≈ 460 Gbps (order of magnitude: hundreds of Gbps)
```

**Interview line:** "Half-billion DAU, low tens of thousands feed QPS peak, low thousands write QPS, CDN egress in hundreds of Gbps — cache and CDN are the story."

---

## 5.2 WhatsApp-scale (messaging)

**Assumptions:**

```
2B users, 500M DAU (example)
50 messages sent per DAU per day
Average message 1 KB (text); 10% media 100 KB
```

**Messages/sec:**

```
Messages/day = 500M x 50 = 25B
QPS_avg = 25e9 / 86400 ≈ 290,000 msg/sec
Peak (2x) ≈ 580,000 msg/sec
```

**Storage (5 years, text-heavy):**

```
Avg size ≈ 0.9 x 1KB + 0.1 x 100KB ≈ 11 KB/message
Daily = 25B x 11 KB ≈ 275 TB/day
5 years ≈ 500 PB logical (before replication; mention sharding by user)
```

**Metrics that matter:** messages/sec, concurrent connections, delivery latency p99, not CDN egress.

---

## 5.3 Uber-scale (location + matching)

**Assumptions:**

```
100M DAU riders + drivers combined
Active trip: 10M concurrent at peak
Location update every 4 seconds while active
3M active drivers updating location
```

**Location write QPS:**

```
Updates/sec = 3M / 4 = 750,000 location writes/sec
(Geospatial index + Kafka — not one SQL row per ping in hot path)
```

**Match / request QPS:**

```
10M rides/day → 10e6 / 86400 ≈ 115 TPS ride requests
Search nearby drivers: 10x reads per request → ~1,150 QPS geo queries
```

**Metrics that matter:** location EPS, match latency, geohash/H3 cell size, surge peak multiplier.

---

## 5.4 YouTube-scale (video)

**Assumptions:**

```
800M DAU, 1B hours watched per day
Average watch bitrate 3 Mbps
500 hours uploaded per minute
```

**Streaming egress:**

```
Bits/day = 1e9 hours x 3600 x 3 Mbps ≈ 1.08e19 bits
Gbps ≈ 1.08e19 / 86400 / 1e9 ≈ 125,000 Gbps ≈ 125 Tbps average
```

**Upload ingress:**

```
500 hr/min x 60 x 24 = 720,000 hours/day uploaded
At 8 Mbps source ≈ 720000 x 3600 x 8e6 / 86400 / 1e9 ≈ 240 Gbps ingress
```

**Transcode storage:** Each hour → 10 renditions — multiply upload storage by 10–20x for processed assets.

**Metrics that matter:** Tbps CDN, transcode queue depth, TTFF, view count EPS (async).

---

## 5.4b Netflix-scale (subscription streaming)

**Assumptions:**

```
200M DAU, 280M subscribers
90 min avg watch/user/day
Peak concurrent streams: 30% of DAU evening ≈ 60M
Average ABR bitrate: 3 Mbps
Open Connect serves ~95% of segments from ISP-local appliances
```

**Streaming egress (theoretical peak):**

```
60M streams x 3 Mbps = 180 Tbps peak
Origin egress (5% miss): ~9 Tbps
```

**Session starts:**

```
400M sessions/day → ~4,600 QPS avg; peak ~18,500 QPS (4x prime time)
Segment requests: 60M / 3 sec ≈ 20M segment QPS at peak (edge-local)
```

**Play heartbeats:**

```
Every 30 sec → 60M / 30 ≈ 2M events/sec peak (Kafka → Cassandra watch history)
```

**Metrics that matter:** TTFF p95, rebuffer ratio, OC catalog fill %, concurrent stream enforcement — not upload QPS.

---

## 5.5 URL shortener

**Assumptions:**

```
100:1 read:write ratio
100M new URLs/day
10-year retention for URLs
500 bytes metadata per URL
```

**QPS:**

```
Creates = 100M / 86400 ≈ 1,160 write QPS
Reads = 116,000 read QPS avg; peak 500K
```

**Storage:**

```
100M x 500 B x 365 x 10 ≈ 180 TB metadata (tiny vs social media)
Keys: 100M/day x 365 x 10 ≈ 365B URLs — 62^7 base62 ≈ 3.5e13 (need 8+ char keys)
```

**Metrics that matter:** read QPS, cache hit ratio (99%+), redirect latency p99.

---

## 5.6 Ticketmaster flash sale

**Assumptions:**

```
1M tickets, on-sale moment
10M users in virtual queue polling every 30s
100K checkout TPS attempted; actual purchase 1K TPS (inventory limited)
```

**Queue poll QPS:**

```
10M / 30 ≈ 333,000 QPS polling
Must edge-cache + WebSocket push to reduce
```

**Purchase TPS:**

```
Successful TPS ~ 1,000 — but **atomic inventory** is bottleneck, not average QPS
Peak multiplier 100x+ vs normal — design for spike, not average
```

**Metrics that matter:** TPS on inventory row, queue depth, hold TTL (5–10 min).

---

## 5.7 Google Search

**Assumptions:**

```
8B searches/day, 500K QPS peak
500 bytes per query log; 100 KB average fetched page (off critical path)
100B pages indexed, 50 KB average document
```

**Search QPS:**

```
Avg = 8e9 / 86400 ≈ 92,600 QPS
Peak ≈ 500,000 QPS (5x avg)
Autocomplete: 5x keystrokes → 2.5M peak QPS (separate tier)
```

**Index storage:**

```
100B x 50 KB ≈ 5 PB inverted index (compressed; say single-digit PB)
Crawl: billions of pages/day — bandwidth and politeness, not user QPS
```

---

# Part 6 — Latency Budget (How to Decompose)

When interviewer asks "feed in 200ms", allocate:

| Component | Budget | Notes |
|-----------|--------|-------|
| Client to edge | 20–50 ms | Mobile network variable |
| API gateway + auth | 5–10 ms | JWT verify, rate limit |
| Feed service | 30–50 ms | Cache hit path |
| Graph / rank | 20–40 ms | Precomputed helps |
| CDN media | 50–100 ms | Parallel with JSON |
| **Total** | ~200 ms | Parallel calls save wall time |

**Rule:** Latencies **add in series**, **max in parallel**.

```
Serial: 50 + 50 + 50 = 150 ms
Parallel: max(50, 50, 50) = 50 ms
```

---

# Part 7 — When to Use Which Calculation

| If interviewer asks about... | Calculate... | Units |
|------------------------------|--------------|-------|
| "Can it handle the traffic?" | QPS avg + peak | req/sec |
| "Database size in 5 years?" | rows/day x row size x retention | TB, PB |
| "CDN cost / capacity?" | egress bytes/day | Gbps, Tbps |
| "Upload pipeline?" | ingress bandwidth + transcode queue | Gbps, jobs/hour |
| "How many servers?" | Little's Law + CPU per request | instances |
| "Cache size?" | working set x object size | GB, TB |
| "Kafka partitions?" | peak EPS / per-partition throughput | partitions |
| "Payment system?" | TPS + p99 latency + idempotency TTL | TPS, ms |
| "Chat?" | messages/sec + connections | msg/sec, million WS |
| "Search index?" | corpus size x expansion factor | PB |
| "Black Friday / on-sale?" | peak multiplier on TPS | 10x–100x spike |

---

# Part 8 — Common Mistakes (Avoid in Interviews)

| Mistake | Fix |
|---------|-----|
| Confusing bits and bytes | Network = bits; storage = bytes; x8 when converting |
| Using MAU for daily QPS without converting | Use DAU or divide monthly actions by 30 |
| Forgetting peak multiplier | Average QPS alone undersizes by 3x–5x |
| Ignoring fan-out | One post != one write |
| Exact false precision | Round: "about 30K QPS" not "29,847.3" |
| One DB for all writes | Shard when > few TB or > 10K write TPS |
| Skipping cache hit ratio | 90% cache hit = 10x less DB load |
| Same latency for read and write | Writes often async; state p99 for reads |

---

# Part 9 — Interview Script (Template)

Say this structure while writing on whiteboard:

```
1. "Let me state assumptions: X DAU, Y actions per user per day."

2. "Daily volume = X x Y = Z events per day."

3. "Average QPS = Z / 86,400 ≈ [round number]."

4. "Peak is typically 3x average → about [N] QPS at peak."

5. "Storage: Z events x [size] x [retention] ≈ [TB/PB] — 
    I'll assume replication factor 3 for durability."

6. "Bandwidth: for [read-heavy/media] system, egress dominates — 
    roughly [Gbps/Tbps] order of magnitude."

7. "This implies [cache / CDN / sharding / async queue] — 
    which feeds into the architecture."
```

---

# Part 10 — Quick Reference Card

```
METRICS
  MAU  = monthly active users
  DAU  = daily active users
  QPS  = queries (requests) per second  = daily / 86400
  TPS  = transactions per second (often writes)
  EPS  = events per second (analytics, Kafka)
  Stickiness = DAU / MAU

FORMULAS
  QPS_avg = per_day / 86400
  QPS_peak = QPS_avg x (2 to 5)   [sales: 10 to 100]
  Storage = daily_new x retention x replication
  Bandwidth (Gbps) = bytes/sec x 8 / 1e9
  Concurrency = QPS x latency_seconds

UNITS
  1 TB = 10^12 bytes
  1 Gbps ≈ 125 MB/s
  p99 latency = tail user experience

BY SYSTEM TYPE
  Social feed   → read QPS, fan-out, CDN egress
  Messaging     → msg/sec, connections, storage
  Video         → Tbps egress, transcode, TTFF
  Marketplace   → search QPS, booking TPS, geo
  Search        → query QPS, index PB, crawl
  Payments      → TPS, p99, idempotency
  Flash sale    → peak TPS, inventory atomicity
```

---

# Part 11 — Map to Case Study Guides

Each guide's **Section 3 (Capacity Estimation)** applies this framework with system-specific numbers:

| Track | Guide | Key capacity focus |
|-------|-------|-------------------|
| 1 | Instagram, TikTok, LinkedIn | DAU, feed QPS, CDN, fan-out |
| 2 | WhatsApp, Discord, Zoom | msg/sec, WS connections, SFU Gbps |
| 3 | Uber, Airbnb, Ticketmaster | geo EPS, search QPS, spike TPS |
| 4 | Dropbox, YouTube, Netflix | sync bandwidth, Tbps, Open Connect, transcode |
| 5 | Google Search, AI Rec | query QPS, index PB, inference QPS |
| 6 | URL Shortener, Cache, LRU Cache, Notifications, Rate Limiter, Payment, Lock, Metrics | specialized QPS/TPS/cardinality |
| 7 | Elevator, Parking Lot | low QPS, latency, correctness |
| 8 | Scaling, DB, Kafka, RabbitMQ, MQ, Network, DNS | fundamentals behind the numbers |
| 9 | K8s, Cloud, CI/CD, Observability, API Gateway, CDN | infra capacity and SLOs |

Open the case study after reading the relevant row here — the numbers will not feel arbitrary.

---

*Practice: Pick a random guide, cover the metric names, and re-derive QPS and storage in under 5 minutes without opening the answer.*
