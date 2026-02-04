# Privacy Boundaries & Protection Framework

## Zero-Knowledge Proof AI Relevance Engine

---

## Executive Summary

Privacy boundaries define:

1. **What data we NEVER capture**
2. **How we protect data we DO capture**
3. **Technical & organizational guarantees**
4. **Legal & compliance obligations**

---

## Data We Never Capture

### Strict Prohibition Tier (System Design Level)

#### User Identification Data

```
❌ NOT CAPTURED:
├─ User ID / Account number
├─ Email address
├─ Phone number
├─ Social Security Number / Tax ID
├─ Passport / Governmental IDs
├─ Credit card / Payment details
└─ Biological identifiers (fingerprint, iris scan)

ENFORCEMENT:
- Client-side code stripped of all ID collection
- Zero-knowledge proof validation rejects any ID embedding
- Backend systems have no user ID field in schema
- Code review checklist includes "no ID capture" verification
```

#### Location Data

```
❌ NOT CAPTURED:
├─ GPS coordinates
├─ IP address (full)
├─ Cell tower data
├─ Home/work address
├─ WiFi SSID/MAC address
├─ Geofence entries/exits
└─ Travel patterns / movement sequences

PERMITTED (Privacy-Preserving):
├─ Top-level geographic region (country-level only)
├─ Timezone (if needed for feature engineering)
└─ Region used only for: CDN routing, fraud detection

ENFORCEMENT:
- VPN/proxy layer strips IP before processing
- Geographic region inferred from user consent only
- No location history retention
- Quarterly audits verify no location tracking
```

#### Device & Hardware Identifiers

```
❌ NOT CAPTURED:
├─ Device ID / UDID
├─ MAC address
├─ Android Advertising ID (AAID)
├─ Apple IDFA
├─ Bluetooth MAC
├─ Browser fingerprint (user-agent, canvas, WebGL)
├─ Hardware serial numbers
└─ SIM card ID

RATIONALE:
- Device IDs enable user re-identification across sessions
- Hardware fingerprints persist across app reinstalls
- Enables cross-device tracking (privacy violation)

ENFORCEMENT:
- Client SDK blocks fingerprinting libraries
- Browser cookies explicitly disabled for personalization
- Session tokens ephemeral (< 1 hour lifetime)
```

#### Personal Attributes

```
❌ NOT CAPTURED:
├─ Age / Date of birth
├─ Gender / Gender identity
├─ Race / Ethnicity
├─ Religion / Political views
├─ Sexual orientation
├─ Health conditions / Medical history
├─ Psychological profile / Personality
├─ Income / Financial status
├─ Education level
├─ Marital/Family status
└─ Genetic information

RATIONALE:
- Sensitive personal attributes prone to discrimination
- GDPR special category data (requires explicit consent)
- Cannot be anonymized reliably (linkage attacks possible)

ENFORCEMENT:
- No form fields for these attributes
- Backend validation rejects any inferences of these
- Model training data excludes any correlation features
```

#### Behavioral Sequences

```
❌ NOT CAPTURED:
├─ Exact timestamps of actions (only time bins)
├─ Sequence of clicks (e.g., path A→B→C)
├─ Search query history
├─ Website visitation order
├─ Content viewing sequence
├─ Purchase order history
├─ Communication with other users
└─ Login/logout times

RATIONALE:
- Sequences are highly re-identifiable
- Even anonymized sequences can be linked via timing attacks
- Enable behavior re-construction of individuals

EXAMPLE OF WHAT WE DON'T DO:
❌ "User logged in, clicked AI article, then scrolled comment thread"
✅ "User has interest in: [AI], engagement level: [High]"

ENFORCEMENT:
- Client-side aggregation before transmission
- Server rejects messages with > 1 action
- Database schema has no sequential ID field
```

#### Content Specificity

```
❌ NOT CAPTURED:
├─ Specific article titles read
├─ Exact video names watched
├─ Specific product names searched
├─ Comment text / Message content
├─ Query strings / Search terms
├─ Code snippets viewed
├─ Document content interacted with
└─ Specific product SKUs purchased

PERMITTED:
├─ Topic category (e.g., "Technology/AI")
├─ Content type (e.g., "video", "article")
├─ Engagement metrics only (time spent, completion %)
└─ Derived features (topic affinity scores)

ENFORCEMENT:
- Regex filter strips content URLs on client
- Content hashing removes identifiable text
- Server validates incoming data schema
```

---

## Data We DO Capture (With Protection)

### Tier 1: Minimized & Aggregated (Highest Protection)

#### Engagement Metrics

```
CAPTURED:
├─ View duration (BINNED into 7 buckets, not exact seconds)
├─ Scroll depth (QUANTIZED into deciles)
├─ Interaction count (AGGREGATED hourly, not per-session)
├─ Return frequency (BINNED into ranges: 1x, 2-5x, 5-10x, 10+x)
└─ Completion ratio (ROUNDED to 10% buckets)

PROTECTION MECHANISMS:
├─ Binning: Reduces cardinality, masks exact behavior
├─ Aggregation: Combines with 1000+ other users
├─ Noise: ± 10% Laplace noise added to counts
├─ Temporal decay: Signals older than 30 days discarded
└─ No linking: Metrics isolated from any identifier

RETENTION:
├─ Raw: 0 hours (discarded after processing)
├─ Aggregated: 30 days (then deleted)
└─ Analytics: Anonymously forever (cannot disaggregate)
```

#### Topic Preferences

```
CAPTURED:
├─ Top-level topic category engagement (e.g., "Technology")
├─ Secondary topics (top 3 only, anonymized with others)
├─ Topic diversity measure (binned count)
└─ Topic recency (within 30-day rolling window)

PROTECTION MECHANISMS:
├─ K-anonymity: Topic never alone identifies user
├─ Hierarchical suppression: Only top-level categories kept
├─ Aggregation: Topics always processed with 1000+ others
└─ Temporal windowing: 30-day lookahead window, older discarded

EXAMPLE:
Raw data: User12345 viewed: [AI article, ML paper, Safety blog, Ethics thread]
Protected: Topic interest in: [AI/Tech] (aggregated with 50K other users)
```

#### Interaction Types

```
CAPTURED:
├─ Interaction modality (click, scroll, play, share, comment)
├─ Engagement depth (surface, moderate, deep)
├─ Content type preference (video vs article vs audio)
└─ Social activity (share, like, comment — TYPE ONLY)

PROTECTION MECHANISMS:
├─ Categorical encoding: No numeric identity info
├─ Aggregation: Always with 10K+ others
├─ No content linkage: Type isolated from specific content
└─ No sequence: Individual interactions discarded after categorization

NOT CAPTURED:
├─ Comment TEXT (only the fact of commenting)
├─ WHO receives shares (share target identity hidden)
├─ WHO sent messages (sender identity hidden)
└─ Emotional sentiment (even if inferred from interaction)
```

---

### Tier 2: Intermediate Protection (Time-Limited)

#### Session-Specific Aggregations

```
CAPTURED:
├─ Session engagement score (aggregate of this session only)
├─ Topics in this session (binned, aggregated)
├─ Interaction methods used in session
└─ Session duration (binned)

PROTECTION MECHANISMS:
├─ Session isolation: No cross-session linking
├─ Ephemeral: Discarded after 1-24 hours
├─ No identification: Session token ≠ user identity
└─ Hashed timestamps: Session time randomized ±15 minutes

RETENTION:
├─ Active memory: 24 hours max
├─ Backup: 7 days (encrypted, then deleted)
└─ Never aggregated with other users' sessions
```

#### Derived Feature Values

```
CAPTURED:
├─ Topic affinity score (aggregate statistic)
├─ Engagement velocity (interactions per day, bucketed)
├─ Format preference ratio (%, rounded)
├─ Recency decay weights (computed, not stored)
└─ Feature confidence scores (statistical)

PROTECTION MECHANISMS:
├─ Mathematical aggregation: No individual values retained
├─ Noise injection: DP noise added to feature values
├─ Feature importance: Only top-K features retained
└─ Model feature: Features used for inference only, not exposed

RETENTION:
├─ For inference: Until model update (30 days)
├─ For metrics: 90 days (aggregated only)
└─ Raw features: Immediately deleted
```

---

### Tier 3: High-Risk Data (Strict Controls)

#### Proof Verification Logs

```
CAPTURED:
├─ Proof structure (to verify format validity)
├─ Verification success/failure (for QA)
├─ Proof timestamp (hashed, not precise)
└─ Error messages (if proof invalid)

PROTECTION MECHANISMS:
├─ Pseudonymization: Proof tagged with random ID, not user ID
├─ No content: Proof math logged, not the data being proved
├─ Aggregation: Success rate tracked at cohort level
├─ Encryption: Logs encrypted at rest with key rotation
├─ Access control: Only analytics team, not product team

RETENTION:
├─ Raw logs: 7 days
├─ Aggregated stats: 30 days
├─ Archived (encrypted): 1 year (for audit trail)
└─ Deletion: Certified deletion every quarter

NOT CAPTURED:
├─ Proof values (only success/failure)
├─ Any reconstructible data from proof
└─ User identities (only cohort statistics)
```

#### Error/Anomaly Detection Data

```
CAPTURED:
├─ Anomalous signal patterns (detected but not acted on)
├─ Proof validation failures (what type of failure)
├─ Rate limiting hits (how many rejected signals)
└─ System performance metrics (latency, throughput)

PROTECTION MECHANISMS:
├─ No individual linking: Anomalies not attributed to users
├─ Statistical only: Patterns aggregated across 10K+ anomalies
├─ Pseudonymized: Random cohort ID, not user ID
├─ Limited retention: 30 days maximum
└─ Access control: Only security/engineering, not analytics

NOT CAPTURED:
├─ Which users had anomalies
├─ What signals caused anomalies
├─ Temporal patterns of anomalies
└─ Attribution to devices/sessions
```

---

## Privacy Boundary Architecture

### Boundary 1: Client-Server Communication Boundary

```
CLIENT SIDE (Privacy Protected)          SERVER SIDE (Verified Processing)
┌──────────────────────────────┐        ┌──────────────────────────────┐
│ Raw User Data                │        │ Anonymized Data              │
│ ├─ User ID ✗                 │        │ ├─ User ID ✓ MISSING         │
│ ├─ Activity logs ✗           │──ZK──▶ │ ├─ Signal bundles ✓          │
│ ├─ Timestamps ✗              │ Proof  │ ├─ Aggregated only ✓         │
│ └─ Device ID ✗               │ Proves │ └─ Never disaggregated ✓     │
│                              │ valid- │                              │
│ Privacy Goal:                │ ity    │ Privacy Guarantee:           │
│ Delete immediately after     │        │ No personal data retention   │
│ ZK proof generation          │        │                              │
└──────────────────────────────┘        └──────────────────────────────┘
```

### Boundary 2: Data Retention Boundary

```
TIME
Today    +1h      +24h     +7d      +30d     +90d      Forever
│        │        │        │        │        │         │
├─Raw ───┤        │        │        │        │         (DELETED)
│ Data   │        │        │        │        │
├────────┼─ Sess──┤        │        │        │         (DELETED)
│        │ Logs   │        │        │        │
├────────┼────────┼─ Logs ─┤        │        │         (DELETED)
│        │        │ Encr   │        │        │
├────────┼────────┼────────┼─Aggr ─┤        │         (DELETED)
│        │        │        │ Stats │        │
├────────┼────────┼────────┼───────┼─Anonym─┤         (DELETED)
│        │        │        │       │ Archi- │
│        │        │        │       │ ve     │
├────────┼────────┼────────┼───────┼────────┼─Forever─────────────
│        │        │        │       │        │  (Anonymized, unable
│        │        │        │       │        │   to disaggregate)
```

### Boundary 3: Access Control Boundary

```
Data Classification → Allowed Roles → Access Mechanism

Raw User Data (NONE STORED)
├─ Engineering team
│  └─ Access: SOURCE CODE ONLY (cannot run with real data)

Aggregated Statistics
├─ Analytics team: Read-only SQL queries
├─ Product team: Published dashboards (no raw queries)
└─ ML team: Feature engineering (no user ID access)

Verification Logs
├─ Security team: Encrypted logs, pseudonymized
├─ Compliance: Audit queries (no disaggregation)
└─ NOT accessible: Product, Marketing, Sales

Sensitive Signals (Early)
├─ ML engineers: Anonymization verified
├─ Data scientists: Feature importance only
└─ NOT accessible: Anyone outside engineering
```

---

## Privacy-Preserving Guarantees by Technique

### 1. Zero-Knowledge Proofs

**Guarantee**: Signal validity without revealing signal values

```
What ZK Proves (without revealing):
✓ Signal follows schema (structure valid)
✓ Values within valid ranges
✓ No future timestamps
✓ Topic in taxonomy
✓ Engagement level reasonable

What ZK Does NOT Prove (intentionally):
✗ User identity (no ID in proof)
✗ Exact signal values (only ranges)
✗ Temporal sequences (only aggregates)
✗ Device identity
✗ Location

Privacy Property: ZERO-KNOWLEDGE
- Verifier learns: "Valid signal" (yes/no)
- Verifier learns: "This signal type" (category)
- Verifier learns: NOTHING about content/identity
```

### 2. Differential Privacy

**Guarantee**: Aggregate statistics don't reveal individual data

```
Mechanism: Laplace Noise Addition

Population: 100,000 users
True count: 47,382 users engaged with AI
DP noise (ε=0.1): ±4,738
Reported: 47,382 + (-2,847) = 44,535

Property: No attacker can distinguish:
- Person X was in dataset vs. not in dataset
- By observing outputs before/after adding person
- Even with auxiliary information
```

### 3. Data Minimization

**Guarantee**: Minimal data = smaller attack surface

```
Traditional approach:
User data → Personal info → Store everything →
Attack risk: HIGH (breach = identity theft)

Our approach:
User data → Anonymize → Bin → Aggregate →
Attack risk: MINIMAL (breach = noisy aggregate statistics)

Information loss is intentional privacy gain
```

### 4. Purpose Limitation

**Guarantee**: Data used ONLY for stated purpose

```
Signal Purpose: Generate relevance recommendations
✓ Allowed use: Train recommendation models
✓ Allowed use: Compute engagement statistics
✗ Prohibited: Marketing targeting
✗ Prohibited: Credit scoring inference
✗ Prohibited: Behavioral prediction (beyond recommendations)
✗ Prohibited: Third-party data broker sales

Enforcement:
├─ Data use agreements (contractual)
├─ Code review (technical)
├─ Compliance audits (operational)
└─ Automated schema validation (system)
```

### 5. Temporal Limitation (Data Deletion)

**Guarantee**: Data automatically deleted after purpose fulfilled

```
Timeline:

Signal arrives
    │
    ├─ Processed for inference (same day)
    │
    ├─ Aggregated into statistics (7 days)
    │
    ├─ Anonymized for long-term analytics (30 days)
    │
    └─ Securely deleted (verified deletion, 90 days max)

No indefinite retention
No "just in case" storage
No secondary use archive
```

---

## Privacy Incidents & Response

### Incident Classification

| Incident Type                | Definition                  | Response SLA                               |
| ---------------------------- | --------------------------- | ------------------------------------------ |
| **Privacy breach (PII)**     | Personal data exposed       | 4 hours notification, 30-day investigation |
| **Proof failure**            | Invalid signal processed    | 24 hours assessment, immediate filtering   |
| **Data retention violation** | Data held > retention limit | Immediate audit, systematic deletion       |
| **Scope creep**              | Data used beyond purpose    | 48 hours remediation, future prevention    |

### Response Framework

```
Detection → Assessment → Remediation → Communication → Prevention

1. Detection (Automated)
   ├─ Anomaly detection alerts
   ├─ Audit log monitoring
   └─ Verification failure patterns

2. Assessment (Within 24 hours)
   ├─ Scope: How many users affected?
   ├─ Severity: What data exposed?
   └─ Root cause: Why did it happen?

3. Remediation (Ongoing)
   ├─ Technical: Fix vulnerability
   ├─ Data: Delete exposed data
   └─ Access: Revoke unauthorized access

4. Communication (Transparent)
   ├─ User notification (if required by law)
   ├─ Regulator notification (GDPR/CCPA)
   └─ Public transparency report (quarterly)

5. Prevention (Systematic)
   ├─ Code review policy update
   ├─ Additional monitoring
   └─ Process improvement
```

---

## Privacy Compliance Mapping

| Regulation          | Requirement           | Our Implementation                        |
| ------------------- | --------------------- | ----------------------------------------- |
| **GDPR Art. 5**     | Data minimization     | Only meaningful signals captured          |
| **GDPR Art. 6**     | Lawful basis          | User consent for signal collection        |
| **GDPR Art. 17**    | Right to be forgotten | No personal data to delete                |
| **GDPR Art. 25**    | Privacy by design     | ZK + DP built into architecture           |
| **CCPA § 1798.100** | Right to know         | No personal data, so no disclosure needed |
| **CCPA § 1798.105** | Right to delete       | No linkable data to delete                |

---

## Privacy Metrics & Monitoring

### Continuous Privacy Monitoring

```
Metric                          Target      Measurement
─────────────────────────────────────────────────────────
ZK Proof success rate          > 99%        Proof validity logs
DP noise level (ε)              < 0.5       Information loss calculation
Data retention compliance       100%        Automated deletion jobs
Unauthorized access attempts    0           Access log audits
PII detection (false positives) < 0.1%      Regex scanning
Identity re-identification risk < 1%        Linkage attack simulations
```

### Quarterly Privacy Assessment

- **Threat modeling**: New attack vectors identified
- **Data audit**: Verify no data beyond scope
- **Regulatory check**: Compliance with new regulations
- **User study**: Collect privacy perception feedback
