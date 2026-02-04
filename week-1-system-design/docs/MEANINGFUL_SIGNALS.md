# Meaningful Signals Definition

## AI Relevance Engine - Signal Framework

---

## Overview

Meaningful signals are behavioral indicators that, when aggregated and analyzed, reveal user intent and content relevance without exposing individual identity or behavior sequences.

---

## Signal Categories

### 1. Engagement Signals

**Definition**: Quantifiable measures of user interaction intensity with content.

| Signal                 | Type       | Example                         | Privacy Handling                            |
| ---------------------- | ---------- | ------------------------------- | ------------------------------------------- |
| **View Duration**      | Numeric    | User spent 2m 45s on page       | Bin into buckets (0-30s, 30-60s, 1-3m, 3m+) |
| **Scroll Depth**       | Percentage | Scrolled 75% of page            | Quantize to deciles (0-10%, 10-20%, ...)    |
| **Click-Through Rate** | Ratio      | Clicked 3 of 5 recommendations  | Hash timestamp, remove click sequence       |
| **Return Frequency**   | Count      | Visited topic 12 times in month | Anonymize temporal pattern                  |
| **Session Length**     | Duration   | Spent 8 minutes total           | Aggregate across all users                  |
| **Form Submission**    | Boolean    | Completed signup form           | No persistent link to identity              |

**Significance**: Higher engagement indicates relevance alignment.

---

### 2. Interaction Type Signals

**Definition**: Categorical indicators of _how_ users engage (not _what_ they engage with).

| Signal                  | Categories                                              | Example                 | Privacy Handling                    |
| ----------------------- | ------------------------------------------------------- | ----------------------- | ----------------------------------- |
| **Interaction Method**  | Click, Scroll, Hover, Share, Comment                    | User shared article     | Categorical label only              |
| **Engagement Depth**    | Surface (browse), Moderate (read), Deep (comment/share) | User left comment       | Bucketed, no temporal sequence      |
| **Content Consumption** | Video watched, Article read, Podcast listened           | Video consumed 80%      | Duration ranges, not exact          |
| **Social Activity**     | Like, Comment, Share, Follow, Reaction                  | User reacted with emoji | Emoji type discarded, binary signal |
| **Conversion Action**   | Subscribe, Purchase, Signup, Download                   | User subscribed         | Boolean flag only                   |

**Significance**: Interaction types reveal engagement modality and user preference for content format.

---

### 3. Topic Tag Signals

**Definition**: Categorization of content themes user engages with.

| Signal                 | Structure                        | Example                                        | Privacy Handling                   |
| ---------------------- | -------------------------------- | ---------------------------------------------- | ---------------------------------- |
| **Primary Topic**      | Hierarchical taxonomy (3 levels) | Technology → AI → Machine Learning             | Store only top-level category      |
| **Secondary Topics**   | Related categories               | Also tagged: Ethics, Safety                    | Top-3 only, aggregated             |
| **Emerging Interests** | New topics within 30-day window  | User viewed "Quantum Computing" for first time | Rolling window, windowed freshness |
| **Topic Diversity**    | Count of unique topics (binned)  | Engaged with 8 different topics                | Bucketed (1-3, 3-5, 5-10, 10+)     |
| **Topic Sentiment**    | Positive, Neutral, Negative      | User clicked positive articles                 | Inferred, aggregated only          |

**Significance**: Topic signals drive recommendation targeting without revealing specific content preferences.

---

## Signal Combinations (Features)

### Derived Features

These are computed from base signals, providing additional relevance insights:

| Derived Feature          | Formula                                                | Example                               | Use Case                    |
| ------------------------ | ------------------------------------------------------ | ------------------------------------- | --------------------------- |
| **Engagement Score**     | (ViewDuration × Depth × ReturnFreq) / Recency          | High engagement on old content        | Identify strong affinity    |
| **Topic Affinity**       | Frequency of topic interaction normalized by diversity | Strong AI affinity with low diversity | Detect specialist interest  |
| **Interaction Velocity** | Interactions per day (bucketed)                        | 15 clicks/day vs 1 click/day          | Gauge activity level        |
| **Format Preference**    | Ratio of video:text:audio interactions                 | 70% video, 20% text, 10% audio        | Personalize content type    |
| **Recency Decay**        | Exponential decay of signal importance over time       | Recent signals weighted 2x more       | Adapt to evolving interests |

---

## Signal Cardinality & Binning Strategy

To preserve privacy while maintaining signal utility, we bin continuous signals:

```
Engagement Duration Signal Binning:
┌─────────────────────────────────┐
│ Raw Duration (seconds)          │
│ 0    30    60    180   300      │
├─────────────────────────────────┤
│ Bucket 0 | 1 | 2 | 3 | 4 | 5   │
├─────────────────────────────────┤
│ Label: "Skimmed" (0-30s)        │
│        "Browsed" (30-60s)       │
│        "Read" (60-180s)         │
│        "Deep Read" (180-300s)   │
│        "Returned" (300s+)       │
└─────────────────────────────────┘
```

### Binning Rules

| Signal Type     | Bins         | Rationale                     |
| --------------- | ------------ | ----------------------------- |
| **Duration**    | 5-7          | Captures engagement intensity |
| **Frequency**   | 4-6          | Distinguishes casual vs loyal |
| **Ratios**      | Deciles (10) | Smooth distribution           |
| **Categorical** | As-is        | Already discrete              |
| **Topic Count** | 6-8          | Breadth of interest           |

---

## Aggregation Levels

Signals are never processed individually; they're always aggregated:

### User Level (Temporary, Client-Side)

```
Single User Session:
├─ Topic engagement (this session)
├─ Interaction types (this session)
└─ Engagement intensity (this session)
→ Discarded after ZK proof generation
```

### Cohort Level (Server-Side, Aggregated)

```
Aggregate Anonymously:
├─ 10,000+ users with AI topic interest
├─ 50,000+ users who prefer videos
└─ 100,000+ users with high engagement
→ Never disaggregated back to individual
```

### Population Level (Analytics)

```
Global Statistics:
├─ Average engagement by topic
├─ Topic popularity trends
└─ Content type distribution
→ Public-facing, no personal inference
```

---

## Signal Validity Constraints

All signals must pass validation to be processed:

### Structural Constraints

- Duration: Must be 0 < duration < 24 hours
- Frequency: Must be non-negative integer
- Topic: Must match predefined taxonomy
- Interaction Type: Must be from defined enum

### Logical Constraints

- No future timestamps (duration binning prevents exact times)
- No negative signals (all normalized 0-1 range)
- Topic diversity < total interactions
- Recency window <= 30 days

### Privacy Constraints

- No raw user identifiers
- No device fingerprints
- No IP addresses
- No sequence patterns

---

## Signal Quality Metrics

### Completeness

- **Target**: 95%+ signals have all required fields
- **Enforcement**: Incomplete signals rejected at verification layer

### Consistency

- **Target**: < 0.1% contradiction rate (e.g., 0 interactions but high engagement)
- **Enforcement**: Logic checks in validator

### Timeliness

- **Target**: Signal-to-processing latency < 1 second
- **Enforcement**: Async ingestion with priority queue

### Utility

- **Target**: Signal distributions yield > 85% model accuracy
- **Enforcement**: Quarterly feature importance analysis

---

## Examples: From Raw Data to Privacy-Preserving Signal

### Example 1: Website Visit

```
Raw Data (NEVER SENT):
User ID: 12345
Timestamp: 2026-02-04 14:35:42 UTC
Page: /articles/ai-safety-risks
Scroll Depth: 68%
Duration: 3m 24s
Location: San Francisco, CA

↓ Anonymization & Binning

Signal Bundle (SENT):
├─ Topic Tag: "Technology/AI/Safety" (top-level only)
├─ Engagement Duration Bucket: "Deep Read" (3m 24s → 3-5m range)
├─ Scroll Depth Decile: 7 (68% → 70th percentile)
├─ Interaction Type: "Page View"
├─ Session Timestamp Hash: "a3f7e9d2..." (only for session dedup)
└─ Generated ZK Proof: "0x4a8f2..." (proves validity without revealing values)

↓ Server Processing

Aggregate Signal (NEVER LINKED TO USER):
"Population engaging with AI/Safety → 78% deep-read, avg 15-20 min sessions"
```

### Example 2: Video Interaction

```
Raw Data (NEVER STORED):
User ID: 67890
Timestamp: 2026-02-04 16:20:15 UTC
Video Title: "Quantum Computing Basics"
Watch Time: 12m 44s out of 18m total
Skip Count: 0
Share: Yes
Topic: Technology/Quantum Computing

↓ Anonymization

Signal Bundle:
├─ Topic Tag: "Technology" (only top-level)
├─ Content Type: "Video"
├─ Completion Rate Bin: 6 (70% → 60-80% bucket)
├─ Interaction Types: ["Video Play", "Share"] → Set union only
├─ Engagement Velocity: High (12+ min invested)
└─ ZK Proof: "0x8c2e4..."

↓ Server Processing

Aggregate Finding:
"Video content on Technology → High completion rates, Shareable"
```

---

## Signal Privacy Guarantee

All signals meet these privacy criteria:

1. **K-Anonymity**: No signal alone identifies < 1000 people
2. **Differential Privacy**: Signal aggregate ± noise level < 0.1 impact
3. **Temporal Privacy**: No timestamp sequence reveals behavior pattern
4. **Linkage Privacy**: Topics alone cannot reconstruct individual interests

---

## Future Signal Additions (Week 3+)

- **Behavioral Signals**: Learning pace, content difficulty preference
- **Cross-Domain Signals**: How interests transfer across topics
- **Quality Signals**: User-provided content ratings (with privacy)
- **Contextual Signals**: Time-of-day patterns (binned, aggregated)
