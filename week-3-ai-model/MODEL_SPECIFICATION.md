# ML Model Specification: Signal Quality Classification

## Week 3 - AI Relevance Model

**Objective**: Build lightweight ML model to classify anonymized activity signals as high-signal (meaningful) or low-signal (low-value)

**Model Type**: Binary classifier (high-signal vs low-signal)

**Training Data**: Anonymized engagement patterns + topic match scores

**Inference Latency Target**: < 100ms per signal

**Implementation Language**: Python (scikit-learn, PyTorch optional)

---

## 1. Problem Definition

### Classification Task

```
Input: Anonymized signal bundle
├── Topic signals (10 features)
├── Engagement signals (8 features)
├── Interaction signals (6 features)
└── Quality signals (5 features)
     ↓
Model predicts: High-Signal (1) or Low-Signal (0)
     ↓
Output: Classification + confidence score
```

### Label Definition

**HIGH-SIGNAL** (1): Signal indicates meaningful user engagement

- User spent significant time (>30s)
- Content matched user interests (topic alignment > 0.6)
- Explicit engagement action (click, scroll, share, not just view)
- Low abandonment risk (user returned within 7 days)
- Topic depth progression (visited similar topics multiple times)

**LOW-SIGNAL** (0): Signal indicates low-value or accidental engagement

- User abandoned quickly (<5s)
- Content topic mismatch (user doesn't typically read this category)
- Passive engagement only (view without interaction)
- One-off exposure (never returned to topic)
- Topic bounce (clicked then immediately left category)

### Dataset Statistics (Typical)

```
Total signals labeled: 100,000+
├─ High-signal: 40,000 (40%)
├─ Low-signal: 60,000 (60%)
Training set: 70,000 (70%)
Validation set: 15,000 (15%)
Test set: 15,000 (15%)
```

---

## 2. Model Architecture

### Option A: Lightweight Gradient Boosting (RECOMMENDED)

**Why**: Fast training, interpretable, no GPU needed, production-ready

```python
class SignalQualityModel:
    """
    Lightweight gradient boosting classifier for signal quality
    """
    def __init__(self):
        self.model = XGBClassifier(
            n_estimators=100,           # 100 trees
            max_depth=5,                # Shallow trees = interpretable
            learning_rate=0.1,          # Conservative learning
            subsample=0.8,              # Random 80% of samples
            colsample_bytree=0.8,       # Random 80% of features
            random_state=42
        )
        self.feature_scaler = StandardScaler()
        self.feature_names = None

    def train(self, X_train, y_train):
        """Train on anonymized features"""
        X_scaled = self.feature_scaler.fit_transform(X_train)
        self.model.fit(X_scaled, y_train)
        self.feature_names = X_train.columns.tolist()

    def predict(self, X):
        """Predict signal quality (0 or 1)"""
        X_scaled = self.feature_scaler.transform(X)
        return self.model.predict(X_scaled)

    def predict_proba(self, X):
        """Predict with confidence scores"""
        X_scaled = self.feature_scaler.transform(X)
        return self.model.predict_proba(X_scaled)

    def feature_importance(self):
        """Which features matter most?"""
        importance_df = pd.DataFrame({
            'feature': self.feature_names,
            'importance': self.model.feature_importances_
        }).sort_values('importance', ascending=False)
        return importance_df
```

**Advantages**:

- Fast training (seconds)
- Fast inference (1-5ms per signal)
- Interpretable feature importance
- Handles non-linear relationships
- Works with mixed feature types
- No GPU required
- Battle-tested in production

**Disadvantages**:

- Less flexible than neural networks
- Requires feature engineering

---

### Option B: Lightweight Neural Network

**Why**: If non-linearity is critical, use small neural net

```python
import torch
import torch.nn as nn

class SignalQualityNN(nn.Module):
    """
    Small neural network for signal quality classification
    ~50KB model size, 2-10ms inference
    """
    def __init__(self, input_dim=29):
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, 64),   # Input layer
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(64, 32),          # Hidden layer
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(32, 16),          # Hidden layer
            nn.ReLU(),
            nn.Linear(16, 1),           # Output layer
            nn.Sigmoid()                # Binary classification
        )

    def forward(self, x):
        return self.network(x)

    def predict(self, x_tensor):
        with torch.no_grad():
            output = self.forward(x_tensor)
            return (output > 0.5).int()

    def predict_proba(self, x_tensor):
        with torch.no_grad():
            return self.forward(x_tensor)
```

**Advantages**:

- Captures complex non-linear patterns
- Can be compressed/quantized for edge deployment
- Handles dynamic input sizes

**Disadvantages**:

- Requires more data to train well
- Harder to interpret decisions
- Needs validation to prevent overfitting

**Recommendation**: Start with XGBoost, move to NN only if XGBoost ROC-AUC < 0.85

---

## 3. Feature Engineering

### Input Features (29 total)

#### Topic Features (10)

```python
topic_features = {
    'primary_topic_code': int,              # 0-9 (Technology, Science, etc.)
    'num_secondary_topics': int,            # 0-3 (count of secondary topics)
    'topic_affinity_score': float,          # 0-1 (user affinity for primary topic)
    'topic_freshness_bin': int,             # 1-5 (recency of topic)
    'content_depth_bin': int,               # 1-5 (how deep/comprehensive)
    'creator_authority_score': float,       # 0-1 (creator credibility)
    'is_controversial': bool,               # 0-1 (divisive topics harder to recommend)
    'is_educational': bool,                 # 0-1 (educational boost signal quality)
    'topic_diversity': float,               # 0-1 (user explores variety)
    'content_type_match': float,            # 0-1 (user preference for article/video/etc)
}
```

#### Engagement Features (8)

```python
engagement_features = {
    'engagement_duration_bin': int,         # 1-5 (Skimmed to Deep Read)
    'engagement_frequency_bin': int,        # 1-5 (user frequency for this topic)
    'scroll_depth_bin': int,                # 1-10 (page depth visited)
    'engagement_intensity': float,          # 0-1 (normalized from frequency)
    'return_probability': float,            # 0-1 (likelihood user returns to topic)
    'recency_days_bin': int,                # 1-4 (how recent engagement)
    'session_depth': int,                   # 1-10 (# items in session)
    'completion_rate': float,               # 0-1 (if article/video consumed)
}
```

#### Interaction Features (6)

```python
interaction_features = {
    'has_explicit_action': bool,            # 1 if click/share/comment
    'num_interactions': int,                # 1-5 (count of interactions)
    'social_action_weight': float,          # 0-1 (normalized social actions)
    'bookmark_indicator': bool,             # 1 if user saved
    'share_indicator': bool,                # 1 if user shared
    'comment_length_bin': int,              # 0-3 (0=no comment, 1=short, etc)
}
```

#### Quality & Derived Features (5)

```python
quality_features = {
    'quality_score': float,                 # 0-1 (aggregated quality metrics)
    'niche_fit': float,                     # 0-1 (user niche alignment)
    'exploration_indicator': bool,          # 1 if user exploring new topics
    'consistency_score': float,             # 0-1 (topic consistency over time)
    'predicted_helpfulness': float,         # 0-1 (content helpfulness score)
}
```

### Feature Preparation Pipeline

```python
def prepare_features(raw_signal: dict) -> pd.DataFrame:
    """
    Convert anonymized signal bundle to model features

    Input: {
        'topic_signals': {...},
        'engagement_signals': {...},
        'interaction_signals': {...},
        'quality_signals': {...}
    }

    Output: DataFrame with 29 features ready for model
    """
    features = {}

    # Topic features
    features['primary_topic_code'] = map_topic_to_code(
        raw_signal['topic_signals']['primary_topic']
    )
    features['num_secondary_topics'] = len(
        raw_signal['topic_signals'].get('secondary_topics', [])
    )
    features['topic_affinity_score'] = raw_signal['topic_signals'].get(
        'topic_affinity', 0.5
    )
    # ... continue for all 29 features

    return pd.DataFrame([features])

# Feature transformations
def normalize_features(X: pd.DataFrame) -> pd.DataFrame:
    """Normalize continuous features to 0-1 range"""
    continuous_cols = [
        'topic_affinity_score', 'creator_authority_score',
        'engagement_intensity', 'return_probability',
        'social_action_weight', 'quality_score', 'niche_fit',
        'consistency_score', 'predicted_helpfulness', 'content_type_match'
    ]
    X_normalized = X.copy()
    scaler = MinMaxScaler()
    X_normalized[continuous_cols] = scaler.fit_transform(X[continuous_cols])
    return X_normalized
```

---

## 4. Model Training

### Training Pipeline

```python
class SignalQualityTrainer:
    """End-to-end training pipeline"""

    def __init__(self):
        self.model = SignalQualityModel()
        self.results = {}

    def load_data(self, path: str):
        """Load labeled signal data"""
        df = pd.read_csv(path)
        X = df.drop('label', axis=1)
        y = df['label']

        # Train/val/test split: 70/15/15
        X_train, X_temp, y_train, y_temp = train_test_split(
            X, y, test_size=0.3, random_state=42, stratify=y
        )
        X_val, X_test, y_val, y_test = train_test_split(
            X_temp, y_temp, test_size=0.5, random_state=42, stratify=y_temp
        )

        return X_train, X_val, X_test, y_train, y_val, y_test

    def train(self, X_train, y_train, X_val, y_val):
        """Train model and monitor validation performance"""
        print("Training signal quality model...")

        self.model.train(X_train, y_train)

        # Validation performance
        y_pred = self.model.predict(X_val)
        y_proba = self.model.predict_proba(X_val)

        self.results['val_accuracy'] = accuracy_score(y_val, y_pred)
        self.results['val_precision'] = precision_score(y_val, y_pred)
        self.results['val_recall'] = recall_score(y_val, y_pred)
        self.results['val_f1'] = f1_score(y_val, y_pred)
        self.results['val_roc_auc'] = roc_auc_score(y_val, y_proba[:, 1])

        print(f"Validation Accuracy: {self.results['val_accuracy']:.3f}")
        print(f"Validation ROC-AUC: {self.results['val_roc_auc']:.3f}")

        return self.model

    def evaluate(self, X_test, y_test):
        """Final evaluation on held-out test set"""
        y_pred = self.model.predict(X_test)
        y_proba = self.model.predict_proba(X_test)

        results = {
            'accuracy': accuracy_score(y_test, y_pred),
            'precision': precision_score(y_test, y_pred),
            'recall': recall_score(y_test, y_pred),
            'f1': f1_score(y_test, y_pred),
            'roc_auc': roc_auc_score(y_test, y_proba[:, 1]),
        }

        # Confusion matrix
        cm = confusion_matrix(y_test, y_pred)
        results['true_negatives'] = cm[0, 0]
        results['false_positives'] = cm[0, 1]
        results['false_negatives'] = cm[1, 0]
        results['true_positives'] = cm[1, 1]

        return results
```

### Hyperparameter Recommendations

```python
# For XGBoost
xgb_params = {
    'n_estimators': [50, 100, 150],         # Number of boosting rounds
    'max_depth': [3, 5, 7],                 # Tree depth
    'learning_rate': [0.01, 0.05, 0.1],    # Step size
    'subsample': [0.6, 0.8, 1.0],          # Row sampling
    'colsample_bytree': [0.6, 0.8, 1.0],   # Column sampling
}

# Grid search
best_params = GridSearchCV(
    XGBClassifier(),
    xgb_params,
    scoring='roc_auc',
    cv=5
).fit(X_train, y_train).best_params_

# For Neural Network
nn_params = {
    'hidden_dim': [32, 64, 128],
    'dropout_rate': [0.1, 0.2, 0.3],
    'learning_rate': [0.001, 0.01, 0.1],
    'batch_size': [32, 64, 128],
    'epochs': [10, 20, 50],
}
```

---

## 5. Performance Targets

### Accuracy Metrics

| Metric        | Target | Rationale                                        |
| ------------- | ------ | ------------------------------------------------ |
| **ROC-AUC**   | > 0.85 | Can distinguish signal quality                   |
| **Precision** | > 0.80 | When model says "high-signal", it's correct 80%+ |
| **Recall**    | > 0.75 | Catches 75%+ of actual high-signal items         |
| **F1-Score**  | > 0.77 | Balanced precision/recall                        |
| **Accuracy**  | > 0.78 | Overall correctness                              |

### Inference Performance

| Target         | Value              | Rationale                 |
| -------------- | ------------------ | ------------------------- |
| **Latency**    | < 100ms            | Real-time recommendations |
| **Throughput** | > 1000 signals/sec | Handle concurrent users   |
| **Memory**     | < 100MB            | Can fit in memory/cache   |
| **Model Size** | < 50MB             | Can ship in updates       |

### Data Balance

| Class       | Target | Why                                 |
| ----------- | ------ | ----------------------------------- |
| High-Signal | ~40%   | Realistic platform distribution     |
| Low-Signal  | ~60%   | Natural skew towards low engagement |

---

## 6. Class Imbalance Handling

Since ~60% of signals are low-value, we must handle class imbalance:

```python
# Strategy 1: Class Weights (RECOMMENDED)
class_weights = compute_class_weight(
    'balanced',
    classes=np.unique(y_train),
    y=y_train
)
# XGBoost
model = XGBClassifier(
    scale_pos_weight=class_weights[1] / class_weights[0]
)

# Strategy 2: Oversampling (if needed for recall)
from imblearn.over_sampling import SMOTE
smote = SMOTE(random_state=42)
X_resampled, y_resampled = smote.fit_resample(X_train, y_train)

# Strategy 3: Threshold tuning (post-training)
# Lower decision threshold to catch more high-signal
y_pred = (y_proba[:, 1] > 0.3).astype(int)  # Instead of 0.5
```

---

## 7. Feature Importance Analysis

### Top Predictive Features (Typical)

```python
# After training, analyze which features matter most
feature_importance = model.feature_importance()

# Expected top features:
# 1. engagement_duration_bin (strong indicator of quality)
# 2. topic_affinity_score (user interests matter)
# 3. return_probability (repeat visits = quality signal)
# 4. has_explicit_action (passive vs active engagement)
# 5. scroll_depth_bin (shows content consumption)
# 6. creator_authority_score (credible sources)
# 7. content_depth_bin (substantive content)
# 8. engagement_frequency_bin (topic familiarity)
# ... rest less important
```

### Interpreting Predictions

```python
def interpret_prediction(signal: dict, model_prediction: dict) -> str:
    """
    Explain why model classified signal
    """
    explanation = f"""
    Signal Classification: {'HIGH-SIGNAL' if model_prediction['prediction'] else 'LOW-SIGNAL'}
    Confidence: {model_prediction['confidence']:.1%}

    Key Factors:
    - Duration engagement: {signal['engagement_duration_bin']}/5
    - Topic fit: {signal['topic_affinity_score']:.0%}
    - User return rate: {signal['return_probability']:.0%}
    - Explicit actions: {signal['has_explicit_action']}
    - Page scroll depth: {signal['scroll_depth_bin']}/10
    """
    return explanation
```

---

## 8. Model Monitoring & Retraining

### Monitoring Metrics

Track these in production:

```python
monitoring_metrics = {
    'prediction_distribution': {
        'high_signal_ratio': 0.40,          # Should stay ~40%
        'confidence_mean': 0.72,            # Avg confidence
    },
    'data_drift': {
        'feature_mean_shift': 0.05,         # Allowed 5% shift
        'feature_std_shift': 0.10,          # Allowed 10% shift
    },
    'performance_drift': {
        'roc_auc': 0.85,                    # Min acceptable ROC-AUC
        'precision': 0.80,                  # Min acceptable precision
    },
}

# Alert if metrics drift beyond thresholds
```

### Retraining Schedule

```
Retraining cadence:
- Every 2 weeks: Check data/performance drift
- If drift detected: Retrain immediately
- Monthly: Full retraining with new data
- Quarterly: Feature engineering review
- Annually: Architecture/algorithm evaluation
```

---

## 9. Model Deployment

### Export & Serialization

```python
import joblib

# Save trained model
joblib.dump(model, 'signal_quality_model.pkl')

# Load for inference
model = joblib.load('signal_quality_model.pkl')

# Convert to ONNX for cross-platform compatibility
import skl2onnx
onnx_model = skl2nnx.convert_sklearn(model, initial_types=[...])
```

### Integration with Week 1-2 System

```
Relevance Engine Pipeline (Updated):

Raw Signal
    ↓
Anonymize (Week 2)
    ↓
Validate 7-layer (Week 2)
    ↓
Generate ZK Proof (Week 2)
    ↓
Verify Proof (Week 2)
    ↓
Extract Features (Week 3) ← NEW
    ↓
Classify Quality [HIGH/LOW] (Week 3) ← NEW
    ↓
Score by Topic + Quality (Week 1)
    ↓
Personalize & Rank
    ↓
Deliver Recommendations
```

---

## 10. Privacy Considerations

### Model-Level Privacy

```
NO personal data in model:
✅ Features are binned (frequency ranges, not exact counts)
✅ Topics are categories, not specific content
✅ No user identifiers stored
✅ Training data aggregated (1000+ users per signal)
✅ Model training on aggregated statistics only

Model outputs are safe:
✅ Classification (HIGH/LOW) is not personal info
✅ Confidence scores are generic
✅ Feature importance reveals nothing about individuals
```

### Adversarial Considerations

**Q**: Can someone reverse-engineer user data from the model?
**A**: No. The model never sees raw data—only binned features aggregated from 1000+ users.

**Q**: Can model outputs be used to re-identify users?
**A**: No. Output is just "high-signal" or "low-signal" for a signal bundle, not linked to individuals.

**Q**: What if someone trains their own model on public outputs?
**A**: The outputs (classification + confidence) don't contain information about specific users. Aggregation irreversibility ensures no re-identification.

---

## 11. Success Criteria

**Model is successful if**:

- ✅ ROC-AUC > 0.85 on test set
- ✅ Inference latency < 100ms per signal
- ✅ Correctly identifies 75%+ of high-signal items (recall > 0.75)
- ✅ Misidentifies < 20% of low-signal as high-signal (precision > 0.80)
- ✅ Maintains privacy (no personal data exposure)
- ✅ Can be retrained weekly without infrastructure changes
- ✅ Integrates seamlessly with Week 1-2 system
- ✅ Improves recommendation relevance metrics (CTR, dwell time)

---

## Next Steps

1. **Implement**: Create training pipeline (TRAINING_GUIDE.md)
2. **Validate**: Test on holdout data (VALIDATION_GUIDE.md)
3. **Integrate**: Wire into relevance engine (INTEGRATION_GUIDE.md)
4. **Monitor**: Track performance in production
5. **Iterate**: Retrain with new signals monthly
