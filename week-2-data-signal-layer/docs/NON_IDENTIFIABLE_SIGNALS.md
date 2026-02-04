# Non-Identifiable Behavioral Signals for Relevance Determination

## AI Prompt Response & Comprehensive Signal Framework

**Prompt**: "List non-identifiable behavioral signals that can be used to determine content relevance without enabling user re-identification."

---

## Core Thesis

A behavioral signal is **non-identifiable** if:

1. **K-Anonymity**: Cannot distinguish individual from 1000+ others with same signal
2. **No Linkage**: Cannot be combined with other data sources to identify individual
3. **Statistical Privacy**: No membership inference attack reveals user participation
4. **No Attribute Leakage**: Cannot infer sensitive attributes (demographics, health, etc.)

---

## Category A: Safe Behavioral Signals (40+ Signals)

### A1. Content Consumption Patterns (Non-Identifiable)

#### Safe Signals:

- **Topic interest bins** (categorical: Technology, Science, Business, etc.)
  - ✅ SAFE: "Interested in Technology" (1M+ users)
  - ❌ UNSAFE: "Interested in 'Kubernetes' + 'Bitcoin' + 'DAO'" (unique)

- **Content format preference** (categorical: video, article, podcast, interactive)
  - ✅ SAFE: "Prefers videos for Technology topics" (50K users)
  - ❌ UNSAFE: "Watched specific video ID 42857 for 8m 34s" (identifiable by timestamp + duration)

- **Content depth level** (ordinal: beginner, intermediate, advanced, expert)
  - ✅ SAFE: "Engages with advanced content" (aggregated with 10K+)
  - ❌ UNSAFE: "Completed advanced course XYZ on 2026-02-04" (timestamp identifies)

- **Content freshness preference** (binned: breaking news, recent, standard, archived)
  - ✅ SAFE: "Prefers recent content (< 7 days)" (temporal bucketed)
  - ❌ UNSAFE: "Viewed article published 2026-02-03 at 14:35 UTC" (exact temporal)

- **Authority/credibility preference** (categorical: peer-reviewed, journalist, expert, user-generated)
  - ✅ SAFE: "Trusts peer-reviewed sources" (aggregated)
  - ❌ UNSAFE: "Reads only articles from NYT at 9am" (temporal sequence + source)

#### Why These Are Safe:

- **Aggregated**: Each signal shared by 1000+ users globally
- **Categorical**: Limited cardinality prevents uniqueness
- **No temporal ordering**: No sequence analysis possible
- **No content specificity**: Only categories, not titles
- **No personal context**: No location, device, or demographic inference

### A2. Engagement Behaviors (Non-Identifiable)

#### Safe Signals:

- **Engagement frequency bins** (ordinal buckets: very_low, low, moderate, high, very_high)
  - ✅ SAFE: "High engagement user (50-200 views/month)" (10K+ in bucket)
  - ❌ UNSAFE: "User viewed 47 pages on 2026-02-04" (exact count + date)

- **Engagement depth per interaction** (ordinal: skimmed, browsed, read, deep_read)
  - ✅ SAFE: "Typically deep reads content (> 5 minutes)" (50K+ users)
  - ❌ UNSAFE: "Spent 8m 47s on article 4521 at 14:35" (exact + timestamp)

- **Interaction type distribution** (categorical set: click, scroll, share, comment, like)
  - ✅ SAFE: "Primarily clicks and scrolls (no comments)" (aggregated)
  - ❌ UNSAFE: "Clicked links 1,2,3,5,7 in order at times T1,T2,T3,T4,T5" (sequence)

- **Scroll depth distribution** (binned: minimal, moderate, substantial, full)
  - ✅ SAFE: "Typically scrolls 75-100% of content" (decile aggregation)
  - ❌ UNSAFE: "Scrolled 82% over 47 seconds" (exact measurements + time)

- **Content revisit behavior** (ordinal: one-time, occasional, regular, frequent)
  - ✅ SAFE: "Occasionally returns to topics (2-5 revisits/month)" (bucketed)
  - ❌ UNSAFE: "Returned to topic on dates: 1/30, 2/1, 2/3" (temporal pattern)

#### Why These Are Safe:

- **Binned/Ordinal**: Cannot calculate uniqueness via exact values
- **Aggregated distributions**: Always combined with 1000+ users
- **No recurrence patterns**: Cannot identify via return timing
- **No exact metrics**: Prevents inference of specific behaviors
- **No temporal precision**: Cannot reconstruct behavioral timeline

### A3. Topic Affinity & Interest Signals (Non-Identifiable)

#### Safe Signals:

- **Topic engagement affinity (binned)** (ordinal: low, medium_low, medium, medium_high, high)
  - ✅ SAFE: "Medium-high affinity for AI topics" (5K+ users in bucket)
  - ❌ UNSAFE: "Engagement score 0.847 for AI specifically" (high precision)

- **Topic diversity level** (ordinal: specialist, balanced, generalist)
  - ✅ SAFE: "Generalist (interested in 8+ topics)" (100K+ users)
  - ❌ UNSAFE: "Interested in: [AI, Crypto, Philosophy, K-pop, Cooking, Sports, etc.]" (unique set)

- **Topic progression** (categorical: same_topics, gradual_evolution, rapid_shifts)
  - ✅ SAFE: "Topic interests stable over time" (aggregated pattern)
  - ❌ UNSAFE: "Topic sequence: AI→ML→DL→Transformers→LLM" (learning path)

- **Cross-topic correlations** (categorical: related_interests, divergent_interests, uncorrelated)
  - ✅ SAFE: "Interested in both AI and Science (correlated topics)" (50K+ pairs)
  - ❌ UNSAFE: "Unique combo: [Art History + Quantum Physics + Urban Gardening]" (identifies)

- **Niche vs mainstream preference** (categorical: mainstream_only, biased_mainstream, balanced, biased_niche, niche_only)
  - ✅ SAFE: "Balanced between mainstream and niche content" (100K+ users)
  - ❌ UNSAFE: "Engages only with obscure topics: [X, Y, Z]" (unique interest set)

#### Why These Are Safe:

- **Bucketed affinity**: Prevents score-based uniqueness
- **Set-based topics**: Not ordered sequences (unordered = less identifying)
- **Population-level patterns**: Cannot isolate individual progression
- **Categorical classifications**: Binned, not continuous
- **Aggregated statistics**: Correlations computed across cohorts

### A4. Quality & Authority Preferences (Non-Identifiable)

#### Safe Signals:

- **Credibility source preference** (categorical: peer_reviewed, journalistic, expert_independent, user_generated)
  - ✅ SAFE: "Prefers peer-reviewed sources" (100K+ users)
  - ❌ UNSAFE: "Only reads Nature, Science, Cell papers from 2024-2025" (narrow + temporal)

- **Creator type affinity** (categorical: institutional, professional, academic, user_generated)
  - ✅ SAFE: "Prefers institutional creators" (1M+ users)
  - ❌ UNSAFE: "Watches only @JohnSmith, @AliaSarah, @TechGuru" (named creators)

- **Domain authority preference** (bucketed: low, medium, high, very_high)
  - ✅ SAFE: "Engages with high-authority domains" (50K+ users)
  - ❌ UNSAFE: "Only visits nytimes.com, theverge.com, arxiv.org" (specific sites)

- **Content polarity preference** (categorical: positive, balanced, critical, negative)
  - ✅ SAFE: "Prefers balanced/neutral content" (aggregated)
  - ❌ UNSAFE: "Only engages with critical/negative political content" (ideological profiling)

#### Why These Are Safe:

- **Creator agnostic**: Focuses on creator TYPE, not specific creators
- **Domain categorical**: Aggregated by authority tier, not specific URLs
- **Content-level tags**: Editorial classification, not personal inference
- **Broad buckets**: Prevents fine-grained profiling
- **No political inference**: Avoids sensitive attribute leakage

### A5. Interaction Quality & Engagement Signals (Non-Identifiable)

#### Safe Signals:

- **Completion/consumption rate** (decile: 0-10%, 10-20%, ..., 90-100%)
  - ✅ SAFE: "Usually completes 70-80% of content" (decile aggregation)
  - ❌ UNSAFE: "Completed 78.3% of content X at time T" (exact + timestamp)

- **Return frequency** (categorical: one_time, occasional, regular, frequent)
  - ✅ SAFE: "Frequently returns to topics (20+ times/month)" (bucketed)
  - ❌ UNSAFE: "Returned on dates 1/2, 1/7, 1/15, 2/1, 2/4" (temporal pattern)

- **Social engagement type** (categorical set: likes, shares, comments, bookmarks)
  - ✅ SAFE: "Typically likes and shares (no comments)" (action type distribution)
  - ❌ UNSAFE: "Liked article 4521, shared to person Y, commented 'This is great'" (content + identity)

- **Session pattern type** (categorical: daily_constant, bursty, episodic, dormant)
  - ✅ SAFE: "Episodic engagement (intense sessions, infrequent)" (pattern type)
  - ❌ UNSAFE: "Active 2026-02-01 to 02-03, then dormant 02-04 to 02-20" (temporal sequence)

#### Why These Are Safe:

- **Decile/percentile**: Coarsen continuous values into 10 buckets
- **Categorical patterns**: Not individual metrics or sequences
- **Set-based**: Unordered interaction types, no sequence
- **Aggregated distribution**: Pattern types shared by 1000+ users

---

## Category B: Dangerous Signals to AVOID (❌ Re-Identification Risk)

### B1. Temporal Precision (NEVER CAPTURE)

```
❌ Exact timestamps: "2026-02-04 14:35:42.123 UTC"
❌ Minute-level precision: Only "afternoon" or "evening"
❌ Day-of-week patterns: No "active Tuesdays"
❌ Sequential timestamps: No ordered action timeline
❌ Time intervals: No "2m 34s between actions"

✅ SAFE: Time bins (hour ranges, days ago buckets, weekly patterns binned)
```

### B2. Content Specificity (NEVER CAPTURE)

```
❌ Article titles: "Top 10 AI Safety Risks in 2026"
❌ Video URLs: "https://youtube.com/watch?v=XxYyZz"
❌ Specific creators: "@JohnSmith", "Dr. Jane Doe"
❌ Exact domains: "nytimes.com", "github.com"
❌ Document IDs: "Document_4521", "Course_ID_892"

✅ SAFE: Content categories (Topics, Content types, Creator types, Domain tiers)
```

### B3. Personal Identifiers (NEVER CAPTURE)

```
❌ User IDs: "user_123456", "session_abc"
❌ Email addresses: Any PII
❌ Phone numbers: Any contact info
❌ Device IDs: AAID, IDFA, MAC address, fingerprint
❌ IP addresses: Any network identifier
❌ Biometric data: Any fingerprint, face, iris

✅ SAFE: Ephemeral session tokens (< 1 hour lifetime, no reuse)
```

### B4. Behavioral Sequences (NEVER CAPTURE)

```
❌ View sequences: Article A → Article B → Article C
❌ Click paths: Click 1 → Click 2 → Click 3 → Link
❌ Search sequences: Query 1 → Query 2 → Query 3
❌ Feature vectors with position: "Viewed topics in order: [AI, Crypto, Safety]"
❌ Learning paths: "Completed: Beginner → Intermediate → Advanced"

✅ SAFE: Unordered sets (topics viewed in session as SET, not list)
✅ SAFE: Categorical patterns ("topic progression: stable vs evolving")
```

### B5. Demographic/Sensitive Inferences (NEVER MAKE)

```
❌ Age estimation: "Probably 25-34 based on content"
❌ Gender inference: "Likely female based on engagement"
❌ Location inference: "Probably from San Francisco based on timezone"
❌ Political profiling: "Likely conservative based on topics"
❌ Health inference: "Probably diabetic based on searches"
❌ Financial status: "Probably high-income based on interests"

✅ SAFE: NO demographic inference of any kind
```

### B6. Unique Combinations (AVOID HIGH-SPECIFICITY COMBOS)

```
❌ "Interested in: [AI, Quantum, Blockchain, Philosophy, K-pop]"
   → High specificity, likely unique

❌ "Views: [AI papers + K-pop videos + Food blogs]"
   → Unusual combination, may identify

❌ "Format: 95% video, 3% article, 2% audio + Topics: [Niche1, Niche2]"
   → Specific distribution, identifying

✅ SAFE: "Interested in Technology + Science (broad categories)"
✅ SAFE: "Prefers videos (50K+ users with preference)"
✅ SAFE: "Balanced format distribution (100K+ users)"
```

---

## Category C: Signal Engineering Best Practices

### Rule 1: Aggregate Threshold

```
BEFORE transmitting any signal:
✓ Ensure signal aggregated with 1000+ users minimum
✓ Use population-level statistics (means, distributions)
✓ Never expose individual values
✓ Always round/bucket continuous values

VALIDATION:
- If signal cardinality > population/1000: Too specific, reject
- If buckets > 10: Too fine-grained, merge buckets
- If precision > integer: Too exact, bin into ranges
```

### Rule 2: Temporal Protection

```
TIMESTAMPS:
✗ Store exact time: 2026-02-04 14:35:42
✗ Store minute precision: 14:35
✗ Store time sequences: [14:35, 14:40, 14:47]

✓ Store time buckets: "afternoon" (14:00-18:00)
✓ Store recency: "2-7 days ago"
✓ Store categorical pattern: "evening-biased"
✓ Store temporal aggregates: "avg 3x per week"

ENCODING:
- Hour level: 24 buckets (too many? merge to 6: night/dawn/morning/afternoon/evening/night)
- Day level: 7 buckets (Monday, Tuesday, ...) then aggregate
- Week level: 4 buckets (week1-4 of month)
- Month level: 12 buckets (January, February, ...)
- Recency: 6 buckets (today, yesterday, 2-7d ago, 1-4 weeks, 1-3 months, 3+ months)
```

### Rule 3: Content Protection

```
CONTENT MAPPING:
Article ID 4521 → (Topic: AI, Depth: Intermediate, Creator Type: Journalist)
               → (Freshness: Recent, Format: Article, Authority: Tier 2)

Video ID 8892 → (Topic: Science, Depth: Advanced, Creator Type: Academic)
             → (Freshness: Standard, Format: Video, Authority: High)

TRANSMISSION:
✗ Send: article_id_4521
✓ Send: {topic: AI, depth: intermediate, creator_type: journalist}

BENEFIT:
- Attacker cannot trace signal back to specific content
- Cannot identify via content title/description
- Multiple contents map to same signal (k-anonymity)
```

### Rule 4: Combination Safety

```
SAFE COMBINATIONS (Aggregated Independently):
- Topic bucket + Engagement frequency bucket
  → "AI + High engagement" (1000+ users have this combo)

- Format preference + Quality preference
  → "Prefers videos from high authority" (5000+ users)

- Depth level + Diversity level
  → "Advanced content + Specialist" (2000+ users)

UNSAFE COMBINATIONS (Overly Specific):
- Topic bucket + Format + Depth + Freshness + Creator type
  → "AI + Video + Advanced + Recent + Academic"
  → Likely unique, too specific

RULE:
- Each individual feature: aggregated with 1000+ users ✓
- Each feature pair: aggregated with 1000+ users ✓
- Feature triplet: verify aggregation with 1000+ users
- Feature 4+: likely too specific, use with caution

TESTING:
- Before using feature combo, query: COUNT(*) WHERE all_features
- If COUNT < 1000: Reject combination as too identifying
- If COUNT >= 1000: Safe to use
```

### Rule 5: Noise Injection Strategy

```
DIFFERENTIAL PRIVACY LAYER (Additional Safety):
Add noise to aggregate statistics before release:

True count: 47,382 users interested in AI
DP noise (ε=0.1): ±4,738 (±10% noise)
Reported count: 47,382 + random(-4738, 4738) = 44,500

PROPERTY:
- Attacker cannot determine exact count
- But cannot determine if individual X is in dataset
- Noise ensures privacy even if features are specific

WHEN TO USE:
- Publishing aggregate statistics ✓
- Feeding aggregates to ML model ✓
- Exposing feature distributions ✓
- Internal analytics/dashboards ✓
```

---

## Category D: Validation Checklist (Before Deploying Signals)

For each signal, answer:

### Signal Definition

- [ ] Can you define signal in one sentence?
- [ ] Does it avoid personal data (no names, IDs, locations)?
- [ ] Is it categorical/ordinal (not continuous)?
- [ ] Can it be computed from content tags, not personal data?

### Re-Identification Risk

- [ ] Is signal cardinality large (1000+ values in each bucket)?
- [ ] Can signal be combined with public data to identify user?
- [ ] Would attacker learn if user in dataset from this signal?
- [ ] Could demographic inference be made from this signal?

### Aggregation Validation

- [ ] Is signal ALWAYS aggregated with 1000+ other users?
- [ ] Are raw individual values NEVER stored or transmitted?
- [ ] Can signal be disaggregated back to individuals? (Should be NO)
- [ ] Is signal used only from aggregate statistics?

### Temporal Safety

- [ ] Does signal avoid exact timestamps?
- [ ] Are temporal values binned (no minute/second precision)?
- [ ] Can sequence of signals reveal behavioral timeline? (Should be NO)
- [ ] Is recency expressed in buckets (not exact dates)?

### Content Safety

- [ ] Does signal avoid specific content (titles, URLs)?
- [ ] Is content identified only by category tags?
- [ ] Would knowing signal allow guessing specific content? (Should be NO)
- [ ] Is creator identified only by type, not name?

### Composition Safety

- [ ] Have you tested combination with other signals?
- [ ] When combined, are 1000+ users in resulting bucket?
- [ ] Would combination enable fine-grained profiling?
- [ ] Is combination used only for aggregate statistics?

---

## Summary: Complete Signal List (Non-Identifiable Safe Signals)

### Topics & Content (13 signals) ✅

1. Primary topic category (binned)
2. Secondary topics (top-3, unordered set)
3. Content depth level (ordinal)
4. Content freshness bucket (temporal binned)
5. Content format type (categorical)
6. Credibility source preference (categorical)
7. Creator type affinity (categorical)
8. Domain authority tier (bucketed)
9. Content polarity preference (categorical)
10. Education value preference (categorical)
11. Controversy preference (categorical)
12. Topic freshness (rolling window)
13. Topic diversity level (ordinal)

### Behaviors & Engagement (15 signals) ✅

14. Engagement frequency bin (ordinal 1-5)
15. Engagement depth per action (ordinal)
16. Interaction type distribution (set, unordered)
17. Scroll depth percentile (decile)
18. Content completion rate (decile)
19. Return frequency bucket (ordinal)
20. Session pattern type (categorical)
21. Interaction velocity (actions/min, binned)
22. Click pattern type (ordinal)
23. Hover engagement (duration binned)
24. Social action type distribution (set)
25. Quality engagement index (ordinal)
26. Re-engagement frequency (ordinal)
27. Format preference ratio (binned)
28. Recency weight of engagement (ordinal)

### Affinity & Preference (12+ signals) ✅

29. Topic affinity score (ordinal 1-5)
30. Format preference strength (ordinal)
31. Quality preference strength (ordinal)
32. Complexity appetite level (ordinal)
33. Niche vs mainstream preference (categorical)
34. Engagement consistency (categorical)
35. Content exploration pattern (categorical)
36. Topic correlation pattern (categorical)
37. Interest diversity index (ordinal)
38. Predictability level (categorical)
39. Cross-topic affinity (categorical)
40. Creator preference distribution (categorical)

### Derived Features (10+ signals) ✅

41. Topic affinity matrix (per-topic ordinal)
42. Recency-weighted affinity (per-topic ordinal)
43. Engagement trend (categorical)
44. Format-topic specificity (per-topic distribution)
45. Quality-topic specificity (per-topic ordinal)
46. Complexity-topic specificity (per-topic ordinal)
47. Behavioral similarity cohort (categorical)
48. Interest evolution stage (categorical)
49. Content consumption style (categorical)
50. Recommendation potential (categorical)

---

## Why This Approach is Safe

### 1. No Individual Identification

- **K-Anonymity**: Every signal aggregates 1000+ users
- **Linkage safety**: Signals don't link to other datasets
- **No re-identification**: Cannot reverse from signal to user

### 2. No Sensitive Attributes

- **No demographics**: Age, gender, location inference blocked
- **No health/financial**: No inference of sensitive categories
- **No behavioral tracking**: No sequence analysis

### 3. Aggregation Irreversibility

- **One-way aggregation**: Cannot disaggregate back to individuals
- **Population-only statistics**: Individual values discarded
- **No storage of raw data**: Only aggregates retained

### 4. Meaningful for Relevance

- **Topic signals**: Drive content categorization
- **Engagement signals**: Measure relevance indicators
- **Quality signals**: Personalize by user preference
- **40+ signals**: Rich feature space for ML models

### 5. Privacy-Preserving by Design

- **No personal data needed**: Signals computed from content + actions
- **Temporal protection**: Binned, not sequences
- **Content protection**: Categories only, not titles
- **Aggregation guarantee**: Math ensures privacy

---

## Conclusion

**Non-identifiable behavioral signals** can determine content relevance through:

- **Topic preferences** (aggregate engagement with categories)
- **Interaction behaviors** (aggregate action types)
- **Engagement patterns** (binned frequency + intensity)
- **Quality/preference signals** (aggregated engagement with signal types)
- **Derived features** (statistical compositions of above)

These 50+ signals **cannot re-identify users** because they are:

1. Aggregated with 1000+ others
2. Binned (not exact values)
3. Categorical (limited cardinality)
4. Free of temporal sequences
5. Free of personal identifiers
6. Free of sensitive attributes
7. Mathematically irreversible

The framework enables **personalized relevance without privacy loss**.
