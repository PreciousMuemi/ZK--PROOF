# Integration Guide

## Wiring Signal Quality Model into Week 1-2 System

**Objective**: Integrate trained ML model into the complete relevance engine

**Integration Points**:

- Signal processing pipeline
- Feature extraction layer
- Ranking/scoring system
- Monitoring & feedback loop

---

## 1. System Architecture Update

### Before (Weeks 1-2)

```
User Activity
    ↓
Anonymize & Validate (Week 2)
    ↓
ZK Proof (Week 2)
    ↓
Aggregate Signals
    ↓
Score by Topic Affinity
    ↓
Rank & Deliver
```

### After (Weeks 1-3)

```
User Activity
    ↓
Anonymize & Validate (Week 2)
    ├─ 7-layer signal validation
    ├─ Privacy checks
    └─ K-anonymity verification
    ↓
ZK Proof (Week 2)
    ├─ Prove signal is valid
    └─ No personal data exposed
    ↓
Extract Features (Week 3) ← NEW
    ├─ 29-feature transformation
    └─ Normalize for model
    ↓
Classify Quality (Week 3) ← NEW
    ├─ [XGBoost Model]
    ├─ HIGH-signal or LOW-signal
    └─ Confidence score
    ↓
Aggregate & Filter
    ├─ Separate high/low quality
    └─ Weight by quality + topic
    ↓
Score & Rank (Week 1)
    ├─ Topic affinity
    ├─ Quality classifier
    └─ Diversity
    ↓
Personalize & Deliver
    └─ Optimized recommendations
```

---

## 2. Data Flow: Signal → Classification

### Step 1: Signal Bundle (from Week 2)

```python
# After Week 2 validation, signal looks like:
signal = {
    'signal_id': 'sig_abc123',
    'timestamp_bin': 'afternoon',

    'topic_signals': {
        'primary_topic': 'Technology',
        'secondary_topics': ['AI', 'ML'],
        'topic_affinity': 0.75,
        'content_freshness_bin': 3,
        'content_depth_bin': 4,
        'creator_authority': 0.85,
        'is_controversial': False,
        'is_educational': True,
        'topic_diversity_score': 0.62,
        'content_format': 'article',
    },

    'engagement_signals': {
        'duration_bin': 4,
        'frequency_bin': 3,
        'scroll_depth_decile': 7,
        'engagement_intensity': 0.68,
        'return_probability': 0.71,
        'recency_bin': 2,
        'session_depth': 5,
        'completion_rate': 0.85,
    },

    'interaction_signals': {
        'has_click': True,
        'num_interactions': 2,
        'social_weight': 0.45,
        'bookmarked': True,
        'shared': False,
        'comment_length_bin': 0,
    },

    'quality_signals': {
        'content_quality_score': 0.78,
        'niche_fit': 0.81,
        'is_exploration': False,
        'topic_consistency': 0.88,
        'predicted_helpfulness': 0.73,
    }
}
# ✅ All 4 signal groups present
# ✅ Passed Week 2 validation
# ✅ Ready for Week 3 processing
```

### Step 2: Feature Extraction

```python
from week3_feature_engineering import FeatureExtractor

# Initialize extractor (once at startup)
feature_extractor = FeatureExtractor()

# For each signal, extract features
features = feature_extractor.extract(signal)
# Returns: pd.Series with 29 features

# Normalize for model input
X_normalized = feature_extractor.normalize(pd.DataFrame([features]))
# Returns: df with normalized features ready for model
```

### Step 3: Classification

```python
import joblib

# Load trained model (once at startup)
model = joblib.load('models/signal_quality_model.pkl')
scaler = joblib.load('models/scaler.pkl')

# For each signal's features
features_2d = X_normalized.values  # Shape: (1, 29)
prediction = model.predict(features_2d)[0]  # 0 or 1
confidence = model.predict_proba(features_2d)[0][prediction]

# Result
result = {
    'signal_id': signal['signal_id'],
    'quality_class': 'HIGH-signal' if prediction == 1 else 'LOW-signal',
    'quality_confidence': confidence,
    'quality_score': model.predict_proba(features_2d)[0][1],  # P(HIGH)
}
```

### Step 4: Integrated Signal Object

```python
# Augment original signal with quality classification
signal_with_quality = {
    **signal,  # Original signal from Week 2
    'quality_classification': {
        'class': 'HIGH-signal',
        'confidence': 0.87,
        'quality_score': 0.87,  # Probability of HIGH-signal
    }
}
# ✅ Ready for downstream ranking/personalization
```

---

## 3. Implementation: Signal Processing Pipeline

### 3.1 Complete Processing Function

```python
import pandas as pd
import joblib
from typing import Dict, Optional

class SignalProcessor:
    """
    Complete signal processing: anonymize → validate → classify
    """

    def __init__(self, model_path: str, scaler_path: str):
        """Initialize with trained model and scaler"""
        self.model = joblib.load(model_path)
        self.scaler = joblib.load(scaler_path)
        self.feature_extractor = FeatureExtractor()

        print("✅ Initialized SignalProcessor with trained model")

    def process(self, signal: Dict) -> Dict:
        """
        Complete signal processing pipeline

        Input: Raw signal from Week 2 validation
        Output: Signal with quality classification
        """
        try:
            # Step 1: Extract 29 features
            features = self.feature_extractor.extract(signal)

            # Step 2: Normalize
            X_normalized = self.feature_extractor.normalize(
                pd.DataFrame([features])
            )

            # Step 3: Classify
            X_array = X_normalized.values
            prediction = self.model.predict(X_array)[0]
            proba = self.model.predict_proba(X_array)[0]

            # Step 4: Augment signal
            signal_with_quality = {
                **signal,
                'quality_classification': {
                    'class': 'HIGH-signal' if prediction == 1 else 'LOW-signal',
                    'confidence': float(proba[prediction]),
                    'quality_score': float(proba[1]),  # P(HIGH-signal)
                }
            }

            return signal_with_quality

        except Exception as e:
            # Log error, return signal without quality (fail gracefully)
            print(f"⚠️  Error processing signal {signal.get('signal_id')}: {e}")
            signal['quality_classification'] = {
                'class': 'UNKNOWN',
                'confidence': 0.0,
                'quality_score': 0.5,  # Neutral default
            }
            return signal

    def process_batch(self, signals: list) -> list:
        """
        Process multiple signals efficiently

        Batch processing is faster than individual signals
        """
        processed = []

        for signal in signals:
            processed_signal = self.process(signal)
            processed.append(processed_signal)

        return processed

# Usage in relevance engine
processor = SignalProcessor('models/signal_quality_model.pkl',
                           'models/scaler.pkl')

# Single signal
signal = {...}  # From Week 2 validation
signal_with_quality = processor.process(signal)

# Batch of signals
signals = [...]  # List of signals
signals_with_quality = processor.process_batch(signals)
```

---

## 4. Ranking & Personalization Update

### 4.1 Updated Relevance Score

Before (Week 1):

```python
relevance_score = topic_affinity * 0.7 + freshness * 0.3
```

After (Week 3):

```python
def compute_relevance_score(signal_with_quality: Dict) -> float:
    """
    Updated scoring with quality classification
    """
    # Extract components
    topic_affinity = signal_with_quality['topic_signals']['topic_affinity']
    quality_score = signal_with_quality['quality_classification']['quality_score']
    freshness = signal_with_quality['topic_signals']['content_freshness_bin'] / 5

    # Weights
    w_topic = 0.50      # Topic affinity
    w_quality = 0.35    # Quality classification
    w_freshness = 0.15  # Freshness

    # Compute final score
    score = (
        topic_affinity * w_topic +
        quality_score * w_quality +
        freshness * w_freshness
    )

    # Boost high-signal, suppress low-signal
    if signal_with_quality['quality_classification']['class'] == 'HIGH-signal':
        score = score * 1.2  # 20% boost
    else:
        score = score * 0.8  # 20% suppression

    return score

# Usage
relevance_score = compute_relevance_score(signal_with_quality)
```

### 4.2 Filtering & Ranking

```python
def rank_and_filter(signals_with_quality: list,
                   top_k: int = 10,
                   quality_threshold: float = 0.5) -> list:
    """
    Rank signals and filter by quality
    """
    # Filter: Only include signals with sufficient quality confidence
    high_confidence = [
        s for s in signals_with_quality
        if s['quality_classification']['confidence'] >= 0.7
    ]

    # Score each signal
    scored_signals = [
        (compute_relevance_score(s), s)
        for s in high_confidence
    ]

    # Sort by relevance (descending)
    scored_signals.sort(key=lambda x: x[0], reverse=True)

    # Return top K
    top_signals = [s for _, s in scored_signals[:top_k]]

    return top_signals
```

---

## 5. Database Schema Update

### 5.1 Signal Table Update

Add columns to store quality classification:

```sql
-- Week 2 schema (existing)
CREATE TABLE signals (
    signal_id VARCHAR(50) PRIMARY KEY,
    timestamp_bin VARCHAR(20),
    primary_topic VARCHAR(30),
    engagement_duration_bin INT,
    -- ... other signal fields
    created_at TIMESTAMP,
    expires_at TIMESTAMP,
);

-- Week 3 additions
ALTER TABLE signals ADD COLUMN (
    quality_class VARCHAR(20),          -- HIGH-signal, LOW-signal, or UNKNOWN
    quality_confidence FLOAT,           -- 0-1, confidence in classification
    quality_score FLOAT,                -- 0-1, P(HIGH-signal)
    quality_model_version VARCHAR(20),  -- Track which model version made prediction
    classified_at TIMESTAMP,            -- When model made prediction
);

-- Index for filtering by quality
CREATE INDEX idx_quality_class ON signals(quality_class);
CREATE INDEX idx_quality_score ON signals(quality_score);
```

### 5.2 Query Examples

```sql
-- Get high-signal items for user topic
SELECT signal_id, quality_score, engagement_duration_bin
FROM signals
WHERE primary_topic = 'Technology'
  AND quality_class = 'HIGH-signal'
  AND quality_confidence >= 0.80
ORDER BY quality_score DESC
LIMIT 10;

-- Monitor classification distribution
SELECT quality_class, COUNT(*) as count, AVG(quality_score) as avg_score
FROM signals
WHERE classified_at >= NOW() - INTERVAL '7 days'
GROUP BY quality_class;

-- Find signals with low confidence (retrain signals)
SELECT COUNT(*) as uncertain_count
FROM signals
WHERE quality_confidence < 0.60
  AND classified_at >= NOW() - INTERVAL '1 day';
```

---

## 6. Monitoring & Feedback Loop

### 6.1 Performance Monitoring

```python
class QualityModelMonitor:
    """Monitor model performance in production"""

    def __init__(self, db_connection):
        self.db = db_connection

    def check_performance(self, days_back: int = 7):
        """
        Check if model predictions align with actual user behavior
        """
        # Get recent signals with labels
        query = f"""
        SELECT
            signal_id,
            quality_score,
            quality_class,
            user_engagement_score,  -- Actual engagement (label)
            classified_at
        FROM signals
        WHERE classified_at >= DATE_SUB(NOW(), INTERVAL {days_back} DAY)
          AND user_engagement_score IS NOT NULL
        """

        results = pd.read_sql(query, self.db)

        # Compare predictions vs actual
        # If user engaged with LOW-signal: model error
        # If user ignored HIGH-signal: model error

        errors = []
        for idx, row in results.iterrows():
            predicted_high = row['quality_class'] == 'HIGH-signal'
            actual_high = row['user_engagement_score'] > 0.5

            if predicted_high != actual_high:
                errors.append(row)

        error_rate = len(errors) / len(results)

        print(f"Quality model accuracy: {(1 - error_rate):.1%}")

        if error_rate > 0.25:  # >25% error
            print("🚨 ALERT: High error rate, trigger retraining")
            return False
        else:
            print("✅ Model performing well")
            return True

    def identify_retrain_signals(self) -> list:
        """
        Find signals to include in next training data
        """
        query = """
        SELECT signal_id
        FROM signals
        WHERE quality_confidence < 0.60
          AND classified_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
        """

        retrain_signals = pd.read_sql(query, self.db)
        print(f"Found {len(retrain_signals)} signals to retrain on")

        return retrain_signals['signal_id'].tolist()

# Usage
monitor = QualityModelMonitor(db_connection)
monitor.check_performance()
retrain_signals = monitor.identify_retrain_signals()
```

### 6.2 Retraining Automation

```python
def schedule_retraining(schedule_type: str = 'weekly'):
    """
    Automatically retrain model with new signals
    """
    import schedule
    import time

    def retrain_job():
        print("Starting scheduled retraining...")

        # 1. Collect new labeled signals
        new_signals = collect_labeled_signals(days=7)

        # 2. Run training pipeline
        model, metrics = train_model(new_signals)

        # 3. Validate new model
        if metrics['roc_auc'] >= 0.85:
            # 4. Deploy new model
            save_model(model, version=get_next_version())
            print("✅ New model deployed")
        else:
            print("❌ New model below performance targets, not deploying")

    if schedule_type == 'weekly':
        schedule.every().sunday.at("02:00").do(retrain_job)
    elif schedule_type == 'daily':
        schedule.every().day.at("02:00").do(retrain_job)

    # Run scheduler
    while True:
        schedule.run_pending()
        time.sleep(60)

# Usage (run in background worker)
# schedule_retraining('weekly')
```

---

## 7. API Integration

### 7.1 REST Endpoint

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# Initialize processor at startup
processor = SignalProcessor('models/signal_quality_model.pkl',
                           'models/scaler.pkl')

@app.route('/api/v1/classify-signal', methods=['POST'])
def classify_signal():
    """
    Endpoint: POST /api/v1/classify-signal

    Request:
    {
        "signal": {...signal object from Week 2...}
    }

    Response:
    {
        "signal_id": "sig_abc123",
        "quality_class": "HIGH-signal",
        "quality_confidence": 0.87,
        "quality_score": 0.87
    }
    """
    try:
        data = request.json
        signal = data.get('signal')

        # Process signal
        signal_with_quality = processor.process(signal)

        # Extract classification
        quality_info = signal_with_quality['quality_classification']

        return jsonify({
            'signal_id': signal['signal_id'],
            'quality_class': quality_info['class'],
            'quality_confidence': quality_info['confidence'],
            'quality_score': quality_info['quality_score'],
        }), 200

    except Exception as e:
        return jsonify({'error': str(e)}), 400

@app.route('/api/v1/classify-batch', methods=['POST'])
def classify_batch():
    """
    Batch endpoint for multiple signals
    """
    try:
        data = request.json
        signals = data.get('signals', [])

        # Process batch
        signals_with_quality = processor.process_batch(signals)

        # Extract classifications
        results = []
        for s in signals_with_quality:
            q = s['quality_classification']
            results.append({
                'signal_id': s['signal_id'],
                'quality_class': q['class'],
                'quality_confidence': q['confidence'],
                'quality_score': q['quality_score'],
            })

        return jsonify({'results': results}), 200

    except Exception as e:
        return jsonify({'error': str(e)}), 400

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 7.2 Client Usage

```python
import requests

# Classify single signal
signal = {...}
response = requests.post(
    'http://localhost:5000/api/v1/classify-signal',
    json={'signal': signal}
)
result = response.json()
print(f"Signal {result['signal_id']}: {result['quality_class']}")

# Classify batch
signals = [...]
response = requests.post(
    'http://localhost:5000/api/v1/classify-batch',
    json={'signals': signals}
)
results = response.json()['results']
```

---

## 8. Deployment Checklist

```
✅ PRE-DEPLOYMENT
├─ Model training complete
├─ Performance targets met (ROC-AUC > 0.85)
├─ Validation tests passed
├─ Privacy verified
├─ No data leakage confirmed
└─ Model serialized and saved

✅ DEPLOYMENT
├─ Load model into service
├─ Create FeatureExtractor instance
├─ Initialize SignalProcessor
├─ Create REST endpoint(s)
├─ Database schema updated
├─ Monitoring dashboards created
└─ Logging configured

✅ POST-DEPLOYMENT
├─ Verify inference latency < 100ms
├─ Monitor classification distribution
├─ Track performance metrics
├─ Collect user feedback
├─ Plan retraining schedule
└─ Document model version
```

---

## 9. Rollback Plan

If model causes issues in production:

```python
def rollback_to_previous_model():
    """
    Revert to previous model version
    """
    # 1. Stop using new model
    processor.model = load_previous_model_version()

    # 2. Alert operations team
    send_alert("Model rolled back to v1.2")

    # 3. Investigate issue
    # - Check error logs
    # - Compare predictions with v1.2
    # - Identify bad signals

    # 4. Retrain with fixes
    # - Add identified bad signals to training data
    # - Retrain model
    # - Validate improvements

    # 5. Deploy fixed version
    # - Test on staging
    # - Gradual rollout (10% → 50% → 100%)
    # - Monitor metrics

print("✅ Rollback procedure defined and tested")
```

---

## Next Steps

1. ✅ **Model trained** - Done
2. ✅ **Validation complete** - Done
3. → **Deploy to production** - Week 4
4. → **Monitor performance** - Ongoing
5. → **Retrain monthly** - Ongoing
