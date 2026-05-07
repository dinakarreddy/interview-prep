# 00 — HLD Interview Template (the playbook for interview day)

> **This is the tactical template.** Read it 30 minutes before walking in. Refer to it mentally during the interview. For the *conceptual* foundation (patterns, calibration delta, pattern library) see `00-framework.md`. This file tells you what to *do*, second-by-second.

---

## How to use this file

- Read it **fully once** at least 24 hours before the interview.
- Re-read **sections 1, 3, 8, 9** in the 30 minutes before the interview.
- During the interview, you should not be thinking about *process* — you should be thinking about the *problem*. This template's job is to make process automatic.

---

## 1. The 60-minute clock

If the interview is 45 minutes, compress everything proportionally — but the *order* never changes.

```
0:00 ─┬─ 0:01  Opening line. State your plan.
       │
0:01 ─┼─ 0:08  CLARIFY (5–7 min)
       │       Ask 4–6 questions. Listen. Take notes top-left.
       │
0:08 ─┼─ 0:11  REQUIREMENTS (2–3 min)
       │       Functional + non-functional. Numbers, not adjectives.
       │
0:11 ─┼─ 0:16  ESTIMATE (3–5 min)
       │       DAU → QPS → storage → bandwidth. Show arithmetic.
       │       Use one number to drive one design decision aloud.
       │
0:16 ─┼─ 0:23  API + DATA MODEL (5–7 min)
       │       Public surface only. Pick partition keys with intent.
       │
0:23 ─┼─ 0:33  HIGH-LEVEL ARCHITECTURE (8–10 min)
       │       Boxes + arrows. Walk request path end-to-end.
       │       Pause at end: "Should I deep-dive A or B?"
       │
0:33 ─┼─ 0:53  DEEP DIVE (15–20 min)  ← Staff calibration happens here
       │       Pick 1–2 components. Go to the next layer of detail.
       │       Expect heavy interviewer pushback — this is the point.
       │
0:53 ─┼─ 0:58  WRAP (5 min)
       │       Failure modes. Productionization. Monitoring. Cost.
       │       Volunteer cellular if there's time.
       │
0:58 ─┴─ 1:00  CLOSE
               The wrap-up speech. Then "what would you like to dig into?"
```

### Hard rules

- **If you're 20 min in and haven't drawn a box, you're losing.** Cut clarifying short.
- **If you're 35 min in and still drawing boxes (no deep dive), you're losing.** Force a deep dive.
- **If you're 55 min in and haven't said "in production we'd…"**, you're losing. Pivot to wrap.
- **Never let the interviewer have to ask you to move on.** Self-pace.

---

## 2. Whiteboard layout

Plan the physical space *before* you write anything. Whether it's a real whiteboard, a virtual one (Excalidraw, Miro), or just paper — this layout works.

```
┌─────────────────────────────────────────────────────────────────┐
│  REQUIREMENTS                       │  CAPACITY                  │
│  ─────────────                      │  ─────────                 │
│  Functional:                         │  DAU: ____                │
│   • _____                            │  QPS: ____                │
│   • _____                            │  Storage: ____            │
│   • _____                            │  Bandwidth: ____          │
│  NFR:                                │                            │
│   • Latency p99: ____                │  Read:Write ratio: ____   │
│   • Availability: ____               │                            │
│   • Consistency: ____                │                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│            HIGH-LEVEL ARCHITECTURE  (the big diagram)            │
│                                                                   │
│            [Client] → [CDN] → [Gateway] → [Services]              │
│                                          ↘  ↗                     │
│                                       [Cache] [DB]                │
│                                          ↑                         │
│                                       [Queue]                      │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│  DEEP DIVE: ___________              │  TRADEOFFS                 │
│  ──────────────────────              │  ─────────                 │
│  [scratch space — sequence            │  • Choice A vs B          │
│   diagrams, schemas, math]            │  • Pro: ___               │
│                                        │  • Con: ___               │
│                                        │  • Pick: ___              │
└─────────────────────────────────────────────────────────────────┘
```

- **Top-left:** requirements stay visible the whole interview. Glance back.
- **Top-right:** capacity numbers — referenced when justifying choices ("at 350K QPS, we can't…")
- **Middle:** the *one* big architecture diagram. Don't redraw — annotate.
- **Bottom-left:** deep-dive scratch space. Wipe and reuse per dive.
- **Bottom-right:** running tradeoff log. Helps the wrap-up speech write itself.

### Drawing rules
- **Top-left → bottom-right** request flow.
- **Numbered arrows** (1, 2, 3 …) so you can refer back ("at step 4, …").
- **Consistent shapes**: rectangle = service, cylinder = DB, parallelogram = queue, cloud = external, stick figure = user.
- **Don't draw before scoping.** First 10 min are pen-down.

---

## 3. The opening 60 seconds (verbatim — memorize)

> *"Thanks. Before I design, let me make sure I understand the problem. I'm going to spend the first ~5 minutes on clarifying questions and scoping, then state the requirements I'm designing for, do a quick capacity estimation, sketch the high-level architecture, and then I'd like to deep-dive on the 1–2 hardest components. I'll save the last 5 minutes for failure modes, productionization, and monitoring. Stop me at any point if you want me to go a different direction. Sound good?"*

### Why this works
- Establishes you have a process (calibrates Staff signal immediately)
- Gets explicit buy-in on the order — interviewer can redirect if they have other priorities
- Pre-commits you to deep-diving (you've now publicly promised to)
- Creates a "stop me" channel — interviewer feels invited to interrupt productively

---

## 4. Clarifying questions — the universal shortlist

You ask **4–6** of these. Pick based on the problem. Never fire all of them — that's a checklist, not a conversation.

| # | Question | What you're listening for |
|---|---|---|
| 1 | "What's the rough scale — DAU and key actions per user per day?" | Drives every later number. If they hand-wave, propose: "Let's assume 200M DAU." |
| 2 | "Is this read-heavy, write-heavy, or balanced?" | Drives DB choice, caching strategy. ~100:1 R:W → cache everything. |
| 3 | "What's the latency target for the primary user-facing operation?" | Numbers. p99 < 200ms vs < 1s changes the design. |
| 4 | "What features are in scope for this discussion?" | Force commitment. Lists out: feature A, feature B — and explicitly *excludes* feature C. |
| 5 | "Is this global / multi-region, or a single market?" | Flag for cellular discussion later. |
| 6 | "What's the consistency requirement — strong, eventual, read-your-writes?" | Drives DB choice. "Eventual is fine for most, RYW for the user's own actions" is usually right. |
| 7 | "Are there existing systems / constraints I should integrate with?" | They may say "assume greenfield" — note it, ask if they want it considered. |
| 8 | "Are there compliance constraints — GDPR, HIPAA, PCI?" | Load-bearing for cellular section. |

### Anti-questions — DO NOT ask
- ❌ "What database should I use?" — *you* propose, *they* push back
- ❌ "Do you want me to talk about X?" — decide and inform
- ❌ "Is this big enough scale to need sharding?" — estimate, then conclude

### After clarifying — say this
> *"Great. So I'm designing for [restate scope in one sentence]. I'll exclude [list of 1–3 explicit out-of-scope items] for time. If we have time at the end I can come back to those."*

This single sentence shows you can scope, can prioritize, and own the time budget.

---

## 5. Estimation worksheet (mental fill-in-the-blank)

Fill this in DURING clarifying-questions phase so it's done by the time you say "OK let me estimate." Show the arithmetic on the board for at least one row.

```
Users
  DAU:                _____ million
  Total accounts:     _____ million

Primary action (e.g., post / message / view)
  Per user per day:   _____
  Total per day:      _____ million
  Avg per second:     _____ K/s
  Peak (3–5×):        _____ K/s

Reverse action (read / fanout) if asymmetric
  Per user per day:   _____
  Total per day:      _____ million
  Peak per second:    _____ K/s

Storage per primary action
  Avg size:           _____ KB
  Per day:            _____ TB
  Per year (×365):    _____ PB
  With 3× replication: _____ PB

Bandwidth
  Egress avg:         _____ GB/s
  Egress peak:        _____ GB/s
```

### The one number that drives a design decision

State this aloud. *"At [N] writes/second, [single DB option] won't keep up — we need to shard. I'll pick [scheme] and explain why in the data model section."*

This is the moment a Senior becomes a Staff candidate. Estimation is *for* something — it's not arithmetic for its own sake.

### Memorized constants (have these on tap — see `00-framework.md` §5)
- 1 day ≈ 10⁵ s
- Single MySQL primary: ~5K writes/s
- Single Redis: ~100K ops/s
- Single Cassandra node: ~10K writes/s
- Single Kafka broker: ~100K msg/s
- A "good" service host: ~10K QPS
- Same-DC RTT: 0.5ms
- Cross-region RTT: 80–150ms

---

## 6. API + data model — the 5-minute version

You don't have time for a full schema. You need **just enough** to expose your sharding strategy.

### What to write
- 3–6 public endpoints. Use REST verbs, real paths.
- 1–3 primary tables/collections.
- For each primary table: **partition key**, **clustering/sort key**, **secondary index needs**.

### What to say while writing
> *"This table is sharded by [X] because the dominant access pattern is [Y]. The price I pay is [Z lookup pattern is awkward], which I'll address with [secondary index / denormalization / cache]."*

That single sentence — partition-key-with-defended-rationale — separates Senior from Staff. **Never** pick a partition key without saying why.

### What to omit
- Auth flows (assume OAuth/mTLS, mention once)
- Pagination internals (just say "cursor-based")
- Field-level types (only call out where the type is load-bearing — e.g., "Snowflake ID for time-sortability")

---

## 7. Architecture-diagram checklist

Every diagram must have, at minimum:

- [ ] **Client** (mobile / web / browser) on the left
- [ ] **Edge / CDN** for static + cacheable
- [ ] **Load balancer / API gateway** with auth + rate limiting noted
- [ ] **Stateless service tier** (named services, not "microservice")
- [ ] **Cache tier** (Redis/Memcached) — say what's cached, TTL, hit rate target
- [ ] **Primary store** (DB) — named (Cassandra, Spanner, PG, Dynamo) with sharding noted
- [ ] **Async / queue tier** if there's any fanout, batching, or retry logic
- [ ] **External services** (Stripe, S3, APNs) — clouds with names

### Walk the request path aloud — twice
- **Once for read:** "User opens app → CDN check → API gateway auth → Feed Service → Redis lookup → on miss, Cassandra → return."
- **Once for write:** "User posts → Post Service writes Cassandra (CL=QUORUM) → publishes to Kafka → Fanout Service consumes → writes timeline shards → 201 returned to user."

### Anti-pattern to avoid
Do not draw a service and never explain its job. Every box on the board must have a 1-sentence purpose stated aloud.

---

## 8. Deep-dive selection — the 30-second decision

After the architecture diagram is drawn, **stop and check in**:

> *"I see two candidates for the deep dive. The first is [component A] — the technical risk is [X], and it dominates [latency / cost / scale]. The second is [component B] — it's the more product-impactful piece because [Y]. Which would you rather see, or do you want me to pick?"*

### How to pick if they say "you pick"

Pick whichever has the **highest entropy** for an interviewer:
1. **Hot-key / fanout** problems (always rich) — celebrity post fanout, hot tenant in rate limiter, hot stream in live comments
2. **Consistency boundaries** — what's strong, what's eventual, why
3. **State management at scale** — sharding, rebalancing, hot-shard mitigation
4. **Failure & recovery** — what breaks, what's the blast radius, how do we recover

Avoid:
- Pure data modeling (boring)
- Auth flows (everyone knows OAuth)
- Algorithm details unless explicitly asked

### Inside the deep dive — the loop

```
1. Pick a component.
2. State the sub-problem in one sentence.
3. Propose your approach.
4. Volunteer the tradeoff: "this gets us X, the price is Y."
5. Wait for pushback.
6. Either defend or update — name what's changing and why.
7. Move to next sub-problem.
```

---

## 9. Phrases for awkward moments (memorize these verbatim)

### When you don't know
> *"Let me think for 10 seconds."*

(Then actually think for 10 seconds. Silence is fine. Filler talk is not.)

### When the interviewer pushes back and you think they're right
> *"You're right — I missed that. Let me update the design. Specifically I'd change [X] to [Y] because [Z]."*

(Concede explicitly. Concession + reasoning is strength, not weakness.)

### When the interviewer pushes back and you think you're right
> *"Let me think about that — my reasoning was [X]. What's the failure mode you're seeing?"*

(Defend, ask for the specific concern, then either concede or push back further.)

### When the interviewer asks "why X?"
> *"Three reasons: [most important], [second], [third]. The price I pay is [tradeoff]. If [condition Y were different], I'd switch to [alternative]."*

(This pattern is the **Staff defending pattern**. Memorize the structure.)

### When you realize mid-answer you're going wrong
> *"Actually, let me back up — I'm going to revise that. The issue with what I just said is [X]. Here's the better answer: [Y]."*

(Self-correction in real time is a *positive* signal, not a negative one. Confidence ≠ never-wrong.)

### When you're running out of time
> *"I'd like to leave time for failure modes and productionization — let me wrap this dive and pivot."*

(Self-paced time management is a Staff signal.)

### When asked something genuinely outside your knowledge
> *"I don't have direct experience with [X], but here's how I'd reason about it from first principles: [Y]. I'd want to validate [Z] in the actual system before committing."*

(Honesty + reasoning + a verification plan beats faking expertise every time.)

### When stuck on what to say next
> *"Let me check in — does the direction I'm heading match what you wanted to see, or would you rather I explore [alternative]?"*

(Always better to redirect than to fill silence with low-quality content.)

---

## 10. The wrap-up speech (fill-in-the-blank — memorize the structure)

Deliver this in the last 90 seconds. Time it.

> *"To wrap up: the dominant constraints here are [CONSTRAINT 1] and [CONSTRAINT 2]. The architecture answers them with [CHOICE 1] and [CHOICE 2]. The two pieces I'd worry about in production are [RISK 1] — mitigated by [MITIGATION] — and [RISK 2] — which I'd accept as a known limitation that shows up at [SCALE/CONDITION]. To productionize I'd [feature flag the rollout / dark launch for N weeks / SLO-alert on Z]. If we had another 30 minutes, the next thing I'd want to deep-dive is [TOPIC]. If time allows, I'd also want to talk about cellular isolation — for this system the natural cell boundary is [BOUNDARY], and that becomes load-bearing once we hit [SCALE/COMPLIANCE TRIGGER]."*

### Why this paragraph is load-bearing

It does six things at once:
1. Restates the **problem framing** (shows you scoped well)
2. Names your **architectural conviction** (interviewer remembers what you stood for)
3. Owns **risks** unprompted (Staff signal: shows you operate systems, not just design them)
4. Volunteers **operational concerns** (productionization)
5. Self-rates the **next priority** (judgment under time constraints)
6. Surfaces **cellular isolation** (the recurring Staff topic)

Most candidates run out of time and skip this. **Do not.** Cut your last deep-dive short by 90 seconds if needed to make room.

---

## 11. Pre-interview checklist (T-30 minutes)

Use the 30 minutes before the interview for these — not for cramming new material.

### Mental
- [ ] Re-read sections 1, 3, 8, 9 of this document
- [ ] Re-read the Senior-vs-Staff calibration table in `00-framework.md` §3
- [ ] Re-read the wrap-up speech template (§10 above)
- [ ] Review your *one* most-prepared problem (e.g., `01-news-feed.md`) — *not* a new one
- [ ] Glance at the Jeff Dean latency numbers (`00-framework.md` §5)

### Physical
- [ ] Water + scratch paper + 2 pens
- [ ] If virtual: tested camera, mic, the whiteboard tool *they* will use (Excalidraw / Miro / CoderPad)
- [ ] If on-site: arrived 10 min early, hit the bathroom
- [ ] Phone on silent
- [ ] Notes app open with the wrap-up template — you can glance during a pause

### Tactical
- [ ] Decide your opening line verbatim
- [ ] Decide your default scale numbers (200M DAU is a fine universal default)
- [ ] Decide your default stack defaults (Cassandra/Scylla, Redis, Kafka, S3, multi-CDN) — so you don't waste time choosing during the interview

### Anti-checklist — DO NOT
- ❌ Try to read a new problem 30 minutes before
- ❌ Try to memorize a specific design
- ❌ Watch a YouTube mock interview right before
- ❌ Talk to anyone about how nervous you are

---

## 12. Post-interview self-debrief (within 1 hour, before memory fades)

Open a notes file. Write fast. 5–10 minutes.

```markdown
# {COMPANY} HLD — {DATE}

## Problem
{1-sentence problem statement}

## What went well
- {3 bullets}

## What I missed
- {Things the interviewer pushed on that I didn't anticipate}
- {Specific knowledge gaps revealed}
- {Decisions I'd reverse with hindsight}

## Where I lost time
- {Phase that ran long}
- {Question I overinvested in}

## Interviewer's hot questions (bring back to study)
- "Why X over Y?" → {what they were probing for}
- "What if Z fails?" → {what they wanted to hear}

## To study before next round
- [ ] {Specific topic}
- [ ] {Specific pattern}

## Calibration estimate
{My honest read: lean-no-hire / borderline / lean-hire / strong-hire}
{Why}
```

### Why this matters

You will have **another HLD round**. The patterns the interviewer pushed on are not random — they reflect what that company tests for. Your second-round prep is 10× more valuable when grounded in real signal from the first.

If you're in a multi-loop, this debrief is the most leveraged 10 minutes of your day.

---

## 13. The "if I forget everything" emergency cheat sheet

If you blank out completely, fall back to these eight steps. Even mechanical execution scores Senior.

```
1. SCOPE       → "Let me scope this. {ask 4 questions}."
2. REQS        → "Functional: A, B, C. NFR: latency X, availability Y."
3. ESTIMATE    → "DAU N → QPS Q → storage S. The constraint is [tightest]."
4. API         → "{3 endpoints}."
5. DATA MODEL  → "Sharded by {key} because {reason}."
6. ARCH        → "{Draw 5 boxes: client, gateway, service, cache, store}."
7. DEEP DIVE   → "Let me drill into {hot path}."
8. WRAP        → "Failures: {2}. Production: {flag rollout}. Monitor: {2 SLOs}."
```

This is the bare minimum. It will not get you Staff. It will get you Senior. Aim higher, but know the floor.

---

## 14. The single rule that matters most

**The interviewer must leave the room being able to repeat your single highest-conviction architectural decision and your reasoning for it.**

If they can't, you scored Senior — even if everything you said was correct.

So: at some point in every interview, plant a flag. *"My biggest architectural commitment here is [X]. The reason is [Y]. The cost is [Z]."*

Make sure they hear it. Restate it in the wrap-up.

---

## See also

- `00-framework.md` — the conceptual underpinning (patterns, calibration delta, pattern library)
- `01-news-feed.md` through `12-live-comments.md` — worked examples (read each in the form an interview would happen)
- `README.md` — how to study, suggested order
