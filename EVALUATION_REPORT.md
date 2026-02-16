# Evaluation Report

## Summary

| Modality | Metric | Value |
|----------|--------|-------|
| Sensor (Failure Classifier) | AUC (mean) | 0.9982 |
| Sensor (Failure Classifier) | AP (mean) | 0.9651 |
| Sensor (TTB Regressor) | MAE (mean) | 270.34 hours (~11.3 days) |
| Image (Rust Detection) | Rust recall (manual) | 100% (5/5) |
| Image (Rust Detection) | Clean false-positive rate (manual) | 40% (2/5) |
| Environmental | Validated scenarios | ✅ 3 scenarios |
| System | P50 latency | ~88 ms @ ~100 req/s |

---

## 1. Model Performance

### 1.1 Sensor Modality (LightGBM)

**Dataset / training setup**
- Rows used (classification): **220,304**
- Rows used (regression subset): **166,425**
- Window size: **30**
- Engineered features: **306**
- Positive rate (failure label): **6.57%** (imbalanced)

#### Failure Classifier (LightGBM)
| Metric | Value |
|--------|-------|
| AUC (mean across folds) | **0.9982** |
| Average Precision (mean) | **0.9651** |
| Folds used | **4** |
| Folds skipped | **1** |

**Why AUC/AP?** With a ~6.6% positive rate, AUC/AP better reflect real discrimination than raw accuracy.

#### Time-to-Breakdown Regressor (LightGBM)
| Metric | Value |
|--------|-------|
| MAE (mean across folds) | **270.34 hours** |
| Folds used | **5** |
| Folds skipped | **0** |

**Interpretation:** TTB labels are noisy/coarse relative to short windows, so absolute MAE is high. This is expected for an initial baseline and is the primary improvement target.

---

### 1.2 Image Modality (Rust Detection, ONNX)

**Models Evaluated**

We trained and tested two image models for rust detection:

**1. MobileNetV3 (Recommended - Default)**
- Architecture: Lightweight CNN pre-trained on ImageNet
- Trained in Colab on Roboflow "rust detection" dataset (binary: `no_rust`, `rust`)
- Exported to: `artifacts/rust_model.onnx` (6 MB)
- Inference speed: ~19-55ms (avg 42ms)
- **Test results (10 images):** 100% accuracy (10/10)
  - Rust images: 5/5 correct (100%)
  - Clean images: 5/5 correct (100%)
- No false positives on clean surfaces

**2. CLIP (Vision-Language Model - Experimental)**
- Architecture: OpenAI CLIP (400M image-text pairs)
- Exported to: `artifacts/rust_clip.onnx` (335 MB, 56x larger)
- Inference speed: ~64-127ms (3-6x slower)
- **Test results (10 images):** 70% accuracy (7/10)
  - Rust images: 5/5 correct (100%)
  - Clean images: 2/5 correct (40%) - **severe false positive rate**
- Not recommended for production use (see [VLM_EVALUATION.md](docs/VLM_EVALUATION.md))

**Current Deployment**
- **Default model:** MobileNetV3 (switchable via `RUST_MODEL_TYPE` env variable)
- Labels: `artifacts/rust_labels.json` = `["no_rust", "rust"]`

**Key design choice (Fusion threshold):**
- We *only fuse* rust into the final fault type when **label = rust AND rust_prob ≥ threshold**.
- This prevents moderate-confidence false positives on clean images from forcing `corrosion_rust`.

> Your current clean false positives are **below the fusion threshold**, so they should not set `predicted_fault_type = corrosion_rust`. They can still appear as an image signal, but do not override fault type.

---

### 1.3 Environmental Modality (Rule-based)

Validated using 3 synthetic scenarios:
| Scenario | Expected multiplier | Observed |
|----------|---------------------|----------|
| Favorable | 1.0x | ✅ |
| Normal | 1.1x | ✅ |
| Critical | 2.0x | ✅ |

Environmental logic is interpretable and meant as a first version until real operational metadata is available.

---

## 2. Multimodal Behavior & Fusion

### Fusion policy (high level)
- **Sensor** produces: `p_fail`, `ttb`, sensor-based signals
- **Image** produces: `label`, `prob`, and an `img:*` signal
- **Fusion applies only if rust confidence is high** (thresholded):
  - `p_fail = clamp(p_fail + bump * rust_prob)`
  - `predicted_fault_type = "corrosion_rust"`
  - `fault_confidence = max(fault_confidence, rust_prob)`

This keeps the system robust in the presence of imperfect image predictions.

---

## 3. Error Analysis

### Sensor
- **Imbalance:** 6.57% positives → AUC/AP are the right primary metrics.
- **TTB noise:** MAE is large (~11 days). Improvement options:
  - better time-based aggregation (longer context windows, frequency-domain features)
  - `log(TTB)` regression + back-transform
  - filtering/denoising labels, or predicting *risk horizon buckets* instead of exact hours

### Image
- **MobileNetV3 (current):** 100% accuracy on 10-image test set, no false positives
- **CLIP (experimental):** 70% accuracy, severe false positives on clean surfaces (60% FP rate)
- **Decision:** Using MobileNetV3 as default for production reliability
- **Mitigation:** Fusion threshold prevents incorrect fault override in case of false positives
- Next improvements:
  - collect pump-specific images (domain gap reduction)
  - test other architectures (EfficientNet, ResNet)
  - augmentations focusing on lighting/metal textures

### Fusion
- We clamp to [0,1] and do not estimate uncertainty intervals.
- Environmental multiplier boosts risk but does not fully override extremely low sensor p_fail (by design).

---

## 4. Load Testing

Measured at 10/50/200 request levels with concurrency=10.

| Metric | Value |
|--------|-------|
| Throughput | ~100 req/s |
| P50 latency | ~88 ms |
| P95 latency | ~147 ms |
| Error rate | 0% |

**Primary runtime bottleneck:** ONNX image inference (dominant share of request time when image is provided).

---

## 5. Automated Test Suite

The system includes 22 automated pytest tests organized into 5 categories:

| Test Class | Tests | Coverage |
|------------|-------|----------|
| `TestHealthEndpoint` | 3 | Health check, model version, status validation |
| `TestPredictEndpoint` | 8 | Single predictions, all modality combinations |
| `TestInputValidation` | 5 | Schema validation, error handling, edge cases |
| `TestBatchEndpoint` | 4 | Batch predictions, mixed inputs, empty batches |
| `TestFeatureExtraction` | 2 | Sensor feature handling, missing values |

**Run tests:**
```powershell
pytest tests/api/test_api.py -v
```

---

## 6. Fault Type Detection

| Fault Type | Detection Trigger | Source Modality |
|------------|-------------------|-----------------|
| `bearing_failure` | High vibration + high std deviation (p≥0.7, std>50) | Sensor |
| `seal_leak` | Moderate failure + short TTB (p≥0.5, TTB≤72h) | Sensor |
| `corrosion_rust` | Rust detected with >85% confidence | Image |
| `environmental_stress` | Environmental risk multiplier ≥1.5 | Environmental |
| `null` | No fault conditions met | - |

---

## 7. Datasets

| Dataset | Source | Size | License |
|---------|--------|------|---------|
| Sensor Data | [Kaggle Pump Sensor Data](https://www.kaggle.com/datasets/nphantawee/pump-sensor-data) | ~220k samples | CC0 Public Domain |
| Image Training | [Roboflow Rust Detection](https://universe.roboflow.com/test-stage/rust-detection-t8vza/dataset/8) | ~2k images | CC BY 4.0 |
| Image Testing | Manual photos | 10 images | Internal |
| Environmental | Synthetic | 3 scenarios | N/A |

See [data/DATASET_MANIFEST.md](data/DATASET_MANIFEST.md) for full data quality documentation.

---

## 8. Recommendations

| Priority | Recommendation |
|----------|----------------|
| High | Improve TTB: richer features, longer context, or target bucketing |
| High | Add confidence intervals / calibration for fused p_fail |
| Medium | Quantize ONNX (INT8) to reduce image inference latency |
| Medium | Calibrate image threshold with more clean pump images |
| Low | Fine-tune image model on pump-specific corrosion dataset |

---

