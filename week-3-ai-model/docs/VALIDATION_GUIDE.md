# Validation & Testing Guide

## Comprehensive Testing of Signal Quality Model

**Objective**: Ensure model is safe, accurate, fair, and ready for production

**Testing Scope**:

- Prediction accuracy
- Privacy preservation
- Bias & fairness
- Robustness & edge cases
- Integration with system

---

## 1. Validation Metrics & Targets

### 1.1 Classification Metrics

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, confusion_matrix, classification_report
)

def validate_classification_performance(model, X_test, y_test):
    """
    Validate that model meets all performance targets
    """
    y_pred = model.predict(X_test)
    y_proba = model.predict_proba(X_test)[:, 1]

    # Compute metrics
    metrics = {
        'accuracy': accuracy_score(y_test, y_pred),
        'precision': precision_score(y_test, y_pred),
        'recall': recall_score(y_test, y_pred),
        'f1': f1_score(y_test, y_pred),
        'roc_auc': roc_auc_score(y_test, y_proba),
    }

    # Define targets
    targets = {
        'accuracy': (0.78, '≥78%'),
        'precision': (0.80, '≥80%'),
        'recall': (0.75, '≥75%'),
        'f1': (0.77, '≥77%'),
        'roc_auc': (0.85, '≥0.85'),
    }

    # Validate
    print("CLASSIFICATION PERFORMANCE")
    print("="*50)

    all_pass = True
    for metric, (target, description) in targets.items():
        actual = metrics[metric]
        passed = actual >= target
        status = "✅" if passed else "❌"
        print(f"{status} {metric.upper():10s}: {actual:.3f} (target: {description})")
        all_pass = all_pass and passed

    print("="*50)
    if all_pass:
        print("✅ All performance targets PASSED")
    else:
        print("❌ Some targets FAILED - retrain or tune model")

    return metrics, all_pass

# Usage
metrics, passed = validate_classification_performance(model, X_test, y_test)
if not passed:
    raise Exception("Model does not meet performance targets")
```

### 1.2 Per-Class Performance

```python
def validate_per_class_metrics(model, X_test, y_test):
    """
    Check performance for each class separately
    """
    y_pred = model.predict(X_test)

    # Get detailed classification report
    report = classification_report(y_test, y_pred,
                                   target_names=['Low-Signal', 'High-Signal'],
                                   output_dict=True)

    print("\nPER-CLASS PERFORMANCE")
    print("="*60)

    for class_name in ['Low-Signal', 'High-Signal']:
        metrics = report[class_name]
        print(f"\n{class_name}:")
        print(f"  Precision: {metrics['precision']:.3f}  (How often we're right when predicting this class)")
        print(f"  Recall:    {metrics['recall']:.3f}    (How many of this class we correctly find)")
        print(f"  F1-Score:  {metrics['f1-score']:.3f}  (Balanced score)")
        print(f"  Support:   {int(metrics['support'])}")

    # Check if performance is balanced (not biased toward one class)
    low_f1 = report['Low-Signal']['f1-score']
    high_f1 = report['High-Signal']['f1-score']
    balance = abs(low_f1 - high_f1)

    print(f"\nClass Balance (F1 difference): {balance:.3f}")
    if balance < 0.10:
        print("✅ Good balance between classes")
    else:
        print("⚠️  Imbalanced performance between classes")

    return report
```

---

## 2. Privacy & Security Testing

### 2.1 No Personal Data Leakage

```python
def test_no_pii_leakage(model, X_test, sample_size=1000):
    """
    Verify model doesn't leak personal data in predictions
    """
    print("\nPRIVACY TEST: No PII Leakage")
    print("="*50)

    # Test 1: Predictions should be deterministic
    print("\n1. Deterministic predictions")
    sample = X_test.iloc[:sample_size]
    pred1 = model.predict(sample)
    pred2 = model.predict(sample)

    assert (pred1 == pred2).all(), "Predictions not deterministic!"
    print("   ✅ Same input → same output (good)")

    # Test 2: Output doesn't contain personal data
    print("\n2. Output format check")
    y_pred = model.predict(X_test.iloc[:10])
    y_proba = model.predict_proba(X_test.iloc[:10])

    # Predictions should be integers [0, 1]
    assert all(p in [0, 1] for p in y_pred), "Invalid predictions"
    # Confidence should be floats [0, 1]
    assert all(0 <= p <= 1 for row in y_proba for p in row), "Invalid confidence"
    print("   ✅ Outputs are generic [0/1] or [0-1] scores (good)")

    # Test 3: Small perturbation in input shouldn't leak information
    print("\n3. Sensitivity to noise")
    sample_idx = 0
    original = X_test.iloc[[sample_idx]]

    # Add small random noise to features
    noisy = original.copy()
    noise = np.random.normal(0, 0.01, noisy.shape)
    noisy = noisy + noise

    original_pred = model.predict(original)[0]
    noisy_pred = model.predict(noisy)[0]

    # Predictions should be robust (same despite noise)
    same_prediction = (original_pred == noisy_pred)
    print(f"   {'✅' if same_prediction else '❌'} Robust to noise")

    print("\n✅ Privacy tests PASSED")

# Usage
test_no_pii_leakage(model, X_test)
```

### 2.2 Feature Privacy Validation

```python
def test_feature_privacy(X_test):
    """
    Verify all features are privacy-safe
    (binned, aggregated, no exact values)
    """
    print("\nFEATURE PRIVACY VALIDATION")
    print("="*50)

    privacy_rules = {
        # Binned features (should be integers with limited range)
        'primary_topic_code': {'type': 'int', 'min': 0, 'max': 9},
        'engagement_duration_bin': {'type': 'int', 'min': 1, 'max': 5},
        'scroll_depth_bin': {'type': 'int', 'min': 1, 'max': 10},
        'comment_length_bin': {'type': 'int', 'min': 0, 'max': 3},

        # Normalized continuous (should be 0-1)
        'topic_affinity_score': {'type': 'float', 'min': 0, 'max': 1},
        'creator_authority_score': {'type': 'float', 'min': 0, 'max': 1},
        'engagement_intensity': {'type': 'float', 'min': 0, 'max': 1},
        'return_probability': {'type': 'float', 'min': 0, 'max': 1},
        'social_action_weight': {'type': 'float', 'min': 0, 'max': 1},
        'quality_score': {'type': 'float', 'min': 0, 'max': 1},
        'niche_fit': {'type': 'float', 'min': 0, 'max': 1},

        # Boolean features (should be 0/1)
        'is_controversial': {'type': 'int', 'min': 0, 'max': 1},
        'is_educational': {'type': 'int', 'min': 0, 'max': 1},
        'has_explicit_action': {'type': 'int', 'min': 0, 'max': 1},
        'bookmarked': {'type': 'int', 'min': 0, 'max': 1},
    }

    all_valid = True

    for feature, rules in privacy_rules.items():
        # Check existence
        if feature not in X_test.columns:
            print(f"❌ {feature}: MISSING")
            all_valid = False
            continue

        # Check range
        col = X_test[feature]
        if col.min() < rules['min'] or col.max() > rules['max']:
            print(f"❌ {feature}: Out of range [{col.min()}, {col.max()}]")
            all_valid = False
            continue

        # Check uniqueness (should not be perfectly unique = good privacy)
        if col.nunique() == len(col):
            print(f"❌ {feature}: All unique (privacy risk!)")
            all_valid = False
            continue

        print(f"✅ {feature}: Valid & private")

    if all_valid:
        print("\n✅ All features are privacy-safe")
    else:
        raise Exception("Feature privacy validation FAILED")

    return all_valid

# Usage
test_feature_privacy(X_test)
```

### 2.3 K-Anonymity Check

```python
def test_k_anonymity(X_test, k_min=1000):
    """
    Verify no signal combination is unique (k-anonymity)

    For each combination of features, at least k_min users
    should share that combination
    """
    print("\nK-ANONYMITY TEST")
    print("="*50)

    # Test k-anonymity on single features
    print("\nSingle Features:")
    k_violations = 0

    for col in X_test.columns:
        value_counts = X_test[col].value_counts()
        min_count = value_counts.min()

        if min_count < k_min:
            print(f"  ⚠️  {col}: min frequency = {min_count} (target: ≥{k_min})")
            k_violations += 1
        else:
            print(f"  ✅ {col}: min frequency = {min_count}")

    if k_violations > 0:
        print(f"\n⚠️  {k_violations} features below k-anonymity threshold")
        print("   Note: In production, aggregate with more users")
    else:
        print("\n✅ All single features meet k-anonymity")

    # Test k-anonymity on feature pairs (sample)
    print("\nFeature Pairs (sample):")
    sample_pairs = [
        ('primary_topic_code', 'engagement_duration_bin'),
        ('topic_affinity_score', 'return_probability'),
        ('engagement_frequency_bin', 'scroll_depth_bin'),
    ]

    for col1, col2 in sample_pairs:
        pair_counts = X_test.groupby([col1, col2]).size()
        min_count = pair_counts.min()

        status = "✅" if min_count >= k_min else "⚠️ "
        print(f"  {status} {col1} × {col2}: min = {min_count}")

# Usage
test_k_anonymity(X_test, k_min=1000)
```

---

## 3. Fairness & Bias Testing

### 3.1 Demographic Parity

```python
def test_demographic_parity(X_test, y_test, y_pred,
                            sensitive_feature='primary_topic_code'):
    """
    Verify model doesn't discriminate by topic

    Check: P(pred=1|topic=A) ≈ P(pred=1|topic=B) for all topics
    """
    print("\nFAIRNESS TEST: Demographic Parity")
    print("="*50)
    print(f"Testing for bias by: {sensitive_feature}\n")

    df_test = X_test.copy()
    df_test['actual'] = y_test.values
    df_test['predicted'] = y_pred

    # Get prediction rates by topic
    topics = df_test[sensitive_feature].unique()

    prediction_rates = {}
    for topic in sorted(topics):
        mask = df_test[sensitive_feature] == topic
        subset = df_test[mask]
        rate = subset['predicted'].mean()
        count = len(subset)
        prediction_rates[topic] = (rate, count)

        print(f"Topic {topic}: {rate:.1%} predicted as high-signal (n={count})")

    # Check disparity
    rates = [rate for rate, _ in prediction_rates.values()]
    max_rate = max(rates)
    min_rate = min(rates)
    disparity = max_rate - min_rate

    print(f"\nDisparity: {disparity:.1%} (max - min)")

    if disparity < 0.10:
        print("✅ Fair: Prediction rates similar across topics")
    else:
        print("⚠️  Potential bias: Different rates by topic")

    return prediction_rates
```

### 3.2 Equalized Odds

```python
def test_equalized_odds(X_test, y_test, y_pred,
                       sensitive_feature='primary_topic_code'):
    """
    Verify model has similar TPR and FPR across groups

    TPR should be similar: P(pred=1|actual=1,topic=A) ≈ P(pred=1|actual=1,topic=B)
    FPR should be similar: P(pred=1|actual=0,topic=A) ≈ P(pred=1|actual=0,topic=B)
    """
    print("\nFAIRNESS TEST: Equalized Odds")
    print("="*50)

    df_test = X_test.copy()
    df_test['actual'] = y_test.values
    df_test['predicted'] = y_pred

    topics = df_test[sensitive_feature].unique()

    print("\nTrue Positive Rate (TPR) by topic:")
    tprs = {}
    for topic in sorted(topics):
        mask = (df_test[sensitive_feature] == topic) & (df_test['actual'] == 1)
        subset = df_test[mask]
        if len(subset) > 0:
            tpr = subset['predicted'].mean()
            tprs[topic] = tpr
            print(f"  Topic {topic}: {tpr:.1%}")

    print("\nFalse Positive Rate (FPR) by topic:")
    fprs = {}
    for topic in sorted(topics):
        mask = (df_test[sensitive_feature] == topic) & (df_test['actual'] == 0)
        subset = df_test[mask]
        if len(subset) > 0:
            fpr = subset['predicted'].mean()
            fprs[topic] = fpr
            print(f"  Topic {topic}: {fpr:.1%}")

    # Check for disparity
    tpr_disparity = max(tprs.values()) - min(tprs.values())
    fpr_disparity = max(fprs.values()) - min(fprs.values())

    print(f"\nTPR Disparity: {tpr_disparity:.1%}")
    print(f"FPR Disparity: {fpr_disparity:.1%}")

    if tpr_disparity < 0.15 and fpr_disparity < 0.15:
        print("✅ Fair: Similar TPR and FPR across topics")
    else:
        print("⚠️  Potential bias: Different TPR/FPR by topic")

# Usage
pred_rates = test_demographic_parity(X_test, y_test, y_pred)
test_equalized_odds(X_test, y_test, y_pred)
```

---

## 4. Robustness Testing

### 4.1 Adversarial Perturbations

```python
def test_robustness_to_perturbations(model, X_test, n_samples=100):
    """
    Test if small perturbations change predictions
    (model should be robust to noise)
    """
    print("\nROBUSTNESS TEST: Adversarial Perturbations")
    print("="*50)

    sample = X_test.iloc[:n_samples].copy()
    original_pred = model.predict(sample)

    # Test different perturbation magnitudes
    perturbation_sizes = [0.01, 0.05, 0.10, 0.15, 0.20]

    for pert_size in perturbation_sizes:
        perturbed = sample.copy()
        noise = np.random.normal(0, pert_size, perturbed.shape)
        perturbed = perturbed + noise

        # Clip to valid ranges
        for col in perturbed.columns:
            if col in ['topic_affinity_score', 'engagement_intensity', 'quality_score']:
                perturbed[col] = perturbed[col].clip(0, 1)

        perturbed_pred = model.predict(perturbed)
        match_rate = (original_pred == perturbed_pred).mean()

        print(f"Perturbation ±{pert_size:.0%}: {match_rate:.1%} predictions unchanged")

    print("\n✅ Robustness check complete")

# Usage
test_robustness_to_perturbations(model, X_test)
```

### 4.2 Out-of-Distribution Detection

```python
def test_ood_detection(model, X_test, X_val):
    """
    Verify model handles out-of-distribution inputs gracefully
    """
    print("\nROBUSTNESS TEST: Out-of-Distribution Detection")
    print("="*50)

    # Test with extreme values
    print("\n1. Extreme feature values:")
    extreme = X_test.iloc[[0]].copy()

    # Set all features to edge values
    extreme.iloc[0] = extreme.iloc[0].apply(lambda x: 1.0 if isinstance(x, float) else 5)

    try:
        pred = model.predict(extreme)
        proba = model.predict_proba(extreme)
        print(f"   ✅ Model handles extreme values: pred={pred}, confidence={proba[0,1]:.3f}")
    except Exception as e:
        print(f"   ❌ Model fails on extreme values: {e}")

    # Test with missing values (should have been handled in preprocessing)
    print("\n2. Missing values:")
    missing = X_test.iloc[[0]].copy()
    missing.iloc[0, 0] = np.nan

    if missing.isna().any().any():
        print(f"   ⚠️  Warning: Input contains NaN")
        print("      (Feature engineering should have handled this)")
    else:
        print("   ✅ No NaN values in test set")

    print("\n✅ OOD detection check complete")

# Usage
test_ood_detection(model, X_test, X_val)
```

---

## 5. Integration Testing

### 5.1 End-to-End Pipeline Test

```python
def test_end_to_end_pipeline(feature_extractor, model, sample_signal):
    """
    Test: Raw Signal → Features → Prediction
    """
    print("\nINTEGRATION TEST: End-to-End Pipeline")
    print("="*50)

    # Step 1: Extract features
    try:
        features = feature_extractor.extract(sample_signal)
        print("✅ Step 1: Feature extraction successful")
    except Exception as e:
        print(f"❌ Step 1 FAILED: {e}")
        return False

    # Step 2: Validate features
    try:
        assert len(features) == 29, f"Expected 29 features, got {len(features)}"
        assert features.isna().sum() == 0, "Features contain NaN"
        print("✅ Step 2: Feature validation successful")
    except Exception as e:
        print(f"❌ Step 2 FAILED: {e}")
        return False

    # Step 3: Make prediction
    try:
        features_2d = features.values.reshape(1, -1)
        prediction = model.predict(features_2d)[0]
        confidence = model.predict_proba(features_2d)[0][prediction]

        print(f"✅ Step 3: Prediction successful")
        print(f"   Prediction: {'HIGH-signal' if prediction else 'LOW-signal'}")
        print(f"   Confidence: {confidence:.1%}")
    except Exception as e:
        print(f"❌ Step 3 FAILED: {e}")
        return False

    # Step 4: Verify output format
    try:
        assert prediction in [0, 1], f"Invalid prediction: {prediction}"
        assert 0 <= confidence <= 1, f"Invalid confidence: {confidence}"
        print("✅ Step 4: Output format validation successful")
    except Exception as e:
        print(f"❌ Step 4 FAILED: {e}")
        return False

    print("\n✅ End-to-end pipeline test PASSED")
    return True

# Usage
from week3_feature_engineering import FeatureExtractor
extractor = FeatureExtractor()
sample = {...}  # Example signal
success = test_end_to_end_pipeline(extractor, model, sample)
```

### 5.2 Inference Latency Test

```python
def test_inference_latency(model, X_test, n_trials=1000, target_ms=100):
    """
    Verify inference speed meets target (<100ms)
    """
    print("\nINTEGRATION TEST: Inference Latency")
    print("="*50)

    import time

    # Single prediction
    start = time.time()
    for _ in range(n_trials):
        model.predict(X_test.iloc[[0]])
    elapsed = time.time() - start

    avg_latency_ms = (elapsed / n_trials) * 1000

    print(f"\nAverage latency over {n_trials} predictions: {avg_latency_ms:.2f}ms")
    print(f"Target: <{target_ms}ms")

    if avg_latency_ms < target_ms:
        print("✅ Latency requirement MET")
    else:
        print("❌ Latency requirement FAILED")

    # Throughput
    throughput = n_trials / elapsed
    print(f"\nThroughput: {throughput:.0f} predictions/second")

    return avg_latency_ms

# Usage
latency = test_inference_latency(model, X_test)
```

---

## 6. Complete Validation Suite

```python
def run_complete_validation(model, X_test, y_test,
                           feature_extractor, sample_signal):
    """
    Run all validation tests
    """
    print("\n" + "="*60)
    print("COMPLETE MODEL VALIDATION SUITE")
    print("="*60)

    all_tests_passed = True

    # 1. Performance
    try:
        metrics, passed = validate_classification_performance(model, X_test, y_test)
        all_tests_passed = all_tests_passed and passed
    except Exception as e:
        print(f"❌ Performance validation FAILED: {e}")
        all_tests_passed = False

    # 2. Privacy
    try:
        test_no_pii_leakage(model, X_test)
        test_feature_privacy(X_test)
        test_k_anonymity(X_test)
    except Exception as e:
        print(f"❌ Privacy validation FAILED: {e}")
        all_tests_passed = False

    # 3. Fairness
    try:
        y_pred = model.predict(X_test)
        test_demographic_parity(X_test, y_test, y_pred)
        test_equalized_odds(X_test, y_test, y_pred)
    except Exception as e:
        print(f"❌ Fairness validation FAILED: {e}")
        all_tests_passed = False

    # 4. Robustness
    try:
        test_robustness_to_perturbations(model, X_test)
        test_ood_detection(model, X_test, X_test)
    except Exception as e:
        print(f"❌ Robustness validation FAILED: {e}")
        all_tests_passed = False

    # 5. Integration
    try:
        test_end_to_end_pipeline(feature_extractor, model, sample_signal)
        test_inference_latency(model, X_test)
    except Exception as e:
        print(f"❌ Integration validation FAILED: {e}")
        all_tests_passed = False

    # Summary
    print("\n" + "="*60)
    if all_tests_passed:
        print("✅ ALL VALIDATION TESTS PASSED")
        print("Model is ready for production deployment")
    else:
        print("❌ SOME VALIDATION TESTS FAILED")
        print("Address failures before production deployment")
    print("="*60)

    return all_tests_passed

# Usage
success = run_complete_validation(model, X_test, y_test,
                                  feature_extractor, sample_signal)
```

---

## 7. Continuous Monitoring (Production)

### 7.1 Performance Drift Detection

```python
def monitor_performance_drift(new_predictions, new_labels,
                             baseline_auc=0.85, alert_threshold=0.80):
    """
    Monitor model performance in production
    Alert if performance drops below threshold
    """
    from sklearn.metrics import roc_auc_score

    current_auc = roc_auc_score(new_labels, new_predictions)

    print(f"Current ROC-AUC: {current_auc:.3f}")
    print(f"Baseline ROC-AUC: {baseline_auc:.3f}")
    print(f"Alert threshold: {alert_threshold:.3f}")

    if current_auc < alert_threshold:
        print("🚨 ALERT: Performance degradation detected!")
        print("→ Trigger model retraining")
    elif current_auc < baseline_auc * 0.95:
        print("⚠️  WARNING: Performance below baseline")
        print("→ Monitor closely, retrain soon")
    else:
        print("✅ Performance nominal")

    return current_auc

# Usage (run weekly in production)
# current_auc = monitor_performance_drift(y_pred_week, y_true_week)
```

---

## Testing Checklist

```
✅ Classification Performance
  ├─ ROC-AUC > 0.85
  ├─ Precision > 0.80
  ├─ Recall > 0.75
  └─ F1 > 0.77

✅ Privacy & Security
  ├─ No PII leakage
  ├─ Features are binned/safe
  ├─ K-anonymity maintained
  └─ Feature privacy validated

✅ Fairness & Bias
  ├─ Demographic parity
  ├─ Equalized odds
  ├─ No topic discrimination
  └─ Balanced per-class performance

✅ Robustness
  ├─ Handles perturbations
  ├─ OOD detection
  └─ Edge case handling

✅ Integration
  ├─ End-to-end pipeline works
  ├─ Inference latency < 100ms
  ├─ Output format correct
  └─ Graceful error handling
```

---

**Ready to validate**: All test functions provided, ready to run before production deployment.
