# Feature Engineering for Signal Quality Classification
## Converting Anonymized Signals to Model Features

**Objective**: Transform anonymized signal bundles into 29 machine-learning features while maintaining privacy

**Feature Count**: 29 total (10 topic + 8 engagement + 6 interaction + 5 quality)

**Data Format**: Each signal bundle → 1 feature vector ready for model

---

## 1. Input Signal Bundle Format

Expected input from Week 2 signal validation:

```python
signal_bundle = {
    'signal_id': 'sig_xyz123',              # Ephemeral ID only
    'timestamp_bin': 'afternoon',           # Binned, not exact time
    
    'topic_signals': {
        'primary_topic': 'Technology',      # Category only
        'secondary_topics': ['AI', 'ML'],   # 0-3 topics
        'topic_affinity': 0.75,             # 0-1 user fit score
        'content_freshness_bin': 3,         # 1-5 (old to brand new)
        'content_depth_bin': 4,             # 1-5 (shallow to deep)
        'creator_authority': 0.85,          # 0-1 credibility
        'is_controversial': False,          # Boolean
        'is_educational': True,             # Boolean
        'topic_diversity_score': 0.62,      # 0-1 user variety
        'content_format': 'article',        # Type: article/video/podcast
    },
    
    'engagement_signals': {
        'duration_bin': 4,                  # 1-5: Skimmed to Deep Read
        'frequency_bin': 3,                 # 1-5: How often user engages this topic
        'scroll_depth_decile': 7,           # 1-10: Page depth visited
        'engagement_intensity': 0.68,       # 0-1: Normalized strength
        'return_probability': 0.71,         # 0-1: Likelihood of return
        'recency_bin': 2,                   # 1-4: Recent, moderate, old
        'session_depth': 5,                 # 1-10: Items in session
        'completion_rate': 0.85,            # 0-1: If consumable content
    },
    
    'interaction_signals': {
        'has_click': True,                  # Any interaction?
        'num_interactions': 2,              # 1-5: Count of actions
        'social_weight': 0.45,              # 0-1: Social action strength
        'bookmarked': True,                 # Boolean
        'shared': False,                    # Boolean
        'comment_length_bin': 0,            # 0-3: 0=none, 1=short, 2=medium, 3=long
    },
    
    'quality_signals': {
        'content_quality_score': 0.78,      # 0-1: Aggregated quality
        'niche_fit': 0.81,                  # 0-1: Niche alignment
        'is_exploration': False,            # Boolean: New topic for user?
        'topic_consistency': 0.88,          # 0-1: Topic continuity
        'predicted_helpfulness': 0.73,      # 0-1: Content usefulness
    }
}
```

---

## 2. Feature Extraction Pipeline

### Step 1: Topic Features (10 features)

```python
def extract_topic_features(signal: dict) -> dict:
    """Extract 10 topic-related features"""
    topic_signals = signal['topic_signals']
    
    features = {
        # Feature 1: Primary topic code
        'primary_topic_code': topic_code_mapping(
            topic_signals['primary_topic']
        ),
        # 0=Technology, 1=Science, 2=Business, 3=Entertainment,
        # 4=Health, 5=Politics, 6=Sports, 7=Culture, 8=Education, 9=Other
        
        # Feature 2: Count of secondary topics
        'num_secondary_topics': min(
            len(topic_signals.get('secondary_topics', [])), 3
        ),
        # Capped at 3 (max secondary topics)
        
        # Feature 3: Topic affinity (user fit)
        'topic_affinity_score': topic_signals.get('topic_affinity', 0.5),
        # Raw 0-1 score, no transformation
        
        # Feature 4: Content freshness
        'content_freshness_bin': topic_signals.get('content_freshness_bin', 3),
        # Already binned 1-5
        
        # Feature 5: Content depth
        'content_depth_bin': topic_signals.get('content_depth_bin', 2),
        # Already binned 1-5
        
        # Feature 6: Creator authority
        'creator_authority_score': topic_signals.get('creator_authority', 0.5),
        # Raw 0-1 score
        
        # Feature 7: Is controversial
        'is_controversial': int(topic_signals.get('is_controversial', False)),
        # Convert boolean to 0/1
        
        # Feature 8: Is educational
        'is_educational': int(topic_signals.get('is_educational', False)),
        # Convert boolean to 0/1
        
        # Feature 9: Topic diversity
        'topic_diversity': topic_signals.get('topic_diversity_score', 0.5),
        # Raw 0-1 score
        
        # Feature 10: Content type match
        'content_type_match': format_preference_score(
            topic_signals.get('content_format', 'article'),
            user_preferences  # Requires user context
        ),
        # 0-1: How well format matches user preference
    }
    
    return features

def topic_code_mapping(topic_name: str) -> int:
    """Map topic name to numeric code"""
    mapping = {
        'Technology': 0,
        'Science': 1,
        'Business': 2,
        'Entertainment': 3,
        'Health': 4,
        'Politics': 5,
        'Sports': 6,
        'Culture': 7,
        'Education': 8,
        'Other': 9,
    }
    return mapping.get(topic_name, 9)
```

### Step 2: Engagement Features (8 features)

```python
def extract_engagement_features(signal: dict) -> dict:
    """Extract 8 engagement-related features"""
    engagement_signals = signal['engagement_signals']
    
    features = {
        # Feature 11: Engagement duration
        'engagement_duration_bin': engagement_signals.get('duration_bin', 2),
        # Already binned 1-5: Skimmed, Browsed, Read, Deep Read
        
        # Feature 12: Topic engagement frequency
        'engagement_frequency_bin': engagement_signals.get('frequency_bin', 2),
        # Already binned 1-5: How often user engages
        
        # Feature 13: Scroll depth (decile)
        'scroll_depth_bin': engagement_signals.get('scroll_depth_decile', 5),
        # Already binned 1-10
        
        # Feature 14: Engagement intensity (normalized)
        'engagement_intensity': engagement_signals.get('engagement_intensity', 0.5),
        # Raw 0-1 score
        
        # Feature 15: Return probability
        'return_probability': engagement_signals.get('return_probability', 0.5),
        # Raw 0-1: likelihood user returns to this topic
        
        # Feature 16: Recency
        'recency_bin': engagement_signals.get('recency_bin', 2),
        # Already binned 1-4: Recent, Moderate, Old, Very Old
        
        # Feature 17: Session depth
        'session_depth': min(
            engagement_signals.get('session_depth', 3), 10
        ),
        # Already binned 1-10, capped at 10
        
        # Feature 18: Completion rate (for videos/articles)
        'completion_rate': engagement_signals.get('completion_rate', 0.5),
        # Raw 0-1: % of content consumed
    }
    
    return features
```

### Step 3: Interaction Features (6 features)

```python
def extract_interaction_features(signal: dict) -> dict:
    """Extract 6 interaction-related features"""
    interaction_signals = signal['interaction_signals']
    
    features = {
        # Feature 19: Has any explicit action
        'has_explicit_action': int(
            interaction_signals.get('has_click', False)
        ),
        # 0/1: Whether user took explicit action
        
        # Feature 20: Number of interactions
        'num_interactions': min(
            interaction_signals.get('num_interactions', 0), 5
        ),
        # Already counted, capped at 5
        
        # Feature 21: Social action weight
        'social_action_weight': interaction_signals.get('social_weight', 0.0),
        # Raw 0-1: Normalized social actions (like, share, comment)
        
        # Feature 22: Bookmarked
        'bookmarked': int(
            interaction_signals.get('bookmarked', False)
        ),
        # 0/1: User saved content
        
        # Feature 23: Shared
        'shared': int(
            interaction_signals.get('shared', False)
        ),
        # 0/1: User shared content
        
        # Feature 24: Comment length
        'comment_length_bin': interaction_signals.get('comment_length_bin', 0),
        # Already binned 0-3: None, Short, Medium, Long
    }
    
    return features
```

### Step 4: Quality & Derived Features (5 features)

```python
def extract_quality_features(signal: dict) -> dict:
    """Extract 5 quality-related features"""
    quality_signals = signal['quality_signals']
    
    features = {
        # Feature 25: Content quality score
        'quality_score': quality_signals.get('content_quality_score', 0.5),
        # Raw 0-1: Aggregated quality metrics
        
        # Feature 26: Niche fit
        'niche_fit': quality_signals.get('niche_fit', 0.5),
        # Raw 0-1: How well content fits user's niche
        
        # Feature 27: Exploration indicator
        'exploration_indicator': int(
            quality_signals.get('is_exploration', False)
        ),
        # 0/1: Is this a new topic for the user?
        
        # Feature 28: Topic consistency
        'consistency_score': quality_signals.get('topic_consistency', 0.5),
        # Raw 0-1: Topic continuity (user sticks to interests)
        
        # Feature 29: Predicted helpfulness
        'predicted_helpfulness': quality_signals.get('predicted_helpfulness', 0.5),
        # Raw 0-1: Content usefulness score
    }
    
    return features
```

### Step 5: Combine All Features

```python
def extract_all_features(signal: dict, user_prefs: dict = None) -> pd.Series:
    """
    Extract all 29 features from signal bundle
    
    Returns:
        pd.Series with 29 features, ready for model
    """
    features = {}
    
    # Extract from each signal group
    features.update(extract_topic_features(signal))
    features.update(extract_engagement_features(signal))
    features.update(extract_interaction_features(signal))
    features.update(extract_quality_features(signal))
    
    # Verify we have exactly 29 features
    assert len(features) == 29, f"Expected 29 features, got {len(features)}"
    
    # Return as Series with proper ordering
    feature_order = [
        'primary_topic_code', 'num_secondary_topics', 'topic_affinity_score',
        'content_freshness_bin', 'content_depth_bin', 'creator_authority_score',
        'is_controversial', 'is_educational', 'topic_diversity', 'content_type_match',
        'engagement_duration_bin', 'engagement_frequency_bin', 'scroll_depth_bin',
        'engagement_intensity', 'return_probability', 'recency_bin',
        'session_depth', 'completion_rate',
        'has_explicit_action', 'num_interactions', 'social_action_weight',
        'bookmarked', 'shared', 'comment_length_bin',
        'quality_score', 'niche_fit', 'exploration_indicator',
        'consistency_score', 'predicted_helpfulness'
    ]
    
    return pd.Series({k: features[k] for k in feature_order})
```

---

## 3. Feature Normalization

### Continuous Features (0-1 range)

These features are already in 0-1 range or need normalization:

```python
def normalize_continuous_features(X: pd.DataFrame) -> pd.DataFrame:
    """
    Normalize continuous features to 0-1 range
    
    Ensures model receives consistent scale
    """
    continuous_cols = [
        'topic_affinity_score',           # Already 0-1
        'creator_authority_score',        # Already 0-1
        'engagement_intensity',           # Already 0-1
        'return_probability',             # Already 0-1
        'social_action_weight',           # Already 0-1
        'completion_rate',                # Already 0-1
        'quality_score',                  # Already 0-1
        'niche_fit',                      # Already 0-1
        'consistency_score',              # Already 0-1
        'predicted_helpfulness',          # Already 0-1
        'topic_diversity',                # Already 0-1
        'content_type_match',             # Already 0-1
    ]
    
    X_normalized = X.copy()
    scaler = StandardScaler()
    
    # Standardize continuous features
    X_normalized[continuous_cols] = scaler.fit_transform(
        X[continuous_cols]
    )
    
    return X_normalized, scaler
```

### Binned Features (1-N range)

These are already categorical, no normalization needed:

```python
binned_cols = [
    'primary_topic_code',               # 0-9
    'num_secondary_topics',             # 0-3
    'content_freshness_bin',            # 1-5
    'content_depth_bin',                # 1-5
    'engagement_duration_bin',          # 1-5
    'engagement_frequency_bin',         # 1-5
    'scroll_depth_bin',                 # 1-10
    'recency_bin',                      # 1-4
    'session_depth',                    # 1-10
    'comment_length_bin',               # 0-3
]

# No scaling needed—these are already properly distributed
```

### Boolean Features (0/1)

```python
boolean_cols = [
    'is_controversial',
    'is_educational',
    'has_explicit_action',
    'bookmarked',
    'shared',
    'exploration_indicator',
]

# Already 0/1, no transformation needed
```

---

## 4. Handling Missing Values

### Strategy: Use Defaults

```python
def handle_missing_values(signal: dict) -> dict:
    """
    Fill missing signal fields with sensible defaults
    
    This prevents crashes but should be rare
    (Week 2 validation should catch all missing required fields)
    """
    defaults = {
        'topic_affinity': 0.5,              # Neutral fit
        'topic_diversity_score': 0.5,       # Unknown diversity
        'engagement_intensity': 0.5,        # Neutral intensity
        'return_probability': 0.3,          # Low return (conservative)
        'completion_rate': 0.5,             # Assume 50% completion
        'social_weight': 0.0,               # No social by default
        'content_quality_score': 0.5,       # Neutral quality
        'niche_fit': 0.5,                   # Neutral fit
        'topic_consistency': 0.5,           # Unknown consistency
        'predicted_helpfulness': 0.5,       # Neutral helpfulness
    }
    
    # Fill any missing fields with defaults
    filled_signal = deep_merge(signal, defaults)
    
    return filled_signal

# Never drop records due to missing values
# (All signals should be complete after Week 2 validation)
```

---

## 5. Feature Scaling for Different Models

### For XGBoost (Minimal Scaling Needed)

```python
def prepare_features_xgboost(X: pd.DataFrame) -> pd.DataFrame:
    """
    Prepare features for XGBoost
    
    XGBoost handles feature scaling internally,
    but we normalize for consistency
    """
    X_prep = X.copy()
    
    # Only normalize continuous features
    continuous_cols = [
        'topic_affinity_score', 'creator_authority_score',
        'engagement_intensity', 'return_probability',
        'social_action_weight', 'completion_rate',
        'quality_score', 'niche_fit', 'consistency_score',
        'predicted_helpfulness', 'topic_diversity', 'content_type_match'
    ]
    
    scaler = StandardScaler()
    X_prep[continuous_cols] = scaler.fit_transform(X_prep[continuous_cols])
    
    return X_prep, scaler
```

### For Neural Network (Full Normalization)

```python
def prepare_features_neural_network(X: pd.DataFrame) -> tuple:
    """
    Prepare features for neural network
    
    Neural networks need all features normalized to similar scale
    """
    X_prep = X.copy()
    
    # 1. Normalize continuous features
    continuous_cols = [
        'topic_affinity_score', 'creator_authority_score',
        'engagement_intensity', 'return_probability',
        'social_action_weight', 'completion_rate',
        'quality_score', 'niche_fit', 'consistency_score',
        'predicted_helpfulness', 'topic_diversity', 'content_type_match'
    ]
    
    scaler_continuous = StandardScaler()
    X_prep[continuous_cols] = scaler_continuous.fit_transform(
        X_prep[continuous_cols]
    )
    
    # 2. Normalize binned features to 0-1
    binned_cols = [
        'primary_topic_code', 'num_secondary_topics',
        'content_freshness_bin', 'content_depth_bin',
        'engagement_duration_bin', 'engagement_frequency_bin',
        'scroll_depth_bin', 'recency_bin', 'session_depth',
        'comment_length_bin'
    ]
    
    scaler_binned = MinMaxScaler()
    X_prep[binned_cols] = scaler_binned.fit_transform(
        X_prep[binned_cols]
    )
    
    # Boolean features already 0/1, no change
    
    return X_prep, (scaler_continuous, scaler_binned)
```

---

## 6. Feature Quality Checks

### Validate Feature Extraction

```python
def validate_features(feature_vector: pd.Series) -> bool:
    """
    Validate that extracted features are in expected ranges
    
    Returns True if valid, False with warnings if not
    """
    validation_rules = {
        # Binned features should be integers
        'primary_topic_code': (0, 9, 'int'),
        'num_secondary_topics': (0, 3, 'int'),
        'content_freshness_bin': (1, 5, 'int'),
        'content_depth_bin': (1, 5, 'int'),
        'engagement_duration_bin': (1, 5, 'int'),
        'engagement_frequency_bin': (1, 5, 'int'),
        'scroll_depth_bin': (1, 10, 'int'),
        'recency_bin': (1, 4, 'int'),
        'session_depth': (1, 10, 'int'),
        'comment_length_bin': (0, 3, 'int'),
        
        # Continuous features should be 0-1
        'topic_affinity_score': (0, 1, 'float'),
        'creator_authority_score': (0, 1, 'float'),
        'engagement_intensity': (0, 1, 'float'),
        'return_probability': (0, 1, 'float'),
        'social_action_weight': (0, 1, 'float'),
        'completion_rate': (0, 1, 'float'),
        'quality_score': (0, 1, 'float'),
        'niche_fit': (0, 1, 'float'),
        'consistency_score': (0, 1, 'float'),
        'predicted_helpfulness': (0, 1, 'float'),
        'topic_diversity': (0, 1, 'float'),
        'content_type_match': (0, 1, 'float'),
        
        # Boolean features should be 0 or 1
        'is_controversial': (0, 1, 'int'),
        'is_educational': (0, 1, 'int'),
        'has_explicit_action': (0, 1, 'int'),
        'bookmarked': (0, 1, 'int'),
        'shared': (0, 1, 'int'),
        'exploration_indicator': (0, 1, 'int'),
    }
    
    all_valid = True
    
    for feature, (min_val, max_val, expected_type) in validation_rules.items():
        value = feature_vector[feature]
        
        # Check type
        if expected_type == 'int' and not isinstance(value, (int, np.integer)):
            print(f"⚠️  {feature}: Expected int, got {type(value)}")
            all_valid = False
        
        # Check range
        if value < min_val or value > max_val:
            print(f"⚠️  {feature}: Value {value} outside range [{min_val}, {max_val}]")
            all_valid = False
    
    return all_valid
```

---

## 7. Feature Engineering Best Practices

### DO ✅

```python
# 1. Keep features independent
# ❌ BAD: feature_1 = engagement_duration_bin + scroll_depth_bin
# ✅ GOOD: Keep as separate features, let model learn relationships

# 2. Maintain privacy during feature engineering
# ✅ GOOD: Map topic to category code (0-9)
# ✅ GOOD: Bin engagement into buckets
# ✅ GOOD: Use aggregated scores (already computed)

# 3. Document feature transformations
"""
Feature extraction pipeline:
1. Parse signal bundle
2. Extract 29 features from 4 signal groups
3. Validate ranges
4. Normalize continuous features
5. Return feature vector ready for model
"""

# 4. Version control feature lists
FEATURE_V1_0 = [
    'primary_topic_code', 'num_secondary_topics', ...
]

# 5. Add metadata to features
feature_metadata = {
    'primary_topic_code': {
        'type': 'categorical',
        'range': [0, 9],
        'source': 'topic_signals',
        'privacy_risk': 'Low (topic is category, not specific)',
    },
    # ... for each feature
}
```

### DON'T ❌

```python
# 1. Don't expose user IDs or identifiable info
# ❌ BAD: features['user_id'] = signal['user_id']

# 2. Don't use raw timestamps or exact engagement counts
# ❌ BAD: features['exact_time'] = signal['timestamp']
# ❌ BAD: features['exact_clicks'] = signal['click_count']

# 3. Don't concatenate unbinned values
# ❌ BAD: features['user_pattern'] = str(signal['interactions'])

# 4. Don't create leakage features
# ❌ BAD: Including the label in features
# ✅ GOOD: Features are input only, label is separate

# 5. Don't skip validation
# ❌ BAD: Pass features to model without range checks
# ✅ GOOD: Validate all features before training
```

---

## 8. Full Feature Extraction Example

```python
import pandas as pd
import numpy as np

class FeatureExtractor:
    """Complete feature engineering pipeline"""
    
    def __init__(self):
        self.feature_names = None
        self.scaler = None
    
    def extract(self, signal_bundle: dict) -> pd.Series:
        """
        Extract all features from signal bundle
        
        Input: Signal bundle from Week 2 validation
        Output: Feature vector ready for model
        """
        # Handle missing values
        signal_bundle = self._handle_missing(signal_bundle)
        
        # Extract 29 features
        features = {}
        features.update(self._extract_topic(signal_bundle))
        features.update(self._extract_engagement(signal_bundle))
        features.update(self._extract_interaction(signal_bundle))
        features.update(self._extract_quality(signal_bundle))
        
        # Create Series with proper column order
        feature_vector = pd.Series(features)
        
        # Validate ranges
        if not self._validate(feature_vector):
            raise ValueError("Feature validation failed")
        
        return feature_vector
    
    def _handle_missing(self, signal: dict) -> dict:
        """Fill any missing fields with defaults"""
        defaults = {
            'topic_affinity': 0.5,
            'return_probability': 0.3,
            # ... full defaults
        }
        return {**defaults, **signal}
    
    def _extract_topic(self, signal: dict) -> dict:
        """Extract 10 topic features"""
        # Implementation from Step 1
        pass
    
    def _extract_engagement(self, signal: dict) -> dict:
        """Extract 8 engagement features"""
        # Implementation from Step 2
        pass
    
    def _extract_interaction(self, signal: dict) -> dict:
        """Extract 6 interaction features"""
        # Implementation from Step 3
        pass
    
    def _extract_quality(self, signal: dict) -> dict:
        """Extract 5 quality features"""
        # Implementation from Step 4
        pass
    
    def _validate(self, feature_vector: pd.Series) -> bool:
        """Validate feature ranges"""
        # Implementation from Section 6
        pass
    
    def normalize(self, X: pd.DataFrame) -> pd.DataFrame:
        """Normalize features for model input"""
        # Implementation from Section 3
        pass


# Usage
extractor = FeatureExtractor()

# Extract features from single signal
signal = {...}  # From Week 2
features = extractor.extract(signal)

# Extract from batch
signals = [...]  # List of signal bundles
feature_df = pd.DataFrame([extractor.extract(s) for s in signals])
feature_df_normalized = extractor.normalize(feature_df)
```

---

## Next Steps

1. **Implement** feature extraction pipeline (code examples in IMPLEMENTATION_GUIDE.md)
2. **Test** with sample signal bundles
3. **Validate** all features pass range checks
4. **Normalize** for model input
5. **Train** ML model (TRAINING_GUIDE.md)

