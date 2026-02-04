# Signal Validation Rules & Input Sanitization

## Week 2: Data & Signal Layer

**Purpose**: Ensure all incoming signals are safe, non-identifiable, and valid for processing.

---

## Signal Validation Pipeline

```
Raw Input
    ↓
[1] SCHEMA VALIDATION (JSON schema compliance)
    ↓
[2] TYPE & CARDINALITY CHECKS (No continuous values, no rare enums)
    ↓
[3] TEMPORAL SANITIZATION (No exact times, only bins)
    ↓
[4] PII DETECTION (Reject if contains user IDs, emails, etc.)
    ↓
[5] SEQUENCE DETECTION (Reject if contains behavioral sequences)
    ↓
[6] AGGREGATION VERIFICATION (Verify cohort size >= 1000)
    ↓
[7] RE-IDENTIFICATION RISK ASSESSMENT (K-anonymity check)
    ↓
SAFE SIGNAL ✅ or REJECTED ❌
```

---

## Validation Rule 1: Schema Compliance

### Rule: All signals must match `signal_schema.json`

**Check**:

```
Does signal JSON match schema?
- topic_signals object required: ✓
- engagement_signals object required: ✓
- interaction_signals object required: ✓
- quality_signals object required: ✓
- All enums are exact values (not custom): ✓
- No additional properties: ✓
```

**Pseudocode**:

```javascript
function validateSchema(signal) {
  const schema = loadSchema("signal_schema.json");
  const validator = new JSONValidator(schema);
  const result = validator.validate(signal);

  if (!result.valid) {
    return {
      status: "REJECTED",
      reason: "Schema validation failed",
      errors: result.errors,
    };
  }
  return { status: "PASSED" };
}
```

**Example Passes**:

```json
{
  "signal_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "signal_timestamp_bin": "afternoon",
  "topic_signals": {
    "primary_topic": "Technology",
    "secondary_topics": ["AI", "Security"],
    "content_depth": "Advanced",
    ...
  },
  ...
}
```

**Example Rejects**:

```json
{
  "user_id": "12345",  // ❌ Not in schema (PII)
  "timestamp": "2026-02-04T14:35:42Z",  // ❌ Exact time (not binned)
  "article_title": "Top 10 AI Safety Risks",  // ❌ Specific content (not allowed)
  ...
}
```

---

## Validation Rule 2: No Exact/Continuous Values

### Rule: All values must be binned, bucketed, or categorical

**Check**:

```
For each numeric field:
- Must be integer in range (minimum/maximum in schema): ✓
- Must be ordinal (1-5, 1-10 decile, etc): ✓
- NOT continuous floating point: ✗
- NOT unbounded: ✗

For each string field:
- Must be exact enum value: ✓
- NOT freeform text: ✗
- NOT user-generated: ✗
```

**Pseudocode**:

```javascript
function validateCardinality(signal) {
  const cardinalityLimits = {
    frequency_bin: { min: 1, max: 5 }, // ✓ 5 buckets
    scroll_depth_decile: { min: 1, max: 10 }, // ✓ 10 buckets
    topic_affinity: { min: 1, max: 5 }, // ✓ 5 buckets
  };

  for (const [field, limits] of Object.entries(cardinalityLimits)) {
    const value = getFieldValue(signal, field);
    if (value < limits.min || value > limits.max) {
      return {
        status: "REJECTED",
        reason: `Cardinality check failed: ${field} = ${value}`,
        acceptable_range: limits,
      };
    }
  }

  return { status: "PASSED" };
}
```

**Examples**:

✅ PASS:

```json
{
  "frequency_bin": 3, // Integer 1-5 bucket ✓
  "scroll_depth_decile": 8, // Integer 1-10 decile ✓
  "primary_topic": "Technology" // Exact enum ✓
}
```

❌ REJECT:

```json
{
  "frequency_bin": 3.7, // Continuous float ✗
  "scroll_depth_decile": 8.4, // Precise decimal ✗
  "engagement_count": 47, // Unbounded exact count ✗
  "primary_topic": "AI & Safety" // Not exact enum ✗
}
```

---

## Validation Rule 3: Temporal Protection (No Exact Timestamps)

### Rule: Only time buckets allowed, never exact timestamps

**Check**:

```
signal_timestamp_bin must be one of:
- "morning" (06:00-12:00)
- "afternoon" (12:00-18:00)
- "evening" (18:00-24:00)
- "night" (00:00-06:00)

NOT:
- Exact time (14:35:42)
- Hour precision (14:00)
- Minute precision (14:35)
- ISO timestamps (2026-02-04T14:35:42Z)
```

**Pseudocode**:

```javascript
function validateTemporal(signal) {
  const allowedBins = ["morning", "afternoon", "evening", "night"];

  if (!allowedBins.includes(signal.signal_timestamp_bin)) {
    return {
      status: "REJECTED",
      reason: "Temporal value not binned",
      provided: signal.signal_timestamp_bin,
      allowed: allowedBins,
    };
  }

  // Check for forbidden exact timestamp patterns
  const forbiddenPatterns = [
    /\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/, // ISO 8601
    /\d{2}:\d{2}:\d{2}/, // HH:MM:SS
    /\d{2}:\d{2}/, // HH:MM
    /\d{4}-\d{2}-\d{2}/, // Date only
  ];

  const signalString = JSON.stringify(signal);
  for (const pattern of forbiddenPatterns) {
    if (pattern.test(signalString)) {
      return {
        status: "REJECTED",
        reason: "Exact timestamp detected",
        pattern: pattern.toString(),
      };
    }
  }

  return { status: "PASSED" };
}
```

**Examples**:

✅ PASS:

```json
{
  "signal_timestamp_bin": "afternoon"
}
```

❌ REJECT:

```json
{
  "signal_timestamp_bin": "2026-02-04T14:35:42Z" // Exact timestamp
}
```

```json
{
  "signal_timestamp_bin": "14:35" // Hour:minute precision
}
```

---

## Validation Rule 4: PII Detection (Reject If Contains Personal Data)

### Rule: Scan for personally identifiable information

**Forbidden Patterns**:

```
User Identifiers:
- user_id, userid, user_ID
- account_id, account_number
- uuid (unless specifically ephemeral session)

Contact Information:
- email (contains @)
- phone (pattern: \d{3}-\d{3}-\d{4})
- postal_code (5-9 digit patterns)
- home_address, address

Location Data:
- latitude, longitude (decimal coordinates)
- gps, location, coordinates
- city, state, country (unless aggregate regional)
- zip_code

Personal Names:
- first_name, last_name, name
- creator_name (must use creator_type instead)
- user_name, username
- handle (@ prefix for creators)

Device Identifiers:
- device_id, device_ID
- mac_address, macaddress
- aaid, idfa, udid
- imei, imsi, phone_id
- device_fingerprint, fingerprint
- browser_fingerprint

Sensitive Attributes:
- age, date_of_birth, dob, age_range
- gender, sex, gender_identity
- race, ethnicity, nationality
- religion, political_view, political_affiliation
- sexual_orientation, marital_status
- health, medical, disease, condition
- income, salary, financial_status
- education_level, degree, school
- occupation, job_title, profession
```

**Pseudocode**:

```javascript
function detectPII(signal) {
  const forbiddenPatterns = {
    user_identifiers: [/user_?id/i, /account_?id/i, /account_?number/i],
    contact: [
      /email/i,
      /phone/i,
      /postal_?code/i,
      /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/, // Email regex
    ],
    location: [
      /latitude|longitude/i,
      /gps|location|coordinates/i,
      /\b\d+\.?\d*,\s*-?\d+\.?\d*\b/, // Coordinate pattern
    ],
    names: [/first_?name|last_?name/i, /creator_?name/i, /user_?name/i],
    device: [
      /device_?id/i,
      /mac_?address|macaddress/i,
      /aaid|idfa|udid/i,
      /device_?fingerprint|fingerprint/i,
    ],
    sensitive: [
      /age/i,
      /date_?of_?birth|dob/i,
      /gender|sex/i,
      /race|ethnicity/i,
      /health|medical|disease/i,
      /income|salary/i,
      /religion|political/i,
    ],
  };

  const signalString = JSON.stringify(signal).toLowerCase();
  const signalValues = JSON.stringify(Object.values(signal));

  for (const [category, patterns] of Object.entries(forbiddenPatterns)) {
    for (const pattern of patterns) {
      if (pattern.test(signalString) || pattern.test(signalValues)) {
        return {
          status: "REJECTED",
          reason: "PII detected",
          category: category,
          pattern: pattern.toString(),
        };
      }
    }
  }

  return { status: "PASSED" };
}
```

**Examples**:

✅ PASS:

```json
{
  "creator_type": "Professional Individual", // Type, not name ✓
  "authority_level": 3, // Tier, not domain ✓
  "signal_timestamp_bin": "afternoon" // Binned time ✓
}
```

❌ REJECT:

```json
{
  "user_id": "USER-12345", // User ID ✗
  "email": "john@example.com", // Email ✗
  "name": "John Smith" // Name ✗
}
```

---

## Validation Rule 5: Sequence Detection (Reject If Contains Behavioral Sequences)

### Rule: No ordered sequences of actions/views

**Check**:

```
Forbidden:
- "views": [article_1, article_2, article_3]  // Sequence reveals order
- "actions": ["click", "scroll", "click"]      // Timeline reconstructible
- "topics_viewed_in_order": ["AI", "ML", ...]  // Learning path visible

Allowed:
- "topics": ["AI", "ML", "Safety"]             // SET (unordered)
- "interaction_types": ["click", "scroll"]     // SET (no order)
- Topic bucket without order
```

**Pseudocode**:

```javascript
function detectSequences(signal) {
  const forbiddenKeys = [
    "sequence",
    "action_sequence",
    "view_sequence",
    "topic_sequence",
    "click_sequence",
    "user_journey",
    "user_path",
    "behavior_timeline",
    "temporal_sequence",
    "interaction_order",
    "engagement_history",
  ];

  function flattenAndCheckKeys(obj, path = "") {
    for (const [key, value] of Object.entries(obj)) {
      const fullKey = path ? `${path}.${key}` : key;

      // Check if key suggests sequences
      if (forbiddenKeys.some((fk) => fullKey.toLowerCase().includes(fk))) {
        return {
          status: "REJECTED",
          reason: "Behavioral sequence detected",
          field: fullKey,
        };
      }

      // Check if value is array with sequence-like content
      if (Array.isArray(value) && value.length > 1) {
        // Arrays are OK if they're sets (unordered), not sequences
        // But we should warn if field suggests temporal ordering
        if (
          fullKey.toLowerCase().includes("time") ||
          fullKey.toLowerCase().includes("order") ||
          fullKey.toLowerCase().includes("sequence")
        ) {
          return {
            status: "REJECTED",
            reason: "Potential sequence in field",
            field: fullKey,
          };
        }
      }

      // Recursively check nested objects
      if (
        typeof value === "object" &&
        value !== null &&
        !Array.isArray(value)
      ) {
        const result = flattenAndCheckKeys(value, fullKey);
        if (result && result.status === "REJECTED") {
          return result;
        }
      }
    }
    return { status: "PASSED" };
  }

  return flattenAndCheckKeys(signal);
}
```

**Examples**:

✅ PASS:

```json
{
  "primary_interactions": ["View", "Click", "Scroll"], // Set (unordered) ✓
  "secondary_topics": ["AI", "Security"], // Set (no sequence) ✓
  "social_actions": ["Like", "Share"] // Set (no order) ✓
}
```

❌ REJECT:

```json
{
  "view_sequence": ["article1", "article2", "article3"] // Sequence ✗
}
```

```json
{
  "action_timeline": [
    { "time": "14:35", "action": "click" },
    { "time": "14:36", "action": "scroll" },
    { "time": "14:37", "action": "click" }
  ] // Timeline reconstructible ✗
}
```

---

## Validation Rule 6: Aggregation Verification

### Rule: Signal must be aggregated with 1000+ other users

**Check**:

```
For safety_metadata.aggregation_cohort_size:
- Must be >= 1000
- This confirms signal not unique to individual
- Spot-check by querying:
  "SELECT COUNT(*) WHERE all_signal_features = X"
  Must return >= 1000
```

**Pseudocode**:

```javascript
function verifyAggregation(signal, database) {
  const requiredCohortSize = 1000;

  // Extract signal features (exclude metadata)
  const features = {
    primary_topic: signal.topic_signals.primary_topic,
    frequency_bin: signal.engagement_signals.frequency_bin,
    engagement_depth: signal.engagement_signals.engagement_depth,
    // ... other key features
  };

  // Query database for cohort size
  const query = `
    SELECT COUNT(*) as cohort_size
    FROM signals_aggregated
    WHERE 
      primary_topic = $1
      AND frequency_bin = $2
      AND engagement_depth = $3
  `;

  const result = database.query(query, Object.values(features));
  const cohortSize = result.rows[0].cohort_size;

  if (cohortSize < requiredCohortSize) {
    return {
      status: "REJECTED",
      reason: "Cohort size too small",
      cohort_size: cohortSize,
      required: requiredCohortSize,
      features: features,
    };
  }

  return {
    status: "PASSED",
    cohort_size: cohortSize,
  };
}
```

**Example**:

```
Signal combination:
- primary_topic: "Technology"
- frequency_bin: 4 (high)
- engagement_depth: "Deep_Read"

Database query result:
- Cohort size: 8,472 users
- Required: 1,000
- Status: ✅ PASS (8,472 >= 1,000)
```

---

## Validation Rule 7: K-Anonymity Risk Assessment

### Rule: Signal combination cannot identify fewer than 1000 users

**Check**:

```
Formal definition (k-anonymity with k=1000):
- For any combination of signal values
- Cannot isolate fewer than 1000 users with that combo
- Test by querying aggregation cohort size
```

**Pseudocode**:

```javascript
function assessKAnonymity(signal, database) {
  const k = 1000; // Minimum anonymity threshold

  // Test 1: Single features (each should have 1000+)
  const singleFeatureTests = [
    { field: "primary_topic", value: signal.topic_signals.primary_topic },
    { field: "frequency_bin", value: signal.engagement_signals.frequency_bin },
    {
      field: "engagement_depth",
      value: signal.engagement_signals.engagement_depth,
    },
  ];

  for (const test of singleFeatureTests) {
    const count = database.query(
      `SELECT COUNT(*) FROM signals WHERE ${test.field} = $1`,
      [test.value],
    ).rows[0];

    if (count < k) {
      return {
        status: "REJECTED",
        reason: "Single feature fails k-anonymity",
        feature: test.field,
        users_in_bucket: count,
        required: k,
      };
    }
  }

  // Test 2: Feature pairs (most specific likely duo)
  const pairTests = [
    { fields: ["primary_topic", "frequency_bin"] },
    { fields: ["frequency_bin", "engagement_depth"] },
    { fields: ["primary_topic", "content_depth"] },
  ];

  for (const test of pairTests) {
    const values = test.fields.map((f) => getNestedValue(signal, f));
    const count = database.query(
      `SELECT COUNT(*) FROM signals 
       WHERE ${test.fields[0]} = $1 AND ${test.fields[1]} = $2`,
      values,
    ).rows[0];

    if (count < k) {
      return {
        status: "REJECTED",
        reason: "Feature pair fails k-anonymity",
        features: test.fields,
        users_in_bucket: count,
        required: k,
      };
    }
  }

  return {
    status: "PASSED",
    k_anonymity_verified: k,
  };
}
```

**Example**:

```
Signal: {
  primary_topic: "Technology",
  frequency_bin: 4,
  engagement_depth: "Deep_Read"
}

K-Anonymity Tests:
1. primary_topic="Technology" → 1,250,000 users ✓
2. frequency_bin=4 → 300,000 users ✓
3. engagement_depth="Deep_Read" → 250,000 users ✓
4. (primary_topic="Technology" AND frequency_bin=4) → 127,000 users ✓
5. (frequency_bin=4 AND engagement_depth="Deep_Read") → 58,000 users ✓
6. (primary_topic="Technology" AND frequency_bin=4 AND engagement_depth="Deep_Read") → 12,800 users ✓

All tests >= 1000 users: ✅ PASSED
```

---

## Complete Validation Checklist

Before accepting any signal:

```
SCHEMA & STRUCTURE:
☐ JSON schema validation passed
☐ Required fields present
☐ No extra/unknown fields
☐ All enums are exact values

VALUES & CARDINALITY:
☐ All numeric values are binned/bucketed
☐ No continuous floating-point values
☐ Cardinality limits respected
☐ No unbounded counts

TEMPORAL SAFETY:
☐ Only time bins allowed (morning/afternoon/evening/night)
☐ No exact timestamps in any field
☐ No date patterns detected
☐ No temporal sequences

PII & IDENTIFICATION:
☐ No user IDs or personal identifiers
☐ No email addresses or contact info
☐ No personal names
☐ No device identifiers
☐ No location data below regional level
☐ No demographic inferences
☐ No sensitive attributes

SEQUENCE & PATTERN LEAKAGE:
☐ No behavioral sequences
☐ No action timelines
☐ No ordered topic views
☐ No learning paths or progressions
☐ No activity patterns

AGGREGATION & PRIVACY:
☐ Cohort size >= 1,000 verified
☐ K-anonymity k=1,000 verified
☐ Signal combination not unique
☐ Cannot identify individual

STATUS: ✅ SAFE or ❌ REJECTED
```

---

## Rejection Codes

When signal fails validation, return specific error code:

| Code           | Reason                       | Example                   |
| -------------- | ---------------------------- | ------------------------- |
| **SCHEMA_001** | Schema validation failed     | Missing required field    |
| **CARD_001**   | Cardinality check failed     | Continuous float value    |
| **CARD_002**   | Enum value not recognized    | Non-standard topic value  |
| **TEMP_001**   | Temporal not binned          | Exact timestamp detected  |
| **PII_001**    | User ID detected             | user_id field found       |
| **PII_002**    | Email detected               | email@example.com pattern |
| **PII_003**    | Personal name detected       | first_name / last_name    |
| **PII_004**    | Device ID detected           | MAC address or AAID       |
| **PII_005**    | Demographics detected        | age / gender / location   |
| **SEQ_001**    | Behavioral sequence detected | view_sequence array       |
| **SEQ_002**    | Action timeline detected     | Ordered actions with time |
| **AGGR_001**   | Cohort too small             | < 1,000 users             |
| **AGGR_002**   | K-anonymity failed           | Single feature < 1,000    |
| **AGGR_003**   | Feature pair too specific    | Feature combo < 1,000     |

---

## Implementation Notes

1. **Order of Operations**: Always validate schema before checking values (fail fast)
2. **Parallelization**: Rules 2-7 can run in parallel after schema passes
3. **Caching**: PII patterns can be compiled regex (cache for performance)
4. **Database**: Aggregation/K-anonymity checks require database access (may be slower)
5. **Feedback**: Return detailed error messages (not just "rejected")
6. **Logging**: Log all rejections for privacy audits
