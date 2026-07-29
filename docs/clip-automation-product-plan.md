# Product Plan: ClipEngine — Automated Twitch-to-Shorts Clip Pipeline

**Status:** Draft v1 · **Date:** 2026-07-29 · **Owner:** Iain Macaskill

---

## 1. One-line pitch

An automated pipeline that ingests Twitch streams from popular gamers, detects highlight
moments with AI, edits them into vertical short-form video optimised for TikTok and
YouTube Shorts, and publishes them at scale — generating revenue through platform
creator-reward programmes and streamer revenue-share partnerships.

---

## 2. The critical constraint that shapes the whole product

Before anything else: **the naive version of this business does not work**, and the plan
below is structured around fixing that.

1. **Copyright.** A Twitch VOD/broadcast is the streamer's copyrighted content (and often
   contains the game publisher's content under a separate licence). Reposting clips
   without permission is infringement; channels doing this get DMCA strikes and
   terminated.
2. **Monetisation eligibility.** Both reward programmes explicitly exclude unoriginal
   content:
   - **TikTok Creator Rewards Program** requires *original* content, ≥1 minute duration,
     10k followers, and 100k valid views in the trailing 30 days. Reposted/unoriginal
     content earns nothing and risks the account.
   - **YouTube Partner Program (Shorts)** applies the "reused content" policy — clips of
     someone else's stream without significant original transformation are demonetised.
     Threshold: 1k subscribers + 10M Shorts views in 90 days (or 4k long-form watch
     hours).
3. **Detection is automated.** Both platforms fingerprint content (Content ID, TikTok's
   matching systems). "Fly under the radar" is not a strategy; it is a countdown timer.

**Therefore the product's moat is not the clipping tech — it's the rights layer.** The
viable framings, in order of attractiveness:

| Model | Description | Who earns the rewards |
|---|---|---|
| **A. Streamer-as-customer (SaaS/agency)** | Streamers (or their agencies) run ClipEngine on *their own* streams; output posts to *their* TikTok/Shorts accounts. Content is original to the account owner → fully rewards-eligible. | The streamer; we charge SaaS fee and/or % of rewards |
| **B. Licensed clip network** | We operate clip channels under written licence + revenue-share agreements with streamers. Transformation (editing, captions, commentary) supports originality claims. | Us, sharing back to streamers |
| **C. Unlicensed reposting** | Scrape and repost. | ❌ Not viable: infringing, demonetised, account bans. **Out of scope.** |

**Recommendation: lead with Model A**, with Model B as an expansion for streamers who
don't want to manage accounts. The same pipeline powers both. Model A also solves cold
start: the streamer's existing audience seeds the follower/view thresholds the reward
programmes require.

---

## 3. Market and opportunity

- **Supply:** Top Twitch streamers produce 4–10 hours of raw content per day; almost all
  of it is under-exploited on short-form platforms. Manually clipping costs $500–3,000/mo
  per streamer (human editors), which only the top ~1% can justify.
- **Demand:** Short-form is the discovery engine for streamers — clips drive new Twitch
  follows and sponsorship value. Mid-tier streamers (1k–50k CCV… roughly 5k–500k
  followers) want this but can't afford editors.
- **Comparable products:** Opus Clip, Eklipse.gg, Momento, StreamLadder. Eklipse is the
  closest comparator (Twitch-focused, freemium). Differentiation targets: (a) better
  highlight detection using chat + game events, not just audio; (b) fully hands-off
  auto-publish with per-platform optimisation; (c) rewards/earnings analytics so the
  streamer sees ROI in currency, not views.
- **Revenue benchmarks for modelling** (order-of-magnitude, volatile):
  - TikTok Creator Rewards: ~$0.40–$1.00 RPM on qualified views (>1 min videos only).
  - YouTube Shorts: ~$0.10–$0.30 RPM.
  - Implication: a channel doing 10M qualified views/month spans roughly $1k–$10k/month.
    Rewards alone are thin margins — which further supports SaaS pricing (Model A) as the
    primary revenue line, with rewards share as upside.

---

## 4. Product overview

### 4.1 User-facing product (Model A)

A web dashboard where a streamer:

1. Connects Twitch (OAuth) + TikTok + YouTube accounts.
2. Sets preferences: games, clip style, caption style, branding overlay, posting cadence,
   auto-publish vs. review queue.
3. Receives clips: pipeline watches their streams/VODs, generates candidate clips ranked
   by predicted performance, and either auto-publishes or queues for one-tap approval.
4. Sees analytics: views, follower growth, estimated rewards earnings per platform, and
   which clip *types* perform (feedback loop into detection).

### 4.2 The pipeline (the core asset)

```
┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌─────────────┐  ┌───────────┐
│ INGEST     │→ │ DETECT       │→ │ EDIT       │→ │ PACKAGE     │→ │ PUBLISH   │
│ VOD/live   │  │ highlights   │  │ cut+crop   │  │ captions,   │  │ TikTok /  │
│ + chat log │  │ multi-signal │  │ vertical   │  │ hook, title │  │ Shorts    │
└────────────┘  └──────────────┘  └────────────┘  └─────────────┘  └─────┬─────┘
                                                                        │
                                          ┌─────────────────────────────▼──────┐
                                          │ ANALYTICS + LEARNING LOOP          │
                                          │ per-clip performance → detector    │
                                          └────────────────────────────────────┘
```

**Stage 1 — Ingest**
- Twitch Helix API + authorised VOD access (streamer OAuth grants VOD download rights).
  Live-mode later: ingest the stream in near-real-time so clips post while the stream is
  still trending.
- Capture the **chat log with timestamps** — this is the highest-value cheap signal.
- Store raw segments in object storage; retain only a rolling window (cost control).

**Stage 2 — Highlight detection (multi-signal scoring)**
Score the timeline in sliding windows using an ensemble:

| Signal | Method | Cost |
|---|---|---|
| Chat velocity + emote spikes (LUL, PogChamp, KEKW clusters) | Time-series spike detection over chat log | Very low |
| Audio excitement (shouting, laughter, sudden silence) | Audio energy + laughter classifier | Low |
| Streamer speech content | Whisper ASR → local/LLM scoring of "clip-worthiness" (jokes, rage, hype callouts) | Medium |
| Game events (kills, clutches, wins) | CV detection of killfeeds/victory screens per supported game; start with 3–5 top games | Medium |
| Twitch native clip creation rate | Viewers clipping = ground truth interest | Very low |

Fuse into a single score; take top-N non-overlapping windows per stream. **Key design
decision:** chat + audio alone get ~80% of the value at ~5% of the compute cost of
frame-level vision. Ship that first; add game-event CV per title later.

**Stage 3 — Edit**
- Boundary refinement: snap cut points to sentence boundaries (from ASR) and
  action boundaries, with a pre-roll to preserve context/setup.
- Vertical reformat (9:16): detect facecam region → facecam top, gameplay bottom (the
  dominant layout meta), or smart-crop gameplay when no facecam. FFmpeg-based, GPU where
  available.
- Duration targeting: **default 61–90 seconds** — this is not aesthetic, it's economic:
  TikTok Creator Rewards only pays on videos over 1 minute. Sub-60s variants only for
  pure-reach posts.

**Stage 4 — Package**
- Burned-in animated captions (word-level Whisper timestamps) — captions are
  table-stakes for short-form retention.
- Hook engineering: first 1.5s gets a text hook ("he did NOT expect this…") generated by
  LLM from the transcript; thumbnails for Shorts.
- Metadata per platform: title, hashtags, sounds-safe check (mute/replace copyrighted
  music segments — critical, background Spotify is the #1 DMCA cause in stream VODs).
- Streamer branding overlay + "follow on Twitch" end-card (this is the value prop that
  makes streamers pay).

**Stage 5 — Publish**
- TikTok Content Posting API (requires audited app approval — long lead time, start
  early) and YouTube Data API `videos.insert`.
- Scheduling engine: per-platform optimal posting windows, spacing rules, per-account
  daily caps (platforms throttle/flag bulk posting).
- Review queue mode (default for new accounts) vs. full auto.

**Stage 6 — Analytics + learning loop**
- Pull per-video stats via platform APIs; join with clip features (signal scores, game,
  duration, hook type).
- Retrain/reweight the detector per streamer — each audience has its own taste. This
  compounding per-customer model quality is the long-term moat.

---

## 5. Architecture & stack (proposed)

- **Orchestration:** queue-based workers (Celery/RQ or Temporal); each pipeline stage a
  worker type; horizontal scale on GPU stages only.
- **Media:** FFmpeg; PySceneDetect; Whisper (faster-whisper on GPU) for ASR; small
  audio classifiers (laughter/excitement); YOLO-class detector for facecam/killfeed.
- **LLM usage:** clip-worthiness scoring, hooks, titles, hashtags. Latency-insensitive →
  batch calls; small local models for high-volume scoring, frontier model (Claude) for
  final packaging copy on top-ranked clips only.
- **App:** FastAPI backend + simple React dashboard; Postgres (clips, scores, schedules,
  earnings); S3-compatible object storage with lifecycle deletion.
- **Cost model target:** < $0.15 fully-loaded compute per published clip (ASR is the
  dominant cost; chat-gated processing keeps us from transcribing 8h of silence).

---

## 6. MVP scope (Phase 1)

**Goal: prove the pipeline produces clips a streamer would actually post, for 3–5 design
partners, in 6–8 weeks.**

In scope:
- VOD-based (not live) ingestion for connected streamer accounts.
- Detection: chat velocity + audio excitement + Twitch clip-rate signals only.
- Facecam-top/gameplay-bottom vertical edit, word-level captions, 61–90s targeting.
- Review queue + manual export; **YouTube Shorts auto-publish only** (TikTok API audit
  will still be pending; MVP exports TikTok-ready files for manual upload).
- Basic dashboard: connect accounts, review/approve, view posted clips.

Explicitly out of scope for MVP: live ingestion, game-event CV, TikTok auto-post,
earnings analytics, Model B network channels.

**MVP success criteria:** ≥50% of generated candidate clips approved by design partners;
≥1 clip from the pipeline outperforming the streamer's manual-clip median views.

## 7. Roadmap after MVP

- **Phase 2 (months 3–4): Publish & measure.** TikTok Content Posting API (submit audit
  in week 1 of MVP), scheduling engine, analytics ingestion, earnings estimates,
  per-streamer detector tuning. Start charging (£49–£199/mo tiers by clip volume).
- **Phase 3 (months 5–6): Scale & sharpen.** Live-mode clipping (post within minutes of
  the moment), game-event CV for top 5 titles, A/B hooks, multi-language caption
  translation (huge cheap reach multiplier).
- **Phase 4 (months 6+): Model B network.** Standard licence + rev-share contract
  template; operate managed channels for streamers who opt in; compilation formats
  (top-10s) for long-form YouTube monetisation.

---

## 8. Business model & unit economics

- **Primary revenue: SaaS.** Tiers by published-clip volume and features (auto-publish,
  live mode, multi-language). Target £99/mo average.
- **Secondary: rewards share** on managed (Model B) channels, e.g. 30% to us / 70%
  streamer.
- **Illustrative economics per Model A customer:** ~60 clips/mo × $0.15 compute ≈ $9
  COGS against £99 revenue → ~90% gross margin. The sensitivity is ASR/GPU cost and
  support load, not platform RPM — which is the point of being SaaS-first.
- **Break-even sanity check for Model B (rewards-dependent):** a managed channel needs
  roughly 1–2M qualified TikTok views/mo to cover its own compute + ops before any
  profit. Treat Model B channels as a portfolio; kill underperformers monthly.

## 9. Key risks & mitigations

| Risk | Severity | Mitigation |
|---|---|---|
| Platform policy: content deemed unoriginal / rewards clawed back | High | Model A (owner-posted) as default; documented licences for Model B; real editorial transformation, never raw re-uploads |
| Music DMCA inside gameplay/stream audio | High | Audio fingerprint check stage; auto-mute/replace music beds before publish |
| TikTok API audit rejection or posting caps | Medium | Apply early; review-queue + export fallback; diversify to Shorts + Instagram Reels |
| Detection quality below "streamer would post this" bar | Medium | Human-in-the-loop review queue from day 1; per-streamer learning loop |
| Platforms change RPM/thresholds (they do, often) | Medium | SaaS-first revenue; rewards treated as upside, never base case |
| Incumbents (Eklipse, Opus) | Medium | Differentiate on chat-signal detection quality, hands-off publishing, and earnings-denominated analytics |
| AI-content disclosure rules tighten | Low | Clips are human-created content, AI-edited; label where required |

## 10. KPIs

- **Pipeline quality:** candidate→approved rate (target >50% MVP, >75% by Phase 3);
  time-from-moment-to-published (VOD: <12h; live mode: <15min).
- **Customer value:** views per published clip vs. streamer's manual baseline; Twitch
  follower conversion from end-cards; churn <5%/mo.
- **Business:** MRR, gross margin (>85%), compute cost per published clip (<$0.15),
  Model B portfolio rewards RPM.

## 11. Immediate next steps

1. Validate demand: 10 conversations with mid-tier streamers (1k–20k CCV); pre-sell 3–5
   design partnerships at a discount.
2. Submit TikTok developer app for Content Posting API audit (longest external lead
   time).
3. Draft the Model A terms + Model B licence/rev-share template with a lawyer (one-time
   cost, unlocks the whole rights strategy).
4. Technical spike (1 week): chat-spike + audio-energy detector against 3 public VODs;
   eyeball whether top-10 windows match the streamer's own posted clips.
5. Build MVP per §6.
