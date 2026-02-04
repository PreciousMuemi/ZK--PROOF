# AI Prompt for System Design

## ZK-Proof AI Relevance Engine

---

## Original Prompt

> Design a system architecture for an AI relevance engine that analyzes anonymized activity signals (engagement frequency, interaction type, topic tags) without storing personal data. Include a ZK-proof verification layer.

---

## Expanded Prompt for Implementation

### Core Requirements

Design a **privacy-first relevance engine** with these constraints:

#### Input Data

- **Engagement Frequency**: How often users interact (e.g., 5 views/day)
- **Interaction Type**: What methods they use (clicks, scrolls, shares)
- **Topic Tags**: What categories interest them (Technology, Science, Finance)
- **Constraint**: NO personal identifiers at any layer

#### Zero-Knowledge Proof Layer

- **Prover**: Client device generates mathematical proof of signal validity
- **Verifier**: Server validates proof without seeing actual signal values
- **Proof Goal**: Confirm signals meet validity rules without revealing content
- **Property**: Proof reveals only: "This is valid signal of type X" (nothing more)

#### No Personal Data Storage

- **Strict Rule**: No user IDs, emails, locations, device identifiers
- **Implication**: After processing, signals have zero re-identification risk
- **Enforcement**: Architectural separation of anonymous data from user data

### Architectural Constraints

#### Data Flow

1. **Client**: Collect activity → Anonymize → Generate ZK proof → Send bundle
2. **Transmission**: Only (Anonymized Signals + ZK Proof + No User ID)
3. **Server**: Verify proof → Aggregate with other users → Run inference → Return results
4. **No Backtrack**: Cannot map results back to any individual

#### Privacy Boundaries

- Client-side: Raw personal data never transmitted
- Network: Encrypted, no IP tracking, signal content hidden
- Server-side: No user identification field in database
- Retention: Signals deleted after processing (< 30 days aggregates)
- Purpose: Relevance recommendations ONLY, no other use cases

#### Security Guarantees

- **Zero-Knowledge**: Proof reveals nothing except validity
- **Non-Interactive**: Single proof transmission, no round trips
- **Deterministic Verification**: Yes/no proof outcome, no ambiguity
- **Completeness**: All valid signals verified, no false rejections (< 0.1%)

### Implementation Details to Address

#### 1. Signal Design

```
Define what each signal represents:
- Engagement frequency: Count aggregation → Binned (0-10, 10-50, 50+)
- Interaction type: Category enum → 5-7 types (click, scroll, share, etc.)
- Topic tags: Hierarchical taxonomy → 3-level (top-level only exposed)
- Validity rules: What makes signal acceptable (ranges, types, etc.)
```

#### 2. ZK Proof Circuit

```
What to prove (without revealing):
- Signal count within valid range (e.g., 0 < count < 1000)
- Topic in predefined set (e.g., matches one of 50 topics)
- Timestamp is recent (e.g., within 30 days)
- Interaction type is valid enum

Circuit technology options:
- circom + snarkjs (JavaScript, browser-compatible)
- Arkworks (Rust, server-side verification)
- Cairo (Cairo VM, Starknet-compatible)
```

#### 3. Aggregation Strategy

```
How to combine anonymous signals:
- Cohort-based: 10K+ users with same topic interest
- Bucketing: Engagement level ranges, not exact counts
- Noise injection: ± 2% statistical noise to prevent inference
- Never disaggregate: Once aggregated, cannot return to individuals
```

#### 4. Verification Layer Architecture

```
Server-side components:
- Proof verifier: Runs ZK proof validation circuit
- Signal validator: Checks signal schema compliance
- Aggregator: Combines with historical cohort data
- ML pipeline: Generates recommendations from aggregate
- Result handler: Returns results without any personal info
```

#### 5. Compliance & Auditing

```
Legal requirements to address:
- GDPR Art. 25: Privacy by design → ZK + anonymization
- CCPA § 1798.100: Right to know → No PII to disclose
- GDPR Art. 17: Right to delete → No personal data = automatic compliance
- Audit trail: Log proof verification for integrity, not content
```

### Design Decisions to Make

| Decision Point              | Option A | Option B          | Rationale                                                   |
| --------------------------- | -------- | ----------------- | ----------------------------------------------------------- |
| **Proof Language**          | circom   | Arkworks Circuits | circom = browser compatible; Arkworks = more efficient      |
| **Aggregation Granularity** | Per-user | Per-cohort        | Per-cohort prevents re-identification                       |
| **Retention Period**        | 7 days   | 30 days           | Balance: long enough for training, short enough for privacy |
| **Noise Level (ε)**         | 0.1      | 0.5               | Smaller ε = more privacy, less utility; 0.1 recommended     |
| **Proof Type**              | SNARK    | STARK             | SNARK = smaller proofs; STARK = post-quantum safe           |

### Success Metrics

**Privacy Metrics:**

- Zero successful re-identification attacks (target: 100%)
- Differential privacy parameter ε < 0.5 (target: < 0.1)
- Proof verification success rate > 99.9%
- No personal data breach incidents

**Performance Metrics:**

- Client-side proof generation: < 2 seconds
- Server proof verification: < 100ms
- End-to-end latency: < 500ms
- Recommendation inference: < 1 second

**Utility Metrics:**

- Recommendation accuracy: > 85% NDCG@10
- False positive rate (unverified proofs rejected): < 0.1%
- Coverage (% of users with at least 1 signal): > 80%

---

## Follow-Up Questions for Refinement

### Phase 1: Clarification

1. What signal types are most critical for relevance (engagement vs topic vs interaction)?
2. Is real-time inference required, or can recommendations be batch-processed?
3. How many users will this system serve initially (impacts aggregation granularity)?
4. What's the acceptable tradeoff between privacy and recommendation accuracy?

### Phase 2: Technical Deep-Dive

5. Which ZK proof system is preferred (SNARK vs STARK vs Plonk)?
6. Should proofs be generated client-side (latency) or server-side (simplicity)?
7. How to handle signal updates (can users modify signals after sending)?
8. What's the strategy for detecting/preventing proof forgery?

### Phase 3: Operational

9. How to run privacy audits on deployed system?
10. What's the incident response procedure if a proof is invalid?
11. How to monitor differential privacy loss over time?
12. Should there be a transparency report on privacy metrics?

---

## Deliverables Checklist

For a complete system design, address:

- [ ] **Architecture Diagram**: Client-server data flow showing ZK proof integration
- [ ] **Signal Specification**: Exact definitions of engagement, interaction, topic signals
- [ ] **Proof Circuit**: ZK circuit pseudocode or circom code for signal validation
- [ ] **Aggregation Rules**: Formal algorithm for combining anonymous signals
- [ ] **Privacy Proof**: Mathematical argument for why architecture prevents re-identification
- [ ] **Threat Model**: What attacks does ZK layer prevent? What remains?
- [ ] **Compliance Matrix**: Map each regulation requirement to system component
- [ ] **API Specification**: Client and server endpoint contracts
- [ ] **Database Schema**: What exactly is stored and for how long
- [ ] **Monitoring Dashboard**: What privacy metrics are tracked continuously

---

## References & Resources

### ZK-Proof Frameworks

- **circom**: https://docs.circom.io/ (circuit design language)
- **snarkjs**: https://github.com/iden3/snarkjs (proof generation & verification)
- **Arkworks**: https://arkworks.rs/ (Rust cryptographic backend)
- **o1js**: https://docs.minaprotocol.com/zk-program (TypeScript ZK framework)

### Privacy Techniques

- **Differential Privacy**: "The Algorithmic Foundations of Differential Privacy" (Dwork & Roth)
- **Data Minimization**: ISO/IEC 27001 privacy controls
- **K-Anonymity**: "k-Anonymity: A Model for Protecting Privacy" (Samarati & Sweeney)

### Regulatory References

- **GDPR Article 25**: Privacy by Design (https://gdpr-info.eu/art-25-gdpr/)
- **CCPA Section 1798.100**: Consumer Privacy Rights (https://oag.ca.gov/privacy/ccpa)
- **Privacy Impact Assessment**: Template from DPO community

---

## Example Implementation Sketch

### Client-Side (Pseudocode)

```javascript
// 1. Collect raw data
const userActivity = {
  viewCount: 47,
  topicEngaged: "Technology/AI/ML",
  interactionTypes: ["click", "scroll"],
  sessionDuration: 847, // seconds
};

// 2. Anonymize
const anonSignal = {
  engagement_bin: 2, // 47 views → bin [10-50]
  topic_category: "Technology", // Top-level only
  interaction_set: [1, 3], // Encoded type set
  duration_bucket: "long_session", // Categorized, not exact
};

// 3. Generate ZK proof
const circuit = new SignalValidityCircuit();
const proof = circuit.prove({
  signals: anonSignal,
  constraints: {
    maxEngagement: 1000,
    topicInSet: true,
    recentTimestamp: true,
  },
});

// 4. Send (no user ID!)
fetch("/api/signals", {
  method: "POST",
  body: JSON.stringify({
    anonymizedSignal: anonSignal,
    zkProof: proof,
    // NO userId, NO email, NO device ID
  }),
});
```

### Server-Side (Pseudocode)

```javascript
// 1. Receive signal + proof
const { anonymizedSignal, zkProof } = request.body;

// 2. Verify proof
const verified = verifyProof(zkProof, anonymizedSignal);
if (!verified) {
  return { status: "rejected" };
}

// 3. Aggregate (no disaggregation possible)
const cohort = await db.query(`
  SELECT AVG(engagement_bin), MODE(interaction_set)
  FROM signals
  WHERE topic_category = $1
  AND timestamp > NOW() - INTERVAL '30 days'
`);

// 4. Infer recommendations (using only aggregate)
const recommendations = mlModel.predict(cohort);

// 5. Return (no way to trace back to individual)
return { recommendations };
```
