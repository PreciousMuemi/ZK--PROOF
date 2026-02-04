# Signal Layer Implementation Guide

## From Theory to Code (Week 2-3)

---

## Quick Start: 30-Second Summary

**Goal**: Collect behavioral signals that reveal content relevance WITHOUT revealing user identity.

**Approach**:

1. Collect raw activity (client-side, ephemeral)
2. Anonymize & bin (remove IDs, categorize)
3. Create ZK proof of validity (client-side)
4. Send signal bundle + proof (no user ID)
5. Verify proof (server-side)
6. Aggregate with 1000+ others (irreversible)
7. Train models on aggregates only
8. Return recommendations (no PII)

**Safety**: K-anonymity, Differential Privacy, Zero-Knowledge Proofs

---

## Part 1: Client-Side Signal Capture & Anonymization

### 1.1 Activity Collector (Browser SDK)

**What to capture** (ephemeral, client-side only):

```javascript
// Example: Article engagement
const activity = {
  event_type: "article_view", // Safe: categorical
  duration_ms: 527340, // Raw: will be binned
  scroll_pixels: 2847, // Raw: will be converted to %
  scroll_percent: 82.3, // Raw: will be deciled
  content_element: article_element, // Raw: extract metadata
  click_count: 3, // Raw: will be categorized
  timestamp: Date.now(), // Raw: will be binned to time bucket
};

// Example: Video engagement
const videoActivity = {
  event_type: "video_play",
  play_duration_ms: 1065000, // 17m 45s
  total_duration_ms: 1320000, // 22m
  watch_percent: 80.68, // Will be deciled
  skip_count: 0,
  timestamp: Date.now(),
};

// Example: Social interaction
const socialActivity = {
  event_type: "interaction",
  action_type: "like", // Safe: categorical enum
  // DO NOT capture: target_user, comment_text, share_destination
};
```

**What NOT to capture**:

```javascript
// ❌ NEVER collect these:
const forbidden = {
  user_id: "USER-12345", // ❌ Personal identifier
  email: "user@example.com", // ❌ PII
  device_id: "AAID-xxx", // ❌ Device fingerprint
  ip_address: "192.168.1.1", // ❌ Network identifier
  exact_timestamp: "2026-02-04T14:35:42Z", // ❌ Exact time
  creator_name: "John Smith", // ❌ Personal name
  location: { lat: 37.7749, lng: -122.4194 }, // ❌ GPS
  page_viewed: ["article1", "article2", "article3"], // ❌ Sequence
};
```

### 1.2 Content Metadata Extraction

For each activity, extract content properties:

```javascript
// Extract from DOM/metadata
function extractContentMetadata(element) {
  const article = element.dataset;

  return {
    // Topic information (from content attributes)
    topics: {
      primary: "Technology", // From article category
      secondary: ["AI", "Safety"], // From tags
    },

    // Content properties (categorical)
    content_type: "article", // From element.tagName
    content_depth: "Intermediate", // From readability score

    // Creator information (type only!)
    creator_type: "Journalist", // From publisher_type attribute
    authority_tier: 2, // From domain_authority attribute

    // Publication info
    freshness: "Recent", // Computed from publish_date
    credibility: "Journalistic", // From source_type attribute
  };
}
```

### 1.3 Anonymization & Binning

Convert raw metrics into safe, binned signals:

```javascript
// Signal Anonymizer Class
class SignalAnonymizer {
  // Bin continuous values into safe buckets
  static binEngagementDuration(milliseconds) {
    const seconds = milliseconds / 1000;
    if (seconds < 30) return "Skimmed";
    if (seconds < 60) return "Browsed";
    if (seconds < 180) return "Read";
    return "Deep_Read";
  }

  static binScrollDepth(percent) {
    // Convert to decile (1-10)
    const decile = Math.ceil(percent / 10);
    return Math.max(1, Math.min(10, decile));
  }

  static binEngagementFrequency(count) {
    // Example: daily views into buckets
    if (count < 10) return 1; // very_low
    if (count < 30) return 2; // low
    if (count < 100) return 3; // moderate
    if (count < 200) return 4; // high
    return 5; // very_high
  }

  static binTimestamp(date = new Date()) {
    const hour = date.getHours();
    if (hour >= 6 && hour < 12) return "morning";
    if (hour >= 12 && hour < 18) return "afternoon";
    if (hour >= 18 && hour < 24) return "evening";
    return "night";
  }

  // Anonymize a complete activity
  static anonymizeActivity(rawActivity, contentMetadata) {
    return {
      // Temporal (binned, not exact)
      signal_timestamp_bin: this.binTimestamp(),

      // Topic (categorical, top-level only)
      topic_signals: {
        primary_topic: contentMetadata.topics.primary,
        secondary_topics: contentMetadata.topics.secondary.slice(0, 3),
        content_depth: contentMetadata.content_depth,
        content_freshness: contentMetadata.freshness,
        creator_type: contentMetadata.creator_type,
        authority_level: contentMetadata.authority_tier,
        credibility_preference: contentMetadata.credibility,
      },

      // Engagement (binned)
      engagement_signals: {
        engagement_depth: this.binEngagementDuration(rawActivity.duration_ms),
        scroll_depth_decile: this.binScrollDepth(rawActivity.scroll_percent),
        completion_percentile: this.binScrollDepth(
          rawActivity.watch_percent || rawActivity.scroll_percent,
        ),
      },

      // Interaction (categorical set, not sequence)
      interaction_signals: {
        primary_interactions: [this.mapEventType(rawActivity.event_type)],
      },
    };
  }

  static mapEventType(eventType) {
    const mapping = {
      article_view: "View",
      video_play: "View",
      link_click: "Click",
      scroll_event: "Scroll",
      like_action: "Like",
      share_action: "Share",
      comment_action: "Comment",
      bookmark_action: "Bookmark",
    };
    return mapping[eventType] || "View";
  }
}

// Usage
const rawData = {
  event_type: "article_view",
  duration_ms: 527340,
  scroll_percent: 82.3,
  timestamp: Date.now(),
};

const metadata = {
  topics: { primary: "Technology", secondary: ["AI"] },
  content_type: "article",
  content_depth: "Intermediate",
  creator_type: "Journalist",
  authority_tier: 2,
  freshness: "Recent",
  credibility: "Journalistic",
};

const safeSignal = SignalAnonymizer.anonymizeActivity(rawData, metadata);
// Returns safely binned signal ready for validation
```

### 1.4 Session Management (Ephemeral Only)

```javascript
class SessionManager {
  constructor() {
    this.sessionId = this.generateEphemeralId();
    this.sessionStart = Date.now();
    this.sessionTTL = 1 * 60 * 60 * 1000; // 1 hour
    this.signals = [];
  }

  // Generate random session ID (NOT linked to user)
  generateEphemeralId() {
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    return Array.from(array, (byte) => byte.toString(16).padStart(2, "0")).join(
      "",
    );
  }

  // Check if session expired
  isExpired() {
    return Date.now() - this.sessionStart > this.sessionTTL;
  }

  // Add signal to session buffer
  addSignal(anonSignal) {
    if (this.isExpired()) {
      // Clear expired session
      this.signals = [];
      this.sessionStart = Date.now();
      this.sessionId = this.generateEphemeralId();
    }

    this.signals.push(anonSignal);
  }

  // Clear signals after transmission
  clearSignals() {
    this.signals = [];
    // Securely erase from memory
    this.sessionId = this.generateEphemeralId();
  }

  // Get signals for transmission
  getSignalsForSubmission() {
    return {
      signal_id: this.sessionId,
      signals: this.signals,
      signal_timestamp_bin: SignalAnonymizer.binTimestamp(),
    };
  }
}
```

---

## Part 2: Zero-Knowledge Proof Generation (Client-Side)

**Purpose**: Prove signal validity without revealing signal values.

### 2.1 ZK Proof Circuit (Circom)

```circom
// signal_validity.circom
// Proves signal is valid without revealing values

pragma circom 2.0;

template SignalValidity() {
    // Private inputs (prover knows these)
    signal input engagement_depth;     // 0-3 (0=skimmed, 3=deep)
    signal input scroll_depth;         // 0-10 (decile)
    signal input frequency_bin;        // 0-5 (frequency bucket)
    signal input topic_code;           // 0-9 (topic enum)

    // Public inputs (everyone knows these)
    signal input topic_hash;           // Hash of topic (commitment)

    // Constraints (prove without revealing)

    // 1. Engagement depth in valid range
    engagement_depth * (engagement_depth - 1) * (engagement_depth - 2) * (engagement_depth - 3) === 0;

    // 2. Scroll depth in valid range (0-10)
    signal scroll_valid;
    scroll_valid <== scroll_depth * (10 - scroll_depth);
    scroll_valid === scroll_valid; // Ensure non-negative

    // 3. Frequency bin in valid range (0-5)
    frequency_bin * (frequency_bin - 1) * (frequency_bin - 2) *
    (frequency_bin - 3) * (frequency_bin - 4) * (frequency_bin - 5) === 0;

    // 4. Topic matches commitment
    signal topic_check;
    topic_check <== topic_code * (topic_code - 1) * (topic_code - 2) *
                      (topic_code - 3) * (topic_code - 4) * (topic_code - 5) *
                      (topic_code - 6) * (topic_code - 7) * (topic_code - 8) * (topic_code - 9);
    // All values 0-9 valid
}

component main { public [ topic_hash ] } = SignalValidity();
```

### 2.2 Proof Generation (JavaScript/snarkjs)

```javascript
const snarkjs = require("snarkjs");
const wasm = require("./circuits/signal_validity_js/signal_validity.js");
const zkey = require("./circuits/signal_validity_final.zkey");

class ProofGenerator {
  // Initialize proof system
  static async initialize() {
    this.wasm = wasm;
    this.zkey = zkey;
  }

  // Generate proof of signal validity
  static async generateProof(anonSignal) {
    // Map signal values to witness
    const witness = {
      engagement_depth: this.mapEngagementDepth(
        anonSignal.engagement_signals.engagement_depth,
      ),
      scroll_depth: anonSignal.engagement_signals.scroll_depth_decile,
      frequency_bin: anonSignal.engagement_signals.frequency_bin || 3,
      topic_code: this.mapTopicToCode(anonSignal.topic_signals.primary_topic),
      topic_hash: this.hashTopic(anonSignal.topic_signals.primary_topic),
    };

    try {
      // Generate proof (client-side, takes ~1-2 seconds)
      const { proof, publicSignals } = await snarkjs.groth16.fullProve(
        witness,
        this.wasm,
        this.zkey,
      );

      return {
        proof: proof,
        publicSignals: publicSignals,
        isValid: true,
      };
    } catch (error) {
      console.error("Proof generation failed:", error);
      return {
        proof: null,
        publicSignals: null,
        isValid: false,
        error: error.message,
      };
    }
  }

  // Map engagement depth string to number
  static mapEngagementDepth(depthString) {
    const mapping = {
      Skimmed: 0,
      Browsed: 1,
      Read: 2,
      Deep_Read: 3,
    };
    return mapping[depthString] || 1;
  }

  // Map topic to numeric code for circuit
  static mapTopicToCode(topic) {
    const mapping = {
      Technology: 0,
      Science: 1,
      Business: 2,
      Entertainment: 3,
      Health: 4,
      Politics: 5,
      Sports: 6,
      Culture: 7,
      Education: 8,
      Other: 9,
    };
    return mapping[topic] || 9;
  }

  // Hash topic (public commitment)
  static hashTopic(topic) {
    const crypto = require("crypto");
    return crypto.createHash("sha256").update(topic).digest("hex");
  }
}

// Usage
async function collectAndProveSignal(rawActivity, contentMetadata) {
  // 1. Anonymize
  const anonSignal = SignalAnonymizer.anonymizeActivity(
    rawActivity,
    contentMetadata,
  );

  // 2. Generate proof
  const proofResult = await ProofGenerator.generateProof(anonSignal);

  // 3. Combine into signal bundle
  return {
    anonymized_signal: anonSignal,
    zk_proof: proofResult.proof,
    public_signals: proofResult.publicSignals,
    proof_valid: proofResult.isValid,
  };
}
```

---

## Part 3: Server-Side Validation & Aggregation

### 3.1 Proof Verification (Rust/Arkworks)

```rust
use ark_groth16::{Groth16, Proof, VerifyingKey};
use ark_bn254::Bn254;

pub struct ProofVerifier {
    vkey: VerifyingKey<Bn254>,
}

impl ProofVerifier {
    pub fn new(vkey: VerifyingKey<Bn254>) -> Self {
        ProofVerifier { vkey }
    }

    pub fn verify_proof(
        &self,
        proof: &Proof<Bn254>,
        public_inputs: &[<Bn254 as Pairing>::ScalarField],
    ) -> Result<bool, String> {
        match Groth16::<Bn254>::verify(&self.vkey, public_inputs, proof) {
            Ok(is_valid) => Ok(is_valid),
            Err(e) => Err(format!("Verification failed: {:?}", e)),
        }
    }
}

// Usage
pub async fn handle_signal_submission(
    signal_bundle: SignalBundle,
    verifier: &ProofVerifier,
) -> Result<ValidatedSignal, String> {
    // 1. Verify ZK proof
    let is_valid = verifier.verify_proof(
        &signal_bundle.proof,
        &signal_bundle.public_signals,
    )?;

    if !is_valid {
        return Err("Proof verification failed".to_string());
    }

    // 2. Validate signal schema
    validate_signal_schema(&signal_bundle.signal)?;

    // 3. Check aggregation constraints
    verify_aggregation(&signal_bundle.signal).await?;

    // 4. Return validated signal (PII-free)
    Ok(ValidatedSignal {
        signal: signal_bundle.signal,
        proof_verified: true,
        timestamp_received: chrono::Utc::now(),
    })
}
```

### 3.2 Signal Aggregation

```python
# SQL aggregation (PostgreSQL)
def get_aggregated_signals(
    primary_topic: str,
    frequency_bin: int,
    engagement_depth: str
):
    """
    Get aggregate statistics for matching signals.
    Ensures signals aggregated with 1000+ users.
    """
    query = """
    SELECT
        COUNT(*) as cohort_size,
        AVG(scroll_depth_decile) as avg_scroll_depth,
        AVG(topic_affinity) as avg_topic_affinity,
        MODE(format_type) as preferred_format,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY completion_percentile)
            as median_completion
    FROM signals_aggregated
    WHERE
        primary_topic = %s
        AND frequency_bin = %s
        AND engagement_depth = %s
        AND created_at > NOW() - INTERVAL '30 days'
    HAVING COUNT(*) >= 1000;  -- k-anonymity constraint
    """

    cursor.execute(query, (primary_topic, frequency_bin, engagement_depth))
    result = cursor.fetchone()

    if not result or result['cohort_size'] < 1000:
        raise ValueError("Cohort too small for k-anonymity")

    return result

# Feed to ML model
def generate_recommendations(validated_signal):
    """
    Generate recommendations from aggregated signal only.
    No individual signal values exposed to model.
    """
    # Get aggregates
    aggregates = get_aggregated_signals(
        primary_topic=validated_signal['primary_topic'],
        frequency_bin=validated_signal['frequency_bin'],
        engagement_depth=validated_signal['engagement_depth']
    )

    # Create feature vector from aggregates (not individual values)
    features = {
        'avg_scroll_depth': aggregates['avg_scroll_depth'],
        'avg_topic_affinity': aggregates['avg_topic_affinity'],
        'preferred_format': encode_categorical(aggregates['preferred_format']),
        'median_completion': aggregates['median_completion']
    }

    # Inference
    recommendations = ml_model.predict([features])

    # Return recommendations (NO personal data)
    return {
        'recommendations': recommendations,
        'cohort_size': aggregates['cohort_size'],
        'timestamp': datetime.now()
    }
```

### 3.3 Automated Cleanup & Privacy Enforcement

```python
# Scheduled job (runs daily)
def cleanup_expired_signals():
    """
    Delete all signals older than 90 days.
    Ensures no long-term storage of behavioral data.
    """
    cutoff_date = datetime.now() - timedelta(days=90)

    # Delete raw signals
    cursor.execute(
        "DELETE FROM signals WHERE created_at < %s",
        (cutoff_date,)
    )

    # Keep only anonymized aggregates
    cursor.execute(
        "DELETE FROM signal_logs WHERE created_at < %s",
        (cutoff_date,)
    )

    logger.info(f"Deleted signals older than {cutoff_date}")

def verify_no_pii_storage():
    """
    Automated check: Ensure no PII is stored anywhere.
    Runs hourly.
    """
    # Scan for forbidden patterns
    forbidden_patterns = [
        r'user_id',
        r'email',
        r'\d{3}-\d{3}-\d{4}',  # Phone
        r'\d+\.\d+,\s*-?\d+\.\d+'  # GPS coordinates
    ]

    cursor.execute(
        """
        SELECT column_name, table_name
        FROM information_schema.columns
        WHERE data_type IN ('text', 'varchar')
        """
    )

    for column, table in cursor.fetchall():
        for pattern in forbidden_patterns:
            sample = cursor.execute(
                f"SELECT {column} FROM {table} LIMIT 100"
            )
            for row in sample:
                if re.search(pattern, str(row)):
                    alert(f"ALERT: PII pattern detected in {table}.{column}")
```

---

## Part 4: Testing & Validation

### 4.1 Unit Tests for Anonymizer

```javascript
describe("SignalAnonymizer", () => {
  test("should bin duration correctly", () => {
    expect(SignalAnonymizer.binEngagementDuration(15000)).toBe("Skimmed"); // 15s
    expect(SignalAnonymizer.binEngagementDuration(45000)).toBe("Browsed"); // 45s
    expect(SignalAnonymizer.binEngagementDuration(120000)).toBe("Read"); // 120s
    expect(SignalAnonymizer.binEngagementDuration(600000)).toBe("Deep_Read"); // 600s
  });

  test("should bin scroll depth to decile", () => {
    expect(SignalAnonymizer.binScrollDepth(5)).toBe(1); // 0-10%
    expect(SignalAnonymizer.binScrollDepth(55)).toBe(6); // 50-60%
    expect(SignalAnonymizer.binScrollDepth(95)).toBe(10); // 90-100%
  });

  test("should reject PII signals", () => {
    const badSignal = {
      user_id: "USER123",
      engagement_depth: "Deep_Read",
    };
    expect(() => SignalAnonymizer.validateSignal(badSignal)).toThrow(
      "PII detected",
    );
  });
});
```

### 4.2 Privacy Attack Simulation

```python
def test_linkage_attack():
    """
    Try to re-identify user via signal + public data.
    Should fail (cannot uniquely identify).
    """
    signal = get_test_signal()

    # Attacker tries: signal + Wikipedia
    user_profile = {
        'topics': signal.get_topics(),
        'format_preference': signal.get_format(),
        'expertise_level': signal.get_complexity()
    }

    # Query: How many users have this exact combo?
    cohort_size = database.query_cohort_size(user_profile)

    # Should have 1000+ matches
    assert cohort_size >= 1000, f"Cohort too small: {cohort_size} < 1000"

def test_membership_inference():
    """
    Can attacker determine if person X is in dataset?
    Should return "indistinguishable" (can't tell).
    """
    signal_with_X = aggregate_with_person_X()
    signal_without_X = aggregate_without_person_X()

    # Distributions should be statistically indistinguishable
    distance = statistical_distance(signal_with_X, signal_without_X)

    # With k=1000, distance should be ~ 1/sqrt(k) = 0.03
    assert distance < 0.05, f"Can distinguish with person: {distance}"
```

---

## Quick Reference: Implementation Checklist

```
WEEK 2-3 IMPLEMENTATION TASKS:

CLIENT-SIDE (JavaScript/Web):
☐ Activity collector (track events)
☐ Content metadata extractor
☐ Signal anonymizer (bin values)
☐ Session manager (ephemeral)
☐ ZK proof generator (snarkjs)
☐ Signal validator (schema check)
☐ Network transmission (HTTPS)
☐ Unit tests (anonymizer)

SERVER-SIDE (Rust/Python):
☐ Proof verifier (arkworks/gnark)
☐ Signal validator (7 layers)
☐ Database aggregation queries
☐ K-anonymity verification
☐ ML model integration
☐ Automatic cleanup jobs
☐ Privacy monitoring
☐ Unit & integration tests

INFRASTRUCTURE:
☐ Database schema (signals_aggregated)
☐ Monitoring (proof success rate)
☐ Logging (validation failures)
☐ Alerting (PII detection)
☐ Backup & recovery (encrypted)

TESTING:
☐ Privacy attack simulations
☐ Performance benchmarks
☐ End-to-end test (signal → proof → verify)
☐ k-anonymity verification
☐ Compliance audit
```

---

## Estimated Timelines

| Task                     | Hours         | Difficulty |
| ------------------------ | ------------- | ---------- |
| Activity Collector       | 4             | Easy       |
| Anonymizer               | 4             | Easy       |
| ZK Proof Integration     | 8             | Medium     |
| Server Validator         | 8             | Medium     |
| Aggregation Queries      | 6             | Medium     |
| Testing & Privacy Audits | 10            | Hard       |
| **Total**                | **~40 hours** | -          |

---

## Success Metrics

✅ All signals pass validation  
✅ K-anonymity k=1000 verified  
✅ Proof verification < 100ms  
✅ Client proof generation < 2s  
✅ No PII in any signal  
✅ Privacy attack simulations fail (good!)  
✅ 95%+ recommendation accuracy
