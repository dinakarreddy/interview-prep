# HLD Interview Prep — Staff SWE @ FAANG (product-heavy)

A self-contained set of high-level-design interview prep notes, written transcript-style as if simulating real interviews. 12 canonical problems plus a shared framework, calibrated for the **Staff Software Engineer** bar at FAANG-class product companies.

> **Total**: ~13,700 lines of prep across 13 files. Each problem doc is 1,000–1,300 lines and follows an identical 19-section structure so you can drill the same workflow on every problem.

---

## How to use this folder

> **Day-of-interview:** read `00-INTERVIEW-TEMPLATE.md` 30 minutes before the interview. It's the tactical playbook — opening line, time-budget, phrases for awkward moments, wrap-up speech template.
>
> **For studying:** start with `00-framework.md` (conceptual), then per-problem files. Recommended order below.

1. **Read `00-framework.md` first.** Internalize the RESHADED template, the Senior-vs-Staff calibration delta, the pattern library, and the wrap-up speech format. None of the per-problem files repeat this material.
2. **Pick a problem.** Recommended order is below.
3. **Don't read the problem doc cover-to-cover first.** Read only Section 0 (the problem statement), close the file, and try to design it yourself for 45 minutes. Time-box yourself per the framework.
4. **Then open the file** and read the full document. Compare your approach to the script. Note where the script went somewhere you didn't.
5. **Re-do the problem 24 hours later from scratch.** Repetition is what builds muscle memory.
6. **After 4–5 problems**, you should notice the same patterns appearing — fanout strategies, geo-indexing, hot-key mitigation, exactly-once, cellular isolation. The goal is not to memorize 12 designs; it's to internalize the *shape* of how a Staff candidate attacks any HLD problem.

---

## The 19-section structure (every problem)

Every problem doc follows this exact structure so the workflow becomes muscle memory:

| § | Section |
|---|---|
| 0 | Problem statement (interviewer drops it vaguely) |
| 1 | Clarifying questions (Q&A dialog) |
| 2 | Functional requirements (P0/P1/P2 + out of scope) |
| 3 | Non-functional requirements (with concrete numbers) |
| 4 | Capacity estimation (DAU → QPS → storage → bandwidth, arithmetic shown) |
| 5 | API design |
| 6 | Data model (with sharding keys) |
| 7 | High-level architecture (ASCII diagram + request walk-through) |
| 8 | Deep dives (3–7 component-level dives) |
| 9 | Tradeoffs & alternatives (tables) |
| 10 | Defending architectural choices — interviewer attacks (dialog) |
| 11 | Failure modes & mitigations (table) |
| 12 | Productionization checklist |
| 13 | Monitoring & metrics (4 layers) |
| 14 | Security & privacy |
| 15 | Cost analysis |
| 16 | Open questions / what I'd validate with PM |
| 17 | Staff-level scorecard for this problem |
| 18 | Wrap-up speech (template paragraph) |
| 19 | **Bonus**: market-based isolation / cellular architecture |

---

## Files

### Framework + interview-day playbook
| File | What it covers | When to read |
|---|---|---|
| [`00-INTERVIEW-TEMPLATE.md`](./00-INTERVIEW-TEMPLATE.md) | **Tactical, in-the-moment playbook.** 60-min clock, whiteboard layout, verbatim opening line, clarifying-question shortlist, mental estimation worksheet, phrases for awkward moments, wrap-up speech template, pre-interview checklist, post-interview debrief, emergency cheat sheet. | **30 minutes before every interview.** Re-read every time. |
| [`00-framework.md`](./00-framework.md) | **Conceptual underpinning.** RESHADED template, Senior-vs-Staff calibration delta, pattern library (sharding, caching, queueing, consistency, DBs, geo-indexing, real-time delivery), interviewer-attack dialogues, productionization checklist, 4-layer monitoring guide, anti-patterns, wrap-up speech format. | Once during prep, then re-skim weekly. |

### The 12 problems

| # | File | Domain | Central insight |
|---|------|--------|------------------|
| 1 | [`01-news-feed.md`](./01-news-feed.md) | News Feed (Twitter / Instagram / Facebook) | **Hybrid push/pull fanout** with celebrity threshold |
| 2 | [`02-whatsapp.md`](./02-whatsapp.md) | WhatsApp / Messenger | Stateful WS gateway + per-device fanout + Signal Protocol |
| 3 | [`03-ride-dispatch.md`](./03-ride-dispatch.md) | Uber / DoorDash dispatch | H3 geo-indexing + per-city cell + window-batched matching |
| 4 | [`04-youtube-netflix.md`](./04-youtube-netflix.md) | YouTube / Netflix | Transcoding farm + multi-tier CDN + ABR + popularity-driven storage tiering |
| 5 | [`05-dropbox-drive.md`](./05-dropbox-drive.md) | Dropbox / Google Drive | Content-addressed dedup + delta sync + conflicted-copy resolution |
| 6 | [`06-slack-discord.md`](./06-slack-discord.md) | Slack / Discord | Workspace-as-cell + WS gateway + channel-as-pubsub-topic |
| 7 | [`07-rate-limiter.md`](./07-rate-limiter.md) | Distributed Rate Limiter | Algorithm tradeoffs + sloppy counters + fail-open default |
| 8 | [`08-ticketmaster.md`](./08-ticketmaster.md) | Ticketmaster / Flash-sale | Redis-atomic-DECR inventory + virtual waiting room + per-event cell |
| 9 | [`09-tiktok-feed.md`](./09-tiktok-feed.md) | TikTok For-You feed | Two-stage retrieval (vector ANN → ranking) + feature store + multi-objective ranking |
| 10 | [`10-notification-system.md`](./10-notification-system.md) | Push notifications at scale | Per-channel queues + idempotent dedup + timezone-bucketed scheduling |
| 11 | [`11-ad-click-aggregation.md`](./11-ad-click-aggregation.md) | Real-time ad click aggregation | Kafka + Flink with watermarks + Druid serving + S3 reconciliation |
| 12 | [`12-live-comments.md`](./12-live-comments.md) | Live streaming chat (Twitch / YouTube Live) | Server-side sampling + hierarchical (tree) fanout + sticky-by-stream sharding |

---

## Recommended study order

### If you have **1 week**
Hit the canonical four — they teach the most patterns:
- Day 1: `01-news-feed` (the canonical Staff warm-up; teaches fanout, hot keys, cellular)
- Day 2: `02-whatsapp` (real-time, persistent connections, ordering, multi-device)
- Day 3: `03-ride-dispatch` (geo-indexing, real-time, per-city cells)
- Day 4: `08-ticketmaster` (high-contention inventory; very different shape)
- Day 5: `07-rate-limiter` (infrastructure problem, algorithm tradeoffs)
- Days 6–7: pick whichever feels weakest, redo from scratch

### If you have **2–3 weeks**
Do all 12 in this order — grouped by family so patterns reinforce:

**Week 1 — Social & messaging (fanout-heavy)**
1. `01-news-feed` — push/pull tradeoff
2. `02-whatsapp` — real-time delivery
3. `06-slack-discord` — workspace multi-tenancy
4. `12-live-comments` — extreme fanout

**Week 2 — Media, geo, & marketplaces**
5. `04-youtube-netflix` — content delivery
6. `03-ride-dispatch` — geo + matching
7. `05-dropbox-drive` — sync + dedup
8. `08-ticketmaster` — high-contention inventory

**Week 3 — Recommendations, infra, & data**
9. `09-tiktok-feed` — recommendation
10. `10-notification-system` — multi-channel ops
11. `07-rate-limiter` — distributed counters
12. `11-ad-click-aggregation` — streaming pipelines

### If you have **only a few days**
Read `00-framework.md` thoroughly, then `01-news-feed.md` cover-to-cover. The framework + one fully-internalized example will get you further than skimming all 12.

---

## What separates Staff from Senior on these problems

(Pulled from `00-framework.md`, Section 3 — the most important section. Read it.)

A Staff candidate, vs. a Senior, will:
- **Identify the 1–2 hardest sub-problems** before drawing, and tell the interviewer they're the focus
- **Use estimation numbers to drive design choices**, not just compute them
- **Volunteer tradeoffs** with the chosen alternative explicitly named
- **Have a 100x scale story**, not just a 10x story
- **Quantify cost** ($/DAU or $/req) and use it as a tradeoff lever
- **Build graceful degradation into the design** — what does the system do when a dependency is down?
- **Bring up cellular / market-based isolation** unprompted, even if just in the wrap-up
- **Close with a wrap-up speech** that restates the dominant constraints, the chosen architecture, the production worries, and the next deep-dive they'd want

If the interviewer can't tell you what your highest-conviction architectural decision was at the end of the round, you scored Senior.

---

## A note on cellular architecture (Section 19 of every problem)

Every problem doc closes with a "market-based isolation" section. This is a Staff-level topic at FAANG product-heavy companies — every meaningful product at scale eventually goes cellular for compliance, blast-radius, and operational reasons.

The doc treats it as a "if time permits" bonus, but in practice you should **bring it up unprompted** in the last 5 minutes. It's a recurring pattern across every problem in this folder, just with different cell boundaries:

| Problem | Natural cell boundary |
|---|---|
| News feed | Geographic region (US, EU, IN) |
| WhatsApp | Geographic region |
| Ride dispatch | **City / metro** |
| YouTube/Netflix | Geographic region (also content rights) |
| Dropbox | Geographic region |
| Slack | **Workspace** |
| Rate limiter | Per-region for routing; per-tenant for quota |
| Ticketmaster | **Per-event for mega-events**, regional otherwise |
| TikTok | Country (especially Douyin / TikTok split) |
| Notifications | Geographic region |
| Ad click | Geographic region |
| Live comments | **Per-event for mega-events**, regional otherwise |

Notice the pattern: the cell boundary is whatever makes a chunk of work fully **independent**. A driver in NYC will never serve a rider in SF — so cell = city. A Slack user in Workspace A will never DM a user in Workspace B's general channel — so cell = workspace. The boundary chooses itself if you ask "where does the work naturally not cross?"

---

## Authoring notes

- Files were written to be opinionated. They commit to specific databases, queues, and consistency models, and **defend** those choices when an interviewer attacks. Don't use them as buffet — use them as a baseline you adjust based on your own context and the constraints in your interview.
- Every numeric estimate in the docs is a rounded back-of-envelope figure intended to drive a design decision, not a benchmark. In a real interview, **show your arithmetic** — the interviewer cares less about the number than your thought process.
- The transcript dialog format is a teaching device. In a real interview, you'll be the only one talking through most of this — the interviewer interjects mostly to push back. Treat the dialog as the ideal flow, not a script to memorize.

---

## What's NOT in this folder

These problems weren't included but are common Staff-level alternatives. Patterns from the included 12 cover them — but if you have time, work them up:

- **Distributed Job Scheduler** (Airflow / Cron at scale) — leverages 07-rate-limiter and 11-ad-click patterns
- **Distributed Cache** (Memcached / Redis-as-a-service) — extension of 07-rate-limiter
- **Web Crawler** — extension of 11-ad-click pipeline patterns
- **Yelp / Nearby Places** — leverages 03-ride-dispatch geo patterns
- **URL Shortener** (bit.ly) — too small for Staff but a classic warm-up
- **Stock Exchange / Trading** — niche unless interviewing in fintech
- **Distributed Metrics / Monitoring (Datadog-class)** — extension of 11-ad-click pipeline patterns
- **Code Deployment System (Jenkins-class)** — niche unless interviewing for DevTools/Platform
- **Payment System (Stripe-class)** — extension of 08-ticketmaster idempotency + 11-ad-click reconciliation
- **Spotify-style music streaming** — extension of 04-youtube + 09-tiktok recommendation

---

Good luck. The bar is high but the patterns repeat. Master the framework + 4 problems and the rest are remixes.
