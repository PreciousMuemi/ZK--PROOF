# Safe, Non-Identifiable Behavioral Signals

## Week 2: Data & Signal Layer Design

**Goal**: Define signals that determine content relevance WITHOUT enabling re-identification

---

## Executive Summary

A behavioral signal is "safe" if it:

1. Cannot identify an individual user (even with auxiliary data)
2. Aggregates meaningfully across thousands of users
3. Reveals content relevance preferences, not personal identity
4. Survives re-identification attacks (linkage, differential, statistical)

This document defines 40+ safe signals across 5 categories, with explicit safeguards against each re-identification vector.

---

## Core Principle: Signal vs. Identity

### ❌ DANGEROUS (Identifiable)

```
Raw Data:
- "User clicked article 42 at 14:35:22 UTC on 2026-02-04"
- "User watched video 'AI Safety Risks' for 8m 34s"
- "User from San Francisco viewed 'Quantum Computing'"
- "User with 95 page views in 30 days"

Risk: Timestamp + content + metadata = re-identification
```

### ✅ SAFE (Non-Identifiable)

```
Processed Signal:
- "Interaction type: article_view, bucketed_duration: 5-10min"
- "Content type: video, completion_percent_decile: 8 (80%)"
- "Topic preference: Quantum Computing, regional_cohort: Pacific"
- "Engagement frequency bin: high (50-200 views/month)"

Safety: Aggregated with 50K+ others, exact values hidden, temporal masked
```

---

## Signal Category 1: Content Tags (Topic & Metadata)

Content tags are **SAFE** because they're categorical, aggregatable, and cannot identify individuals.

### 1.1 Content Category Tags

| Signal                | Definition                  | Values                                                                                                | Aggregation                 | Safety                       |
| --------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------- | --------------------------- | ---------------------------- |
| **Primary Topic**     | Top-level content category  | `[Technology, Science, Entertainment, Politics, Health, Business, Sports, Culture, Education, Other]` | Shared by 50K+ users        | High - categorical           |
| **Secondary Topics**  | Related categories (binned) | Any 3 of above, can combine                                                                           | Binned to top-3 most common | High - coarse grained        |
| **Content Depth**     | Complexity level            | `[Beginner, Intermediate, Advanced, Expert]`                                                          | Aggregated by topic         | High - ordinal, not personal |
| **Content Freshness** | Publication recency         | `[Breaking (0-6h), Recent (6h-7d), Standard (7d-30d), Archived (30d+)]`                               | Temporal binned             | High - time-bucketed         |
| **Content Type**      | Medium format               | `[Article, Video, Audio/Podcast, Interactive, Data Visualization, Course]`                            | Ratio per topic             | High - format preference     |

### 1.2 Content Quality Tags

| Signal                 | Definition              | Values                                                                               | Aggregation                    | Safety                          |
| ---------------------- | ----------------------- | ------------------------------------------------------------------------------------ | ------------------------------ | ------------------------------- |
| **Credibility Level**  | Source authority        | `[High Authority, Peer-Reviewed, Journalist, Blog, User-Generated, Viral, Disputed]` | Distribution across users      | High - editorial tag            |
| **Sentiment Polarity** | Content tone            | `[Positive, Neutral, Negative]`                                                      | Inferred from text, aggregated | High - not personal opinion     |
| **Education Value**    | Learning content rating | `[Tutorial, How-to, Reference, Opinion, News, Entertainment]`                        | Binned per topic               | High - content attribute        |
| **Controversy Level**  | Potential disagreement  | `[Consensus, Debated, Controversial, Polarizing]`                                    | Content-based tag, aggregated  | High - editorial classification |

### 1.3 Domain/Authority Tags

| Signal               | Definition                  | Values                                                                                                            | Aggregation                             | Safety                        |
| -------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------- | ----------------------------- |
| **Creator Type**     | Content producer category   | `[Institutional (Brand), Professional (Individual), Academic, User-Generated, Government, Non-profit, Celebrity]` | Aggregated per topic                    | High - creator attribute only |
| **Domain Authority** | Publisher credibility score | Bucketed: `[Low: 0-30, Medium: 30-60, High: 60-85, Very High: 85-100]`                                            | Not stored individually, only aggregate | High - binned score           |
| **Publication Tier** | Media source rank           | `[Tier 1 (NYT), Tier 2 (Major), Tier 3 (Niche), Tier 4 (Independent)]`                                            | Distribution per topic                  | High - categorical            |

**Safety Guarantee**: All content tags are derived from _content properties_, not user identity. No tag reveals who engaged with content.

---

## Signal Category 2: Interaction Types (Behavioral Actions)

Interaction types are **SAFE** because they're categorical, not temporal, and cannot form sequences that identify individuals.

### 2.1 Direct Interaction Signals

| Signal                 | Definition                 | Safe Values                                                                  | Unsafe (NOT Captured)              | Safety Measure                  |
| ---------------------- | -------------------------- | ---------------------------------------------------------------------------- | ---------------------------------- | ------------------------------- |
| **View Event**         | Content was displayed      | `[article_view, video_view, audio_play, preview_seen]`                       | NOT: Exact time, view order        | Categorical only, no sequences  |
| **Engagement Depth**   | How user engaged           | `[skimmed (0-30s), browsed (30-60s), read (60-180s), deep (180s+)]`          | NOT: Exact duration                | Binned duration, not timestamps |
| **Scroll Interaction** | Vertical scroll behavior   | `[minimal (0-25%), moderate (25-50%), substantial (50-75%), full (75-100%)]` | NOT: Scroll speed, scroll patterns | Depth bucketed, not temporal    |
| **Click Pattern Type** | Click interaction category | `[no_clicks, sparse (1-2), moderate (3-5), frequent (5+)]`                   | NOT: Click sequence, click targets | Count binned, not order         |
| **Hover Signals**      | Attention indicator        | `[no_hover, brief (< 5s), engaged (5-15s), sustained (15s+)]`                | NOT: Exact duration, hover path    | Duration binned, aggregated     |

### 2.2 Social Interaction Signals

| Signal                   | Definition               | Safe Values                               | Unsafe (NOT Captured)             | Safety Measure                 |
| ------------------------ | ------------------------ | ----------------------------------------- | --------------------------------- | ------------------------------ |
| **Like/Upvote**          | Positive feedback        | `[action: liked, value: 1 point]`         | NOT: Who liked, sequence of likes | Binary event only, no linking  |
| **Share/Recommendation** | Content forwarding       | `[action: shared, channel: undefined]`    | NOT: Share destination, to whom   | Action type only, no targets   |
| **Comment Engagement**   | Discussion participation | `[action: commented, sentiment: neutral]` | NOT: Comment text, who replied    | Action type binned, no content |
| **Bookmark/Save**        | Favoriting behavior      | `[action: bookmarked]`                    | NOT: Bookmark reason, retrieval   | Binary flag only               |
| **Report/Flag**          | Content moderation       | `[action: reported, reason: category]`    | NOT: Report text, reporter ID     | Category reason only           |

### 2.3 Conversion Signals

| Signal                | Definition                | Safe Values                                  | Unsafe (NOT Captured)                    | Safety Measure                |
| --------------------- | ------------------------- | -------------------------------------------- | ---------------------------------------- | ----------------------------- |
| **Subscribe Action**  | Subscription initiation   | `[subscribed: boolean]`                      | NOT: Subscription tier, payment method   | Boolean flag only, no details |
| **Download/Export**   | Content capture           | `[action: downloaded, format: pdf]`          | NOT: Downloaded file name, personal data | Format type only              |
| **Signup Completion** | Account/list registration | `[action: signed_up, form_type: newsletter]` | NOT: Email, signup context               | Action type only              |
| **Purchase Signals**  | Transaction indicator     | `[action: purchased, category: tier]`        | NOT: Product name, amount, payment ID    | Category aggregation only     |

**Safety Guarantee**: Interactions are _actions without context_. No interaction type alone identifies a user or enables re-identification across sessions.

---

## Signal Category 3: Engagement Frequency & Intensity

Frequency signals are **SAFE** when aggregated and binned; they become unsafe only when combined with exact timestamps.

### 3.1 Temporal Frequency Signals (Binned)

| Signal               | Definition                        | Safe Bins                                                                             | Unsafe (NOT Captured)                  | Safety                  |
| -------------------- | --------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------- | ----------------------- |
| **Views per Hour**   | Content consumption rate (hourly) | `[inactive (0), low (1-2), moderate (3-5), high (5+)]`                                | NOT: Exact times, hourly pattern       | Binned into 4 buckets   |
| **Views per Day**    | Daily engagement                  | `[very_low (1-3), low (4-10), moderate (11-30), high (31-100), very_high (100+)]`     | NOT: Time of day, day pattern          | 5-bucket distribution   |
| **Views per Week**   | Weekly activity                   | `[minimal (1-5), light (6-20), moderate (21-50), heavy (51-100), very_heavy (100+)]`  | NOT: Weekly pattern, day-of-week       | 5-bucket distribution   |
| **Views per Month**  | Monthly engagement                | `[dormant (0-5), casual (6-20), regular (21-50), active (51-100), power_user (100+)]` | NOT: Month-to-month trends             | 5-bucket distribution   |
| **Return Frequency** | Revisit behavior                  | `[one_time (1), occasional (2-5), regular (6-20), frequent (20+)]`                    | NOT: Return interval, revisit patterns | 4-bucket categorization |

### 3.2 Topic Engagement Frequency

| Signal                             | Definition                       | Safe Bins                                                                      | Unsafe (NOT Captured)              | Safety              |
| ---------------------------------- | -------------------------------- | ------------------------------------------------------------------------------ | ---------------------------------- | ------------------- |
| **Topic Views per Topic**          | Per-category engagement          | `[rare (1-3), occasional (4-10), frequent (11-30), obsessive (30+)]`           | NOT: Topic sequence, transitions   | Per-topic bucketed  |
| **Topic Diversity Index**          | Number of topics (30-day window) | `[specialist (1-3), generalist (4-10), polymath (10+)]`                        | NOT: Topic sequence                | Count bucketed      |
| **Format Preference Distribution** | Video vs article vs audio ratio  | `[video_heavy (60-100%), mixed (30-60%), text_heavy (0-30%)]`                  | NOT: Format sequence               | Ratio ranges        |
| **Content Depth Progression**      | Complexity engagement over time  | `[beginner_only, beginner→intermediate, intermediate→advanced, advanced_only]` | NOT: Learning path, exact sequence | Categorical buckets |

### 3.3 Recency-Weighted Engagement

| Signal                    | Definition                              | Safe Representation                                                  | Unsafe (NOT Captured)                | Safety                        |
| ------------------------- | --------------------------------------- | -------------------------------------------------------------------- | ------------------------------------ | ----------------------------- |
| **Recent Activity Spike** | Last 7-day engagement vs 30-day average | `[increasing, stable, declining]`                                    | NOT: Daily values, exact change      | Categorical trend only        |
| **Engagement Momentum**   | Activity acceleration/deceleration      | `[accelerating (trend: ↑), stable (trend: →), declining (trend: ↓)]` | NOT: Velocity metric, actual numbers | Direction only, not magnitude |
| **Interest Freshness**    | How recent are topic interests          | `[fresh (added 0-7d), active (7-30d), dormant (30d+)]`               | NOT: Exact dates, timeseries         | Window-based category         |
| **Session Recency**       | Time since last activity                | `[today, yesterday, 2-7d_ago, 7-30d_ago, 30d+_ago]`                  | NOT: Exact timestamp                 | Time bucket only              |

**Safety Guarantee**: Frequency signals are _aggregated distributions_, never individual trajectories. Individual binning prevents both identification and behavior pattern analysis.

---

## Signal Category 4: Engagement Quality Metrics

Quality signals measure _how meaningfully_ users engage, not _who_ they are.

### 4.1 Depth & Intensity Metrics

| Signal                           | Definition                                 | Safe Measurement                                                               | Unsafe (NOT Captured)                | Safety                  |
| -------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------ | ------------------------------------ | ----------------------- |
| **Completion Rate**              | How much content was consumed              | `[10th_percentile, 20th, ..., 90th_percentile]`                                | NOT: Individual percentile ranking   | Decile only, aggregated |
| **Time Investment Intensity**    | Duration spent relative to content length  | `[skim (0-25%), scan (25-50%), read (50-75%), immerse (75-100%)]`              | NOT: Exact time, time patterns       | Binned ratio only       |
| **Interactive Engagement Index** | Clicks + scrolls + interactions normalized | `[passive (0-1 action/min), active (1-2), engaged (2-5), highly_engaged (5+)]` | NOT: Individual interaction sequence | Count per minute binned |
| **Returning Ratio**              | Revisits to initial content                | `[none, single (1 return), multiple (2-5), frequent (5+)]`                     | NOT: Return timing, intervals        | Count bucket only       |

### 4.2 Preference Strength Signals

| Signal                         | Definition                      | Safe Expression                                                                              | Unsafe (NOT Captured)                   | Safety                 |
| ------------------------------ | ------------------------------- | -------------------------------------------------------------------------------------------- | --------------------------------------- | ---------------------- |
| **Topic Affinity Score**       | Aggregate engagement per topic  | `[low (bucket 1), medium_low (2), medium (3), medium_high (4), high (5)]`                    | NOT: Individual score, derivation       | 5-level ordinal scale  |
| **Format Preference Strength** | How strongly prefers one format | `[flexible (even distribution), biased (70-30), strong (80-20), exclusive (95-5)]`           | NOT: Individual ranking, scores         | Distribution bucket    |
| **Content Type Preference**    | Depth/complexity preference     | `[surface (beginners), mainstream (intermediates), specialist (advanced), expert (extreme)]` | NOT: Individual level, progression      | Category bucket        |
| **Niche vs Mainstream**        | Affinity for obscure content    | `[mainstream_only, mainstream_biased, balanced, niche_biased, niche_only]`                   | NOT: Specific niches, personal identity | General classification |

### 4.3 Engagement Consistency

| Signal                       | Definition                     | Safe Measurement                                                               | Unsafe (NOT Captured)              | Safety                     |
| ---------------------------- | ------------------------------ | ------------------------------------------------------------------------------ | ---------------------------------- | -------------------------- |
| **Engagement Volatility**    | Variability in activity levels | `[stable, moderate_variance, high_variance]`                                   | NOT: Individual trend, patterns    | Categorical variance level |
| **Topic Consistency**        | Stability of topic interests   | `[constant (same topics), evolving (gradual change), shifting (rapid change)]` | NOT: Exact topic changes, timeline | Categorical stability      |
| **Content Type Consistency** | Stability of format preference | `[loyal (same format), flexible (adapts), experimenting (tries new)]`          | NOT: Specific transitions          | Categorical behavior       |
| **Predictability Index**     | How consistent is behavior     | `[predictable, variable, erratic]`                                             | NOT: Individual pattern, specifics | Categorical only           |

**Safety Guarantee**: Quality metrics aggregate across users with same engagement patterns. No metric reveals individual identity or enables re-identification.

---

## Signal Category 5: Derived & Composite Signals

Derived signals combine primary signals into features for relevance modeling, while maintaining safety through aggregation.

### 5.1 Topic-Based Composite Signals

| Signal                        | Formula                                     | Safe Representation                              | Example                                                          | Safety                                         |
| ----------------------------- | ------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------- | ---------------------------------------------- |
| **Topic Affinity Matrix**     | Sum(engagement in topic) / Total engagement | Sparse matrix: topic → affinity bucket           | `AI: 0.35 (bucket 3), Science: 0.20 (bucket 2), ...`             | Aggregated across users with similar interests |
| **Content Type Preference**   | Count(video) / (Count(all)) for each topic  | Topic-format ratios, binned                      | `AI→video: 65%, article: 30%, audio: 5%`                         | Distribution aggregated across cohort          |
| **Depth Progression Score**   | (Advanced views) / (All views) per topic    | Ordinal scale (0=beginner only, 5=advanced only) | Topic "Quantum Computing": depth score 3 (intermediate-advanced) | Binned progression level                       |
| **Topic Exploration Pattern** | (New topics in 7d) / (Topics in 30d)        | Categorical: explorer, specialist, balanced      | Pattern: "balanced" (70% old, 30% new topics)                    | Aggregated explorer behavior                   |

### 5.2 Temporal Composite Signals

| Signal                        | Formula                                  | Safe Representation                            | Example                                                 | Safety                                             |
| ----------------------------- | ---------------------------------------- | ---------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------- |
| **Recency-Weighted Affinity** | Sum(engagement × decay_weight) per topic | Topic → recency-adjusted bucket (1-5)          | `AI: 4 (recent focus), Science: 2 (older interest)`     | Aggregate decay function, no individual timeseries |
| **Engagement Trend**          | (Last 7d avg) vs (Prior 30d avg)         | Categorical: accelerating, stable, declining   | Trend for user: "accelerating" (7d higher than 30d avg) | Aggregated across users with trend                 |
| **Seasonal/Cyclical Pattern** | Engagement variation by time bucket      | Categorical: consistent, time_biased, variable | Pattern: "time_biased" (higher engagement evenings)     | Aggregated at cohort level, no individual pattern  |
| **Session Pattern Type**      | (# of sessions) vs (# of days active)    | Categorical: daily_constant, bursty, episodic  | Pattern: "episodic" (few sessions, high intensity each) | Aggregated pattern type                            |

### 5.3 Multi-Topic Composite Signals

| Signal                       | Formula                                       | Safe Representation                                      | Example                                                             | Safety                                                 |
| ---------------------------- | --------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------ |
| **Topic Correlation**        | Co-engagement of topics (Pearson correlation) | Symmetric matrix of topic pairs → correlation bucket     | `AI ↔ Science: corr 0.7 (high)`, `AI ↔ Sports: corr 0.1 (low)`      | Population-level correlations, no individual inference |
| **Interest Diversity Index** | Entropy of topic distribution                 | Categorical: specialist (low entropy), generalist (high) | Diversity: "generalist" (entropy score 3.2 out of 4)                | Aggregated diversity type                              |
| **Topic Jump Behavior**      | Distance between engaged topics in taxonomy   | Categorical: related_topics, distant_topics, random      | Behavior: "related_topics" (mostly adjacent in taxonomy)            | Aggregated cohort behavior                             |
| **Cross-Topic Affinity**     | How well topics predict each other            | Categorical: high_predictability, moderate, low          | Affinity: "high_predictability" (knowing one topic predicts others) | Population-level predictive power                      |

### 5.4 Content-Signal Interactions

| Signal                       | Formula                                            | Safe Representation                                            | Example                                                          | Safety                     |
| ---------------------------- | -------------------------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------- | -------------------------- |
| **Quality Preference**       | Affinity with [High Authority, Peer-Reviewed] tags | Categorical: trusts_experts, prefers_mainstream, flexible      | Preference: "trusts_experts" (80% consumption of high authority) | Aggregated over cohort     |
| **Complexity Appetite**      | Affinity with [Beginner] vs [Advanced] content     | Ordinal scale: novice_seeking (1) to expert_seeking (5)        | Appetite: "expert_seeking" (75% advanced content)                | Binned preference level    |
| **Format-Topic Specificity** | Different format preferences per topic             | Topic → [format distribution]                                  | `AI: video 60% (learns visually), Science: article 50% (reads)`  | Per-topic format binned    |
| **Creator Type Affinity**    | Preference for [Institutional] vs [Independent]    | Categorical: institutional_heavy, creator_mix, creator_focused | Preference: "creator_mix" (50-50 split)                          | Aggregated preference type |

**Safety Guarantee**: Derived signals are computed from aggregated primary signals. No derived signal reveals individual identity or enables unique identification.

---

## Signal Safety Validation Framework

All signals must pass 4 safety tests:

### Test 1: K-Anonymity Check

```
Question: Can this signal alone identify fewer than 1000 users?
- If YES: ❌ UNSAFE - Signal reveals too much
- If NO: ✅ SAFE - Signal aggregates with 1000+ others

Example:
- ❌ "User watched 'Specific Video Title' for 7m 23s" → Unique
- ✅ "Video engagement depth bucket: 7 (70-80%)" → 50K+ users in bucket
```

### Test 2: Linkage Attack Resistance

```
Question: Can this signal be linked with other signals to re-identify?
- Without temporal sequence: Harder to link
- Without personal attributes: Cannot identify
- Without device ID: Cannot cross-device link

Example:
- ❌ (View A → View B → View C) + Timestamp → Unique sequence
- ✅ (Topic affinity: AI, Format: video) + 100K others with same → Safe
```

### Test 3: Auxiliary Information Resistance

```
Question: Does this signal leak information if combined with public data?
- Public data: Wikipedia, Wikidata, news, social media
- If combined → Can identify? ❌ UNSAFE
- If only noisy aggregate → Cannot identify? ✅ SAFE

Example:
- ❌ "Viewed articles: [AI Safety, Quantum Computing, Cryptography]" + Wikipedia = Unique person
- ✅ "Topic affinity buckets: [Technology, Science]" + 10K+ users = Safe
```

### Test 4: Re-Identification Attack Simulation

```
Simulate 3 types of attacks:

A. Inference Attack: Can attacker deduce identity from signal?
   - If YES: ❌ UNSAFE
   - If NO: ✅ SAFE

B. Membership Inference: Can attacker know if X was in dataset?
   - With signal: P(in) = 0.5X (cannot distinguish)
   - If distinguishable: ❌ UNSAFE
   - If indistinguishable: ✅ SAFE

C. Attribute Inference: Can attacker deduce sensitive attributes?
   - Demographics, health, location, political views?
   - If YES: ❌ UNSAFE
   - If NO: ✅ SAFE
```

---

## Signal Composition Rules (What NOT to Mix)

### ❌ Unsafe Combinations

```
DO NOT COMBINE:
1. Exact timestamp + specific content
   → Enables activity reconstruction

2. Demographic signal + topic signal
   → Enables demographic profiling

3. Device ID + engagement pattern
   → Enables cross-device tracking

4. Location + content preference
   → Enables behavioral geo-mapping

5. Interaction sequence + timestamps
   → Enables behavior pattern analysis

6. Exact count + user identifier
   → Enables re-identification via count uniqueness
```

### ✅ Safe Combinations

```
THESE ARE SAFE TO COMBINE:
1. Topic bucket + Format bucket
   → "AI + video" (binned, no identity)

2. Engagement frequency bucket + Recency bucket
   → "High engagement + Recent" (temporal masked)

3. Quality preference bucket + Diversity bucket
   → "Expert-seeking + Generalist" (aggregated)

4. Multiple topic buckets (no sequence)
   → "AI + Science + Technology" (set, not sequence)

5. Derived features (all aggregated)
   → "Affinity score + Consistency score" (composite safe)
```

---

## Signal Validation Rules

Every signal must satisfy:

### Rule 1: Cardinality Constraint

```
Signal cardinality must be SMALL (prevents uniqueness):
- Topic buckets: 10 categories
- Frequency buckets: 4-5 bins
- Quality levels: 4-5 ordinal
- NOT: Continuous values, unbounded counts
```

### Rule 2: Aggregation Guarantee

```
Every signal aggregated with 1000+ other users:
- No individual signal values stored
- Only aggregate statistics retained
- Minimum cohort size: 10,000 users
```

### Rule 3: Temporal Masking

```
No exact timestamps stored:
- Only time buckets (hour, day, week, month bins)
- No minute/second precision
- No sequence information
- Recency expressed as "days ago" bucket
```

### Rule 4: Content Masking

```
Never store specific content:
- Only category tags (Technology, Science, etc.)
- Not: Article titles, Video names, URLs
- Not: Specific creators, Sources
- Only: Creator types (Institutional, Independent, etc.)
```

### Rule 5: No Personal Attributes

```
Never infer or store:
- Age, Gender, Ethnicity
- Location below country level
- Health, Financial, Legal info
- Educational level, Occupation
- Family status, Relationship info
```

---

## Signal Examples in Context

### Example 1: Article Engagement

```
RAW (NEVER TRANSMITTED):
- User ID: 12345
- Timestamp: 2026-02-04 14:35:42.123 UTC
- URL: /articles/ai-safety-risks-2026
- Title: "Top 10 AI Safety Risks in 2026"
- Duration: 8m 47s
- Scroll: 82%
- Device: iPhone 14 Pro

↓ ANONYMIZATION & BINNING

SAFE SIGNAL (TRANSMITTED):
- Topic: Technology (top-level only)
- Secondary: AI, Safety (top-2 secondary)
- Content Type: Article
- Content Depth: Intermediate
- Content Freshness: Recent (published < 7d)
- Creator Type: Journalist
- Domain Authority: Tier 2
- Engagement Depth: Deep read (8m 47s → 5-10m bucket)
- Scroll Depth: Substantial (82% → 75-100% bucket)
- Interaction Type: article_view
- Session Context: Part of "heavy engagement" session

ZK PROOF: Proves above signals valid without revealing them
```

### Example 2: Video Consumption

```
RAW (NEVER TRANSMITTED):
- User ID: 67890
- Video: "Quantum Computing Explained" (YouTuber John)
- Duration: 18m 34s watch time out of 22m total
- Skip: 0 skips
- Share: Shared to 3 people
- Like: Yes

↓ ANONYMIZATION & BINNING

SAFE SIGNAL (TRANSMITTED):
- Topic: Science
- Secondary: Quantum Computing
- Content Type: Video
- Content Depth: Intermediate-Advanced
- Creator Type: Individual Creator
- Domain Authority: Bucket 3 (Popular/Viral)
- Engagement Depth: Watched (84% completion → 80-90% bucket)
- Completion Rate: High (decile 8)
- Scroll/Interactions: None (video-specific)
- Interaction Type: video_play
- Social Action: shared (type only, no targets)

DERIVED SIGNALS:
- Format Preference: +0.3 toward Video (aggregated)
- Topic Affinity: Science +0.15 (aggregated)
- Quality Preference: Popular creator affinity (aggregated)

ZK PROOF: Proves signal validity without revealing values
```

### Example 3: Topic Browsing Session

```
RAW SESSION DATA (NEVER SENT AS SEQUENCE):
- 14:00 - Viewed AI article (8m)
- 14:08 - Viewed quantum article (12m)
- 14:22 - Liked AI article
- 14:25 - Viewed ML course (3m, skipped)
- 14:28 - Shared quantum article

↓ ANONYMIZATION (NO SEQUENCES)

SAFE SIGNALS (AGGREGATED, NOT SEQUENCED):
- Session Duration: 28 minutes → "long_session"
- Topics Viewed (SET, not sequence): {AI, Quantum Computing, ML}
- Topic Diversity: "specialist" (3 topics, all STEM)
- Content Types (SET): {Article, Video Course}
- Format Distribution: 67% article, 33% course
- Engagement Depth: Binned by topic
  - AI: Deep read + Like interaction
  - Quantum: Deep read + Share interaction
  - ML: Light (skipped)
- Engagement Frequency: "high" (binned range)
- Topic Affinity Update:
  - AI: +0.1 (aggregated)
  - Quantum: +0.2 (aggregated)
  - ML: -0.05 (skipped = lower)

NOT CAPTURED:
- Viewing order (prevents sequence analysis)
- Exact times (only session duration bin)
- Specific articles/videos (only topics)
- Who was communicated with (only "shared" action type)

ZK PROOF: Proves signals aggregated correctly, no sequences reconstructible
```

---

## Signal Data Types & Encoding

### Safe Data Representations

```json
{
  "topic_primary": "Technology", // Categorical enum
  "topic_secondary": ["AI", "Safety"], // Set, not sequence (sorted)
  "engagement_depth": "deep_read", // Ordinal enum
  "engagement_frequency": "high", // Ordinal enum (bucket 4 of 5)
  "completion_percentile": 8, // Decile (0-10 scale)
  "format_type": "article", // Categorical enum
  "creator_type": "journalist", // Categorical enum
  "quality_affinity": "high_authority", // Categorical enum
  "freshness_bin": "recent", // Temporal bucket
  "return_frequency": "occasional", // Ordinal bucket
  "interaction_type": "view", // Action type enum
  "social_action": null, // null if no action (not collected)
  "sentiment_inferred": null, // null (not inferred - privacy risk)
  "derived_affinity_ai": 0.35, // Aggregate feature (0-1)
  "derived_consistency": "stable" // Aggregate pattern type
}
```

### Forbidden Data Representations

```json
{
  "user_id": "USER123", // ❌ Personal ID
  "timestamp": "2026-02-04T14:35:42Z", // ❌ Exact timestamp
  "article_url": "/article/ai-safety", // ❌ Specific content
  "ip_address": "192.168.1.1", // ❌ Network identifier
  "device_id": "AAID-xxx", // ❌ Device fingerprint
  "creator_name": "John Smith", // ❌ Personal name
  "location": "37.7749, -122.4194", // ❌ GPS coordinates
  "sequence": [1, 2, 3, 4, 5], // ❌ Behavioral sequence
  "exact_duration": 513, // ❌ Precise values (use buckets)
  "age_inferred": 35, // ❌ Demographic inference
  "political_view_inferred": "liberal" // ❌ Sensitive inference
}
```

---

## Summary: 40+ Safe Signals at a Glance

### Content Tags (11 signals)

✅ Primary Topic | Secondary Topics | Content Depth | Content Freshness | Content Type | Credibility Level | Sentiment Polarity | Education Value | Controversy Level | Creator Type | Domain Authority

### Interaction Types (12 signals)

✅ View Events | Engagement Depth | Scroll Interaction | Click Pattern | Hover Signals | Like/Upvote | Share/Recommendation | Comment Engagement | Bookmark/Save | Report/Flag | Subscribe Action | Download/Export

### Frequency & Intensity (15 signals)

✅ Views per Hour | Views per Day | Views per Week | Views per Month | Return Frequency | Topic Views | Topic Diversity | Format Preference | Content Depth Progression | Recent Activity Spike | Engagement Momentum | Interest Freshness | Session Recency | Completion Rate | Time Investment Intensity

### Quality & Preference (12 signals)

✅ Interactive Engagement Index | Returning Ratio | Topic Affinity Score | Format Preference Strength | Content Type Preference | Niche vs Mainstream | Engagement Volatility | Topic Consistency | Content Type Consistency | Predictability Index | Quality Preference | Complexity Appetite

### Derived & Composite (15+ signals)

✅ Topic Affinity Matrix | Content Type Preference | Depth Progression Score | Topic Exploration Pattern | Recency-Weighted Affinity | Engagement Trend | Seasonal Pattern | Session Pattern Type | Topic Correlation | Interest Diversity Index | Topic Jump Behavior | Cross-Topic Affinity | Quality Preference | Complexity Appetite | Format-Topic Specificity

**Total**: 40+ Safe, Non-Identifiable Signals
