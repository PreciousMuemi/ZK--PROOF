# Model Training Guide
## End-to-End Training Pipeline for Signal Quality Classification

**Objective**: Train lightweight XGBoost model to classify signals as high/low quality

**Training Time**: ~15-30 minutes on CPU (100K+ signals)

**Data Required**: 100,000+ labeled signal bundles from Week 2

---

## 1. Data Preparation

### 1.1 Load Training Data

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

def load_training_data(csv_path: str) -> tuple:
    """
    Load labeled signal data from CSV
    
    CSV format:
    signal_id,primary_topic,duration_bin,...,label
    sig_001,Technology,4,...,1
    sig_002,Science,2,...,0
    ...
    
    Returns: (X, y) feature matrix and labels
    """
    df = pd.read_csv(csv_path)
    
    # Separate features from label
    X = df.drop(['signal_id', 'label'], axis=1)
    y = df['label']
    
    print(f"Loaded {len(df)} signals")
    print(f"Class distribution:")
    print(f"  High-signal (1): {(y == 1).sum()} ({(y == 1).sum() / len(y) * 100:.1f}%)")
    print(f"  Low-signal (0): {(y == 0).sum()} ({(y == 0).sum() / len(y) * 100:.1f}%)")
    
    return X, y

# Usage
X, y = load_training_data('signals_labeled.csv')
```

### 1.2 Train/Val/Test Split

```python
def split_data(X: pd.DataFrame, y: pd.Series, 
               train_ratio=0.7, val_ratio=0.15) -> tuple:
    """
    Split data into train/val/test sets
    Maintain class distribution (stratified)
    
    Returns:
        X_train, X_val, X_test, y_train, y_val, y_test
    """
    # First split: train vs. temp (val + test)
    X_train, X_temp, y_train, y_temp = train_test_split(
        X, y,
        test_size=1 - train_ratio,          # 30% for val + test
        random_state=42,
        stratify=y                          # Maintain class distribution
    )
    
    # Second split: val vs. test
    test_ratio = 0.5  # Split temp 50/50 into val and test
    X_val, X_test, y_val, y_test = train_test_split(
        X_temp, y_temp,
        test_size=test_ratio,               # 50% of temp
        random_state=42,
        stratify=y_temp                     # Maintain class distribution
    )
    
    print(f"Train set: {len(X_train)} samples ({len(X_train)/len(X)*100:.1f}%)")
    print(f"Val set: {len(X_val)} samples ({len(X_val)/len(X)*100:.1f}%)")
    print(f"Test set: {len(X_test)} samples ({len(X_test)/len(X)*100:.1f}%)")
    
    return X_train, X_val, X_test, y_train, y_val, y_test

# Usage
X_train, X_val, X_test, y_train, y_val, y_test = split_data(X, y)
```

### 1.3 Handle Class Imbalance

```python
def compute_class_weights(y_train: pd.Series) -> dict:
    """
    Compute weights to handle class imbalance
    (60% low-signal, 40% high-signal is typical)
    
    Returns: dict with weights for each class
    """
    from sklearn.utils.class_weight import compute_class_weight
    
    class_weights = compute_class_weight(
        'balanced',
        classes=np.unique(y_train),
        y=y_train
    )
    
    weight_dict = {
        0: class_weights[0],  # Low-signal weight
        1: class_weights[1],  # High-signal weight
    }
    
    print(f"Class weights:")
    print(f"  Low-signal (0): {weight_dict[0]:.2f}")
    print(f"  High-signal (1): {weight_dict[1]:.2f}")
    
    return weight_dict

# Usage
weights = compute_class_weights(y_train)
scale_pos_weight = weights[1] / weights[0]  # For XGBoost
```

### 1.4 Feature Normalization

```python
def normalize_features(X_train: pd.DataFrame, 
                       X_val: pd.DataFrame, 
                       X_test: pd.DataFrame) -> tuple:
    """
    Normalize continuous features using training set statistics
    
    Important: Fit scaler ONLY on training data,
    then apply to val/test (prevents data leakage)
    """
    continuous_cols = [
        'topic_affinity_score', 'creator_authority_score',
        'engagement_intensity', 'return_probability',
        'social_action_weight', 'completion_rate',
        'quality_score', 'niche_fit', 'consistency_score',
        'predicted_helpfulness', 'topic_diversity', 'content_type_match'
    ]
    
    # Fit scaler ONLY on training data
    scaler = StandardScaler()
    X_train_norm = X_train.copy()
    X_train_norm[continuous_cols] = scaler.fit_transform(
        X_train[continuous_cols]
    )
    
    # Apply to val/test using training statistics
    X_val_norm = X_val.copy()
    X_val_norm[continuous_cols] = scaler.transform(
        X_val[continuous_cols]
    )
    
    X_test_norm = X_test.copy()
    X_test_norm[continuous_cols] = scaler.transform(
        X_test[continuous_cols]
    )
    
    print(f"Normalized {len(continuous_cols)} continuous features")
    
    return X_train_norm, X_val_norm, X_test_norm, scaler

# Usage
X_train_norm, X_val_norm, X_test_norm, scaler = normalize_features(
    X_train, X_val, X_test
)
```

---

## 2. Model Training

### 2.1 Train XGBoost Model

```python
from xgboost import XGBClassifier
from sklearn.metrics import roc_auc_score, precision_score, recall_score, f1_score

def train_model(X_train: pd.DataFrame, y_train: pd.Series,
                X_val: pd.DataFrame, y_val: pd.Series,
                scale_pos_weight: float = 1.0) -> XGBClassifier:
    """
    Train XGBoost classifier
    
    Hyperparameters tuned for interpretability + speed:
    - Shallow trees (max_depth=5) → interpretable
    - Moderate learning rate (0.1) → stable
    - Class weights → handle imbalance
    """
    
    print("Training XGBoost model...")
    
    model = XGBClassifier(
        # Tree structure
        n_estimators=100,                   # Number of boosting rounds
        max_depth=5,                        # Shallow = interpretable
        min_child_weight=1,                 # Min samples per leaf
        
        # Learning
        learning_rate=0.1,                  # Conservative step size
        subsample=0.8,                      # Random 80% of samples
        colsample_bytree=0.8,               # Random 80% of features
        colsample_bylevel=0.8,              # Column sampling
        
        # Regularization
        reg_alpha=0.1,                      # L1 regularization
        reg_lambda=1.0,                     # L2 regularization
        
        # Class balance
        scale_pos_weight=scale_pos_weight,  # Weight for positive class
        
        # Training
        random_state=42,
        n_jobs=-1,                          # Use all CPU cores
        verbosity=1,                        # Print progress
    )
    
    # Train with early stopping on validation set
    eval_set = [(X_val, y_val)]
    eval_metric = ['logloss', 'auc']
    
    model.fit(
        X_train, y_train,
        eval_set=eval_set,
        eval_metric=eval_metric,
        early_stopping_rounds=20,           # Stop if no improvement
        verbose=True
    )
    
    print("✅ Training complete")
    return model

# Usage
model = train_model(X_train_norm, y_train, X_val_norm, y_val, 
                   scale_pos_weight=1.5)
```

### 2.2 Hyperparameter Tuning

```python
from sklearn.model_selection import GridSearchCV

def tune_hyperparameters(X_train: pd.DataFrame, y_train: pd.Series):
    """
    Grid search for best hyperparameters
    
    Warning: Takes 1-2 hours on large datasets
    """
    
    param_grid = {
        'n_estimators': [50, 100, 150],
        'max_depth': [3, 5, 7],
        'learning_rate': [0.01, 0.05, 0.1],
        'subsample': [0.6, 0.8, 1.0],
        'colsample_bytree': [0.6, 0.8, 1.0],
    }
    
    print("Running grid search (this may take a while)...")
    
    grid_search = GridSearchCV(
        XGBClassifier(random_state=42, n_jobs=-1),
        param_grid,
        scoring='roc_auc',
        cv=5,                               # 5-fold cross-validation
        n_jobs=-1,                          # Parallel
        verbose=1
    )
    
    grid_search.fit(X_train, y_train)
    
    print(f"\nBest parameters: {grid_search.best_params_}")
    print(f"Best ROC-AUC: {grid_search.best_score_:.3f}")
    
    return grid_search.best_estimator_

# Optional: Use if default params don't achieve 0.85+ ROC-AUC
# best_model = tune_hyperparameters(X_train_norm, y_train)
```

### 2.3 Monitor Training Progress

```python
import matplotlib.pyplot as plt

def plot_training_progress(model: XGBClassifier):
    """
    Visualize training progress
    """
    results = model.evals_result()
    
    fig, axes = plt.subplots(1, 2, figsize=(12, 4))
    
    # Loss over epochs
    axes[0].plot(results['validation_0']['logloss'], label='Validation Loss')
    axes[0].set_xlabel('Epoch')
    axes[0].set_ylabel('Loss')
    axes[0].set_title('Training Progress: Loss')
    axes[0].legend()
    axes[0].grid()
    
    # AUC over epochs
    axes[1].plot(results['validation_0']['auc'], label='Validation AUC')
    axes[1].set_xlabel('Epoch')
    axes[1].set_ylabel('AUC')
    axes[1].set_title('Training Progress: AUC')
    axes[1].legend()
    axes[1].grid()
    
    plt.tight_layout()
    plt.savefig('training_progress.png')
    print("✅ Saved training_progress.png")

# Usage
plot_training_progress(model)
```

---

## 3. Model Evaluation

### 3.1 Validation Metrics

```python
def evaluate_model(model: XGBClassifier, 
                   X_val: pd.DataFrame, y_val: pd.Series,
                   dataset_name='Validation') -> dict:
    """
    Compute validation metrics
    """
    y_pred = model.predict(X_val)
    y_proba = model.predict_proba(X_val)[:, 1]
    
    metrics = {
        'accuracy': accuracy_score(y_val, y_pred),
        'precision': precision_score(y_val, y_pred),
        'recall': recall_score(y_val, y_pred),
        'f1': f1_score(y_val, y_pred),
        'roc_auc': roc_auc_score(y_val, y_proba),
    }
    
    print(f"\n{dataset_name} Metrics:")
    print(f"  Accuracy:  {metrics['accuracy']:.3f}")
    print(f"  Precision: {metrics['precision']:.3f}")
    print(f"  Recall:    {metrics['recall']:.3f}")
    print(f"  F1-Score:  {metrics['f1']:.3f}")
    print(f"  ROC-AUC:   {metrics['roc_auc']:.3f}")
    
    return metrics

# Usage
val_metrics = evaluate_model(model, X_val_norm, y_val, 'Validation')

# Check if meets targets
if val_metrics['roc_auc'] >= 0.85 and val_metrics['precision'] >= 0.80:
    print("✅ Model meets performance targets!")
else:
    print("⚠️  Model below targets, consider retraining")
```

### 3.2 Confusion Matrix

```python
from sklearn.metrics import confusion_matrix
import seaborn as sns

def plot_confusion_matrix(model: XGBClassifier,
                         X_val: pd.DataFrame, y_val: pd.Series):
    """
    Visualize confusion matrix
    """
    y_pred = model.predict(X_val)
    cm = confusion_matrix(y_val, y_pred)
    
    plt.figure(figsize=(8, 6))
    sns.heatmap(cm, annot=True, fmt='d', cmap='Blues',
                xticklabels=['Low-Signal', 'High-Signal'],
                yticklabels=['Low-Signal', 'High-Signal'])
    plt.xlabel('Predicted')
    plt.ylabel('Actual')
    plt.title('Confusion Matrix')
    plt.tight_layout()
    plt.savefig('confusion_matrix.png')
    print("✅ Saved confusion_matrix.png")
    
    # Print detailed breakdown
    tn, fp, fn, tp = cm.ravel()
    print(f"\nConfusion Matrix Breakdown:")
    print(f"  True Negatives:  {tn}")
    print(f"  False Positives: {fp}")
    print(f"  False Negatives: {fn}")
    print(f"  True Positives:  {tp}")

# Usage
plot_confusion_matrix(model, X_val_norm, y_val)
```

### 3.3 ROC Curve

```python
from sklearn.metrics import roc_curve, auc

def plot_roc_curve(model: XGBClassifier,
                   X_val: pd.DataFrame, y_val: pd.Series):
    """
    Visualize ROC curve
    """
    y_proba = model.predict_proba(X_val)[:, 1]
    fpr, tpr, _ = roc_curve(y_val, y_proba)
    roc_auc = auc(fpr, tpr)
    
    plt.figure(figsize=(8, 6))
    plt.plot(fpr, tpr, label=f'ROC Curve (AUC = {roc_auc:.3f})')
    plt.plot([0, 1], [0, 1], 'k--', label='Random')
    plt.xlabel('False Positive Rate')
    plt.ylabel('True Positive Rate')
    plt.title('ROC Curve')
    plt.legend()
    plt.grid()
    plt.tight_layout()
    plt.savefig('roc_curve.png')
    print("✅ Saved roc_curve.png")

# Usage
plot_roc_curve(model, X_val_norm, y_val)
```

### 3.4 Feature Importance

```python
def plot_feature_importance(model: XGBClassifier, top_n=15):
    """
    Show which features matter most
    """
    importance_df = pd.DataFrame({
        'feature': model.get_booster().feature_names,
        'importance': model.feature_importances_
    }).sort_values('importance', ascending=False)
    
    plt.figure(figsize=(10, 6))
    plt.barh(importance_df['feature'][:top_n], 
             importance_df['importance'][:top_n])
    plt.xlabel('Feature Importance')
    plt.title(f'Top {top_n} Most Important Features')
    plt.gca().invert_yaxis()
    plt.tight_layout()
    plt.savefig('feature_importance.png')
    print("✅ Saved feature_importance.png")
    
    print(f"\nTop 10 Features:")
    for idx, row in importance_df.head(10).iterrows():
        print(f"  {row['feature']}: {row['importance']:.3f}")

# Usage
plot_feature_importance(model, top_n=15)
```

---

## 4. Test Set Evaluation

### 4.1 Final Performance on Test Set

```python
def evaluate_test_set(model: XGBClassifier,
                     X_test: pd.DataFrame, y_test: pd.Series) -> dict:
    """
    Final evaluation on held-out test set
    
    This is the TRUE performance indicator
    (Val set was used for early stopping, so don't trust it alone)
    """
    y_pred = model.predict(X_test)
    y_proba = model.predict_proba(X_test)[:, 1]
    
    from sklearn.metrics import (
        accuracy_score, precision_score, recall_score, f1_score,
        roc_auc_score, confusion_matrix
    )
    
    # Compute all metrics
    metrics = {
        'accuracy': accuracy_score(y_test, y_pred),
        'precision': precision_score(y_test, y_pred),
        'recall': recall_score(y_test, y_pred),
        'f1': f1_score(y_test, y_pred),
        'roc_auc': roc_auc_score(y_test, y_proba),
    }
    
    # Confusion matrix
    cm = confusion_matrix(y_test, y_pred)
    tn, fp, fn, tp = cm.ravel()
    
    metrics.update({
        'true_negatives': tn,
        'false_positives': fp,
        'false_negatives': fn,
        'true_positives': tp,
        'specificity': tn / (tn + fp),  # True negative rate
        'sensitivity': tp / (tp + fn),  # True positive rate (recall)
    })
    
    # Print results
    print("\n" + "="*50)
    print("TEST SET RESULTS")
    print("="*50)
    print(f"Accuracy:    {metrics['accuracy']:.3f}")
    print(f"Precision:   {metrics['precision']:.3f}  ← When we say 'high-signal', we're correct {metrics['precision']:.0%}")
    print(f"Recall:      {metrics['recall']:.3f}   ← We catch {metrics['recall']:.0%} of actual high-signals")
    print(f"F1-Score:    {metrics['f1']:.3f}")
    print(f"ROC-AUC:     {metrics['roc_auc']:.3f}   ← Ability to rank high vs low signals")
    print(f"\nConfusion Matrix:")
    print(f"  TN: {tn}  FP: {fp}")
    print(f"  FN: {fn}  TP: {tp}")
    print(f"\nSpecificity: {metrics['specificity']:.3f}  ← Correctly identify low-signals")
    print(f"Sensitivity: {metrics['sensitivity']:.3f}  ← Correctly identify high-signals")
    
    # Check targets
    print("\n" + "="*50)
    print("PERFORMANCE TARGETS")
    print("="*50)
    targets = {
        'ROC-AUC > 0.85': metrics['roc_auc'] > 0.85,
        'Precision > 0.80': metrics['precision'] > 0.80,
        'Recall > 0.75': metrics['recall'] > 0.75,
        'F1 > 0.77': metrics['f1'] > 0.77,
    }
    
    for target, achieved in targets.items():
        status = "✅" if achieved else "❌"
        print(f"{status} {target}")
    
    if all(targets.values()):
        print("\n🎉 All targets achieved!")
    else:
        print("\n⚠️  Some targets not met, consider retraining")
    
    return metrics

# Usage
test_metrics = evaluate_test_set(model, X_test_norm, y_test)
```

---

## 5. Save & Export Model

### 5.1 Save for Production

```python
import joblib
import json
from datetime import datetime

def save_model(model: XGBClassifier, scaler, feature_names: list,
               output_dir='models/'):
    """
    Save trained model and metadata for production
    """
    import os
    os.makedirs(output_dir, exist_ok=True)
    
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    
    # Save model
    model_path = f"{output_dir}/signal_quality_model_{timestamp}.pkl"
    joblib.dump(model, model_path)
    print(f"✅ Model saved: {model_path}")
    
    # Save scaler
    scaler_path = f"{output_dir}/scaler_{timestamp}.pkl"
    joblib.dump(scaler, scaler_path)
    print(f"✅ Scaler saved: {scaler_path}")
    
    # Save metadata
    metadata = {
        'timestamp': timestamp,
        'model_type': 'XGBClassifier',
        'feature_names': feature_names,
        'n_features': len(feature_names),
        'classes': [0, 1],
        'class_names': ['low_signal', 'high_signal'],
        'feature_importance': dict(zip(
            feature_names,
            model.feature_importances_.tolist()
        ))
    }
    
    metadata_path = f"{output_dir}/metadata_{timestamp}.json"
    with open(metadata_path, 'w') as f:
        json.dump(metadata, f, indent=2)
    print(f"✅ Metadata saved: {metadata_path}")
    
    return model_path, scaler_path, metadata_path

# Usage
model_path, scaler_path, metadata_path = save_model(
    model, scaler, X_train_norm.columns.tolist()
)
```

### 5.2 Load Model for Inference

```python
def load_model_for_inference(model_path: str, scaler_path: str):
    """
    Load trained model for making predictions
    """
    model = joblib.load(model_path)
    scaler = joblib.load(scaler_path)
    
    print(f"✅ Loaded model from {model_path}")
    print(f"✅ Loaded scaler from {scaler_path}")
    
    return model, scaler

# Usage
model, scaler = load_model_for_inference(model_path, scaler_path)
```

---

## 6. Complete Training Script

```python
#!/usr/bin/env python3
"""
Complete training pipeline for signal quality classifier
"""

def main():
    # 1. Load and prepare data
    print("Step 1: Loading data...")
    X, y = load_training_data('signals_labeled.csv')
    
    print("\nStep 2: Splitting data...")
    X_train, X_val, X_test, y_train, y_val, y_test = split_data(X, y)
    
    print("\nStep 3: Computing class weights...")
    weights = compute_class_weights(y_train)
    scale_pos_weight = weights[1] / weights[0]
    
    print("\nStep 4: Normalizing features...")
    X_train_norm, X_val_norm, X_test_norm, scaler = normalize_features(
        X_train, X_val, X_test
    )
    
    # 2. Train model
    print("\nStep 5: Training model...")
    model = train_model(X_train_norm, y_train, X_val_norm, y_val,
                       scale_pos_weight=scale_pos_weight)
    
    # 3. Evaluate
    print("\nStep 6: Evaluating on validation set...")
    val_metrics = evaluate_model(model, X_val_norm, y_val)
    
    print("\nStep 7: Visualizing results...")
    plot_training_progress(model)
    plot_confusion_matrix(model, X_val_norm, y_val)
    plot_roc_curve(model, X_val_norm, y_val)
    plot_feature_importance(model)
    
    print("\nStep 8: Final test set evaluation...")
    test_metrics = evaluate_test_set(model, X_test_norm, y_test)
    
    # 4. Save
    print("\nStep 9: Saving model...")
    model_path, scaler_path, metadata_path = save_model(
        model, scaler, X_train_norm.columns.tolist()
    )
    
    print("\n" + "="*50)
    print("✅ TRAINING COMPLETE")
    print("="*50)
    print(f"Model path: {model_path}")
    print(f"Scaler path: {scaler_path}")
    print(f"Metadata path: {metadata_path}")

if __name__ == '__main__':
    main()
```

---

## 7. Troubleshooting

### Model ROC-AUC < 0.85?

```
Try:
1. Increase training data (more labeled signals)
2. Add more features from Week 2 signals
3. Tune hyperparameters (GridSearchCV)
4. Check for data quality issues
5. Try ensemble (combine XGBoost + neural network)
```

### Overfitting (Val AUC > Test AUC by >0.05)?

```
Try:
1. Increase regularization (reg_alpha, reg_lambda)
2. Decrease max_depth (more shallow trees)
3. Increase early_stopping_rounds
4. Add more training data
5. Feature selection (remove weak features)
```

### Class Imbalance Issues?

```
Try:
1. Adjust scale_pos_weight (increase if recall too low)
2. Use SMOTE oversampling
3. Adjust decision threshold (0.3 instead of 0.5)
4. Use different evaluation metric (F1 instead of accuracy)
```

---

## Next Steps

1. ✅ **Training complete** - Model trained and evaluated
2. → **Validation** - Test on realistic signals (VALIDATION_GUIDE.md)
3. → **Integration** - Wire into relevance engine (INTEGRATION_GUIDE.md)
4. → **Monitoring** - Track performance in production

