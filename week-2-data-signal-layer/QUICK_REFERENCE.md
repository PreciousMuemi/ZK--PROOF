# Quick Reference: Safe Signals Cheat Sheet

## Week 2 Data & Signal Layer

---

## 🚨 The Golden Rule

**Every signal must be safe to publish without revealing user identity.**

---

## ✅ SAFE Signals (Binned & Aggregated)

### Topic Preferences

```
✅ Primary topic: "Technology" (1M+ users)
✅ Topics as set: {AI, Security} (unordered)
✅ Creator type: "Journalist" (NOT "John Smith")
✅ Authority tier: 2 (1-4 bucket, NOT score)
✅ Content depth: "Advanced" (categorical)

❌ Exact content: "Top 10 AI Safety Risks"
❌ Creator name: "John Smith"
❌ Authority score: 0.847
❌ Topic sequence: [AI, Security, Quantum] → order
```

### Engagement Metrics

```
✅ Duration binned: "5-10 minutes" (not 8m 47s)
✅ Scroll depth decile: 8 (80th percentile)
✅ Completion % decile: 7 (70th percentile)
✅ Frequency bucket: "high" (4 of 5)
✅ Return frequency: "regular" (4 of 4)

❌ Exact duration: 527 seconds
❌ Exact percentage: 82.3%
❌ Exact count: 47 views
❌ Exact timestamp: 2026-02-04 14:35:42 UTC
```

### Interaction Patterns

```
✅ Interaction set: {View, Click, Scroll} (order doesn't matter)
✅ Social action type: "Like" (NOT who was liked)
✅ Interaction frequency: "Moderate" (binned)
✅ Session pattern: "Bursty" (categorical)

❌ Interaction sequence: View → Click → View → Click (order)
❌ Timeline: "Clicked at 14:35, 14:40, 14:47"
❌ Click targets: [Link1, Link2, Link3]
❌ Share destination: "Shared to John"
```

### Quality Preferences

```
✅ Quality affinity: "High Authority" (categorical)
✅ Complexity preference: "Advanced Seeking" (ordinal)
✅ Format preference strength: "Strong" (categorical)
✅ Format ratio: "60% video, 40% article" (aggregate)

❌ Specific authority score: 0.847
❌ Individual complexity: "Exactly 3.5 out of 5"
❌ Format exact %: "63.2% video"
```

---

## ❌ UNSAFE Signals (REJECT IMMEDIATELY)

### Personal Identifiers

```
❌ user_id: "USER-12345"
❌ email: "john@example.com"
❌ phone: "555-123-4567"
❌ device_id: "AAID-xyz"
❌ ip_address: "192.168.1.1"
❌ session_id: (if linked to user)
```

### Behavioral Sequences

```
❌ View sequence: [article_1, article_2, article_3]
❌ Click timeline: [(time1, click_link_1), (time2, click_link_2)]
❌ Learning path: Beginner → Intermediate → Advanced
❌ Topic path: AI → ML → Deep Learning → Transformers
```

### Demographic/Sensitive Data

```
❌ age: 35
❌ gender: "Female"
❌ location: "San Francisco, CA"
❌ health_condition: "Diabetes"
❌ political_view: "Progressive"
❌ income: "$100K+"
```

### Exact/Continuous Values

```
❌ Duration: 527 (use: "5-10m bucket")
❌ Percentage: 82.3 (use: decile 8)
❌ Count: 47 (use: "high" bucket)
❌ Score: 0.847 (use: "4 out of 5")
❌ Timestamp: 2026-02-04 14:35:42 (use: "afternoon" bin)
```

### Specific Content

```
❌ Article title: "Top 10 AI Safety Risks"
❌ Video URL: "youtube.com/watch?v=XyZ"
❌ Domain: "nytimes.com"
❌ Creator: "John Smith"
❌ Product SKU: "PROD-12345"
```

---

## 🔐 Privacy Test Checklist

For each signal, ask:

### Test 1: K-Anonymity (k=1000)

```
Question: Can this signal identify < 1000 users?
Answer:   NO ✅ (aggregates with 1000+ others)

Example:
❌ "Watched exactly this sequence of 5 videos" → Few users
✅ "Video format preference: 60% distribution" → 100K users
```

### Test 2: Linkage Attack

```
Question: Can signal + public data re-identify someone?
Answer:   NO ✅ (binned, aggregated, no linkage)

Example:
❌ [Specific videos watched] + Wikipedia = Identify
✅ [Topic categories] + 100K others = Cannot identify
```

### Test 3: Temporal Safety

```
Question: Can signal reveal behavioral patterns?
Answer:   NO ✅ (no sequences, only binned time)

Example:
❌ Action timeline: 14:35, 14:40, 14:47 → Pattern visible
✅ Time bin: "afternoon" → No pattern info
```

### Test 4: No Sensitive Attributes

```
Question: Can demographics be inferred?
Answer:   NO ✅ (no inference signals present)

Example:
❌ Topics [Birth Control, Antidepressants] → Health inferred
✅ Topics [Health, Science] → No inference
```

---

## 📊 Signal Binning Strategy

### Time Binning (Temporal)

```
Hour level:
- morning (06:00-12:00)
- afternoon (12:00-18:00)
- evening (18:00-24:00)
- night (00:00-06:00)

Day level:
- today
- yesterday
- 2-7 days ago
- 1-4 weeks ago
- 1-3 months ago
- 3+ months ago

Week level:
- week_1, week_2, week_3, week_4 (of month)

(NOT: exact timestamp, minute precision, or sequences)
```

### Engagement Binning (Numeric)

```
5-bucket scale (frequency):
- 1: Very Low (0-10 events)
- 2: Low (11-30)
- 3: Moderate (31-100)
- 4: High (101-200)
- 5: Very High (200+)

4-bucket scale (depth):
- Skimmed (0-30 seconds)
- Browsed (30-60 seconds)
- Read (60-180 seconds)
- Deep Read (180+ seconds)

Decile scale (percentile):
- 1: 0-10th percentile
- 2: 10-20th percentile
- ...
- 10: 90-100th percentile

(NOT: exact values, continuous floats)
```

### Topic Binning (Categorical)

```
Top-level categories (10 only):
- Technology
- Science
- Business
- Entertainment
- Health
- Politics
- Sports
- Culture
- Education
- Other

Secondary (max 3, unordered set):
- {AI, Security, Privacy}  ✅ (order doesn't matter)
- NOT: [AI, Security, Privacy] ❌ (suggests sequence)

(NOT: specific subtopics, named creators)
```

---

## 🔄 Validation Pipeline (7 Layers)

```
Input Signal
    ↓
[1] SCHEMA VALIDATION
    ├─ JSON schema check
    ├─ Required fields present
    └─ No unknown fields

[2] CARDINALITY CHECK
    ├─ No continuous values
    ├─ All numeric binned
    └─ Limited buckets (3-10)

[3] TEMPORAL SANITIZATION
    ├─ Only time bins allowed
    ├─ No exact timestamps
    └─ No temporal sequences

[4] PII DETECTION
    ├─ Scan for user IDs
    ├─ Email pattern check
    ├─ Name detection
    └─ Device ID detection

[5] SEQUENCE DETECTION
    ├─ No ordered arrays
    ├─ No timelines
    └─ Sets only

[6] AGGREGATION VERIFICATION
    ├─ Cohort size >= 1000
    └─ Database query check

[7] K-ANONYMITY ASSESSMENT
    ├─ Single features >= 1000
    └─ Feature pairs >= 1000

    ↓
✅ ACCEPTED or ❌ REJECTED
```

---

## 📋 Signal Checklist Before Sending

```
BEFORE TRANSMITTING SIGNAL:

Schema:
☐ JSON matches signal_schema.json
☐ All required fields present
☐ No extra/unknown fields
☐ All enums exact values

Values:
☐ All numeric values binned
☐ No floating point numbers
☐ No unbounded counts
☐ No continuous values

Temporal:
☐ Time is binned (morning/afternoon/evening/night)
☐ No exact timestamp anywhere
☐ No date patterns (YYYY-MM-DD)
☐ No time sequences

PII:
☐ No user_id / account_id
☐ No email address
☐ No phone number
☐ No creator name
☐ No device identifier
☐ No location coordinates
☐ No demographic info

Sequences:
☐ No ordered arrays
☐ No click paths
☐ No view timelines
☐ No action sequences

Aggregation:
☐ Ready to aggregate with 1000+ users
☐ No individual values exposed
☐ K-anonymity = 1000 verified

→ READY TO SEND ✅
```

---

## 💾 What Gets Stored (Retention Policy)

```
RAW SIGNALS (Individual):
├─ Retention: 0 hours
├─ Storage: Client-side memory only
└─ Deletion: After anonymization

ANONYMIZED SIGNALS (Ephemeral):
├─ Retention: 24 hours
├─ Storage: Redis (in-memory)
└─ Deletion: Automatic TTL

AGGREGATED STATISTICS:
├─ Retention: 30 days
├─ Storage: PostgreSQL (aggregates only)
└─ Deletion: Scheduled job

ANALYTICS AGGREGATES:
├─ Retention: Permanent
├─ Storage: Data warehouse (anonymized)
└─ Comment: Cannot be disaggregated

AUDIT LOGS:
├─ Retention: 7 days (proof logs)
├─ Storage: Encrypted logs
└─ Deletion: Automated
```

---

## 🎓 Quick Example

### Raw Data (Never Transmitted)

```
User ID: 12345
Timestamp: 2026-02-04 14:35:42.876 UTC
Article: /articles/ai-safety-2026
Duration: 8m 47s
Scroll: 82%
Clicks: 3
Location: San Francisco
```

### Safe Signal (Transmitted)

```
{
  "signal_timestamp_bin": "afternoon",
  "primary_topic": "Technology",
  "secondary_topics": ["AI"],
  "content_depth": "Intermediate",
  "engagement_depth": "Deep_Read",
  "scroll_depth_decile": 8,
  "creator_type": "Journalist",
  "interaction_set": ["View", "Click"]
}
```

### After Validation ✅

```
✅ Schema: Valid
✅ Cardinality: Binned
✅ Temporal: Binned
✅ PII: None
✅ Sequences: None
✅ Aggregation: 50K users
✅ K-anonymity: Verified

→ ACCEPTED & AGGREGATED
```

---

## ⚠️ Common Mistakes (Avoid These!)

```
❌ Storing exact duration (use buckets)
   → "527 seconds" → ✅ "5-10m"

❌ Storing exact timestamp (use bins)
   → "2026-02-04 14:35:42" → ✅ "afternoon"

❌ Storing specific content (use categories)
   → "Top 10 AI Safety Risks" → ✅ "Technology/AI"

❌ Storing creator name (use type)
   → "John Smith" → ✅ "Journalist"

❌ Storing behavioral sequence (use set)
   → [article_1 → article_2 → article_3] → ✅ {article_topics}

❌ Storing exact percentages (use deciles)
   → "82.3%" → ✅ "decile 8"

❌ Storing multiple signals per user (aggregate first)
   → User: [signal1, signal2, signal3] → ✅ Aggregate cohort

❌ Linking to user ID (make ephemeral session)
   → user_id + signal → ✅ session_id (1hr TTL) + signal

❌ Storing demographic inference (never infer)
   → Age inferred from topics → ✅ (no inference)

❌ Storing location coordinates (use region)
   → "37.7749, -122.4194" → ✅ "Pacific region"
```

---

## 📞 When in Doubt

1. **Is this a continuous number?** → BIN IT
2. **Is this linked to a user?** → ANONYMIZE IT
3. **Is this a sequence/timeline?** → AGGREGATE IT
4. **Is this personally identifiable?** → REJECT IT
5. **Does this enable sensitive inference?** → REMOVE IT

**Default**: If uncertain, it's probably not safe. Reject and redesign.

---

## 📚 Key Documents Reference

| Question                    | Document                      |
| --------------------------- | ----------------------------- |
| What signals can we use?    | SAFE_SIGNALS_SPECIFICATION.md |
| How do we validate signals? | SIGNAL_VALIDATION_RULES.md    |
| How do we implement this?   | IMPLEMENTATION_GUIDE.md       |
| Why is this safe?           | NON_IDENTIFIABLE_SIGNALS.md   |
| What's the overview?        | README.md                     |

---

**Remember**: Privacy by design means signals are safe FROM THE START, not made safe later.
