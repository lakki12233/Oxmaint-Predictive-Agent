# Optimization Study 

Answers to the five key questions about model optimization and deployment.

---

## Q1: Can/Should Models Be Fine-Tuned?

| Model | Fine-Tune? | Why |
|-------|------------|-----|
| Sensor (LightGBM) | **YES** | Using default hyperparameters. Tuning could improve accuracy 10-20% |
| Image - MobileNetV3 (ONNX) | No | Already 100% accuracy on test set. Monitor in production first |
| Image - CLIP (ONNX) | Not Recommended | 70% accuracy with severe false positives. Use MobileNetV3 instead |
| Environmental | No | Rule-based, works well, interpretable |

### Sensor Fine-Tuning Strategy
1. **Hyperparameters**: Tune `num_leaves`, `learning_rate`, `n_estimators` with Optuna
2. **Features**: Add FFT, autocorrelation, cross-sensor ratios
3. **Class Balancing**: Use SMOTE or class weights for rare BROKEN samples

### Image Model Strategy
- **Current:** MobileNetV3 (100% test accuracy, 6 MB, 42ms inference)
- **Tested:** CLIP (70% test accuracy, 335 MB, 100ms inference) - rejected due to false positives
- **Next Steps:** Monitor MobileNetV3 performance in production; consider EfficientNet if needed

---

## Q2: Structured Reasoning Flow and Ensemble Methods?

### Current Fusion Flow
```
Sensor p_fail → Image rust detection → Environmental multiplier → Final prediction
```

### Ensemble Recommendation

| Strategy | Benefit | Trade-off |
|----------|---------|-----------|
| Multi-seed LightGBM | 2-5% better accuracy | Minimal latency impact |
| LightGBM + XGBoost + CatBoost | 5-10% better accuracy | 3x inference time |

**Recommendation**: If latency budget allows (200-300ms), use heterogeneous ensemble.

---

## Q3: Best Deployment Approach?

### Recommendation: Docker on Cloud VM

| Option | Pros | Cons |
|--------|------|------|
| **Docker on VM (ECS/Cloud Run)** ⭐ | Simple, scalable, portable | Cold start latency |
| Serverless (Lambda) | Pay-per-use | Cold start, model size limits |
| Kubernetes | Advanced orchestration | Overkill for single service |
| Edge | No network latency | Hard to update models |

**Best Choice**: AWS ECS Fargate or GCP Cloud Run
- Auto-scaling based on traffic
- ~$50-150/month for 2-5 instances
- Easy rollback between versions

---

## Q4: Database Connections Needed?

| Use Case | Database Type | Priority |
|----------|---------------|----------|
| **Prediction Logging** | TimescaleDB | ⭐ High |
| Asset Metadata | PostgreSQL | Medium |
| Model Versioning | S3 + PostgreSQL | Low |

### Recommended Architecture
```
FastAPI → Message Queue → Logging Service → TimescaleDB
                                ↓
                           PostgreSQL (assets, model versions)
```

**Minimum**: Add TimescaleDB for prediction logging before production.

---

## Q5: Latency Bottleneck and Mitigation?

### What We Found

**Original Problem**: API was slow (~850ms per request, only 11 requests/second)

**Root Causes**:
1. Pandas rolling window calculations were slow for feature extraction
2. Single-threaded server couldn't handle concurrent requests

### What We Fixed

| Problem | Solution | Result |
|---------|----------|--------|
| Slow pandas rolling stats | Replaced with NumPy direct computation | 4-5x faster |
| Single worker bottleneck | Multi-worker uvicorn (4 workers) | 3-4x more throughput |

### Optimization Journey

| Stage | Throughput | P50 Latency | P95 Latency | Change |
|-------|-----------|-------------|-------------|--------|
| **Baseline** (single worker, pandas) | 11 req/s | 850 ms | 960 ms | - |
| **Multi-worker** (4 workers) | 42 req/s | 220 ms | 280 ms | +3.8x |
| **NumPy features** | 101 req/s | 89 ms | 147 ms | +9.2x total |

**Final Performance (200 requests, concurrency=10):**
- Throughput: **94.6 req/s**
- P50 Latency: **88.3 ms**
- P95 Latency: **213.1 ms**
- Error Rate: **0%**

### Remaining Bottleneck

ONNX image inference is now the slowest part (42ms, 52% of total time).

| Component | Time | % |
|-----------|------|---|
| ONNX inference | 42ms | 52% |
| Image preprocessing | 12ms | 15% |
| Base64 decode | 8ms | 10% |
| Sensor inference | 6ms | 8% |
| Other | 12ms | 15% |

### Future Improvements

| Step | Action | Gain | Status |
|------|--------|------|--------|
| 1 | NumPy feature extraction | 4-5x faster | ✅ Done |
| 2 | Multi-worker deployment | 3-4x throughput | ✅ Done |
| 3 | Quantize ONNX to INT8 | 2-3x faster | **Next** |
| 4 | GPU acceleration | 5-10x faster | If needed |
| 5 | Binary image upload | 10-20ms saved | Nice-to-have |

---

## Load Testing Methodology

**Tool**: Locust (Python load testing framework)

**Test Script**: `tests/scripts/load_test.py`

**Configuration**:
- Request levels: 10, 50, 200
- Concurrency: 10 users
- Endpoint: POST /predict with sensor + image data
- Host: localhost:8000 (Docker container)

**Run Load Test**:
```powershell
cd tests/scripts
locust -f load_test.py --headless -u 10 -r 2 -t 30s --host http://localhost:8000
```

---




