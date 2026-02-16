# Oxmaint Predictive Maintenance Agent

**A multimodal AI system for predicting pump failures using sensor data, image analysis, and environmental context.**

[![Model Version](https://img.shields.io/badge/model-v2.2.0-blue)]()
[![Modalities](https://img.shields.io/badge/modalities-3-green)]()
[![Latency](https://img.shields.io/badge/p50_latency-89ms-brightgreen)]()
[![Throughput](https://img.shields.io/badge/throughput-100_req/s-brightgreen)]()

---

## �️ Getting Started Path

**New to this project?** Follow this path:

1. **📖 Read this README** - Understand what the system does (5 min)
2. **🚀 Follow [QUICK_START.md](QUICK_START.md)** - Get the system running (5 min)
3. **🏗️ Read [ARCHITECTURE.md](ARCHITECTURE.md)** - Understand system design
4. **🎨 Try the [UI](ui/README.md)** - Test all features visually (10 min)
5. **📊 Review [EVALUATION_REPORT.md](EVALUATION_REPORT.md)** - See model performance

---

## 🎯 Overview

This system implements a production-ready predictive maintenance agent for centrifugal pumps that:
- **Predicts failure probability** (0-1 scale) and estimated time to breakdown
- **Identifies fault types** (bearing failure, seal leak, corrosion/rust, environmental stress)
- **Integrates 3 modalities**: Sensor time-series, visual inspection (rust detection), operational context
- **Handles missing modalities gracefully** (any subset of inputs accepted)
- **Achieves <100ms p50 latency** at 100 requests/second throughput

---

## ✨ Features

### Three Modalities

| Modality | Type | Key Details |
|----------|------|-------------|
| **Sensor** | Mandatory | LightGBM, AUC 0.998, TTB prediction (MAE: 270h) |
| **Image** | Optional | MobileNetV3 CNN (6MB, ~15ms), rust detection >94% confidence |
| **Environmental** | Optional | Rule-based risk scoring (0.5-2.0x multiplier) |

### Production Features

- RESTful API (`/predict`, `/predict/batch`, `/health`)
- Docker containerization + multi-worker uvicorn
- Human-readable explanations for every prediction
- 22 automated pytest tests

---

## 🚀 Quick Start

**Get started in under 5 minutes!**

### Option 1: Docker (Recommended)

```powershell
# Build and run
docker build -f docker/Dockerfile -t oxmaint-api .
docker run --rm -p 8000:8000 --name oxmaint-api oxmaint-api

# Test it
Invoke-RestMethod "http://localhost:8000/health"
```

### Option 2: Direct Python

```powershell
# Install and run
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000

# Test it
Invoke-RestMethod "http://localhost:8000/health"
```

### Option 3: With UI

```powershell
# Terminal 1: API
docker run --rm -p 8000:8000 oxmaint-api

# Terminal 2: UI
cd ui; python serve_ui.py

# Open: http://localhost:8080/ui/ui.html
```

**📖 For detailed setup instructions, see [QUICK_START.md](QUICK_START.md)**

---

## 🔄 Model Switching

| Model | Speed | Accuracy | Size | Recommended |
|-------|-------|----------|------|-------------|
| **MobileNetV3** | 19-55ms | 100% | 6 MB | ✅ **Yes** |
| **CLIP** | 64-127ms | 70% | 335 MB | ❌ No (false positives) |

**Switch via environment variable:** `$env:RUST_MODEL_TYPE="mobilenet"` (default) or `"clip"`

> **CLIP files** (335 MB) not included - download from [Google Drive](https://drive.google.com/file/d/1F_SfsV89RgQvpJyD_w_Bd-_dJs-uuwuE/view?usp=sharing) if needed. See [VLM_EVALUATION.md](docs/VLM_EVALUATION.md) for comparison details.

---

## Project Structure

| Folder | Purpose |
|--------|---------|
| `app/` | FastAPI application + modalities (sensor, image, environmental) |
| `artifacts/` | Trained models (LightGBM, ONNX rust detection) |
| `tests/` | Pytest suite (22 tests), test images, testing scripts |
| `train/` | LightGBM training script |
| `Rust_Detection_Notebook/` | Colab notebooks for rust model training |
| `ui/` | Web-based testing interface |
| `docker/` | Dockerfile for containerization |
| `docs/` | Additional documentation |
| `data/` | Dataset and manifest |
| `samples/` | Example API request payloads |

---

## 📊 Model Outputs

### Response Schema

```json
{
  "asset_id": "pump_017",
  "failure_probability": 0.497,
  "estimated_time_to_breakdown_hours": 448.79,
  "predicted_fault_type": "corrosion_rust",
  "fault_confidence": 0.995,
  "top_signals": [
    "failure_risk_medium",
    "img:rust(p=0.995)",
    "stable_vibration",
    "no_sensor_spike",
    "env:maint_critical(90d)"
  ],
  "inference_ms": 46,
  "model_version": "sensor_lgbm_v2_multimodal",
  "explanation": "CAUTION: Moderate failure risk (49.7%). Visual inspection detected rust/corrosion on pump components. Estimated breakdown: ~18 days. Schedule maintenance soon. Environmental conditions are significantly increasing risk. Key factors: img:rust(p=0.995), stable_vibration."
}
```

### Field Explanation

| Field | Description |
|-------|-------------|
| `asset_id` | Asset identifier from request |
| `failure_probability` | Probability (0-1) of failure state (considering all modalities) |
| `estimated_time_to_breakdown_hours` | Predicted hours until next failure |
| `predicted_fault_type` | Fault classification: `bearing_failure`, `seal_leak`, `corrosion_rust`, `environmental_stress`, or `null` |
| `fault_confidence` | Confidence score (0-1) for fault type prediction |
| `top_signals` | Top 5 contributing factors (sensor, image, environmental) |
| `inference_ms` | End-to-end inference latency in milliseconds |
| `model_version` | Model version identifier |
| `explanation` | **Human-readable explanation** of the prediction for operators/stakeholders |

### Fault Types

- **`bearing_failure`**: High vibration, acoustic anomalies
- **`seal_leak`**: Pressure drop, flow rate anomalies
- **`corrosion_rust`**: Rust detected in image analysis
- **`environmental_stress`**: Critical operating conditions (high hours, overdue maintenance, extreme environment)
- **`null`**: No fault detected (normal operation)
---

## 🔌 API Documentation

### Endpoints

| Method | Endpoint | Description | Request Type |
|--------|----------|-------------|--------------|
| `GET` | `/health` | Service status, model versions | N/A |
| `POST` | `/predict` | Single asset prediction | `PredictRequest` |
| `POST` | `/predict/batch` | Batch predictions | `List[PredictRequest]` |

### Request Schema

```json
{
  "asset_id": "pump_001",
  "timestamp": "2026-02-14T14:30:00Z",
  "sensor_window": [
    {"ts": "2026-02-14T14:28:00Z", "sensor_00": 0.1, "sensor_01": 0.2, ...},
    {"ts": "2026-02-14T14:28:10Z", "sensor_00": 0.2, "sensor_01": 0.1, ...}
  ],
  "image_base64": "iVBORw0KGgoAAAANSUhEUgAA...",  // Optional
  "environmental": {                            // Optional
    "operating_hours": 2500,
    "days_since_last_maintenance": 90,
    "ambient_temperature_c": 48,
    "ambient_humidity_percent": 88,
    "load_factor": 0.98,
    "maintenance_overdue": true
  }
}
```

### Environmental Fields

See [docs/ENVIRONMENTAL_MODALITY.md](docs/ENVIRONMENTAL_MODALITY.md) for detailed environmental risk logic and field descriptions.

---

## 🏋️ Training

### Dataset
- **Source**: [Kaggle Pump Sensor Data](https://www.kaggle.com/datasets/nphantawee/pump-sensor-data)
- **Size**: ~220k time-series samples
- **Features**: 52 sensor readings per sample
- **Labels**: NORMAL, BROKEN, RECOVERING

### Train Models

```powershell
# Place dataset at data/sensor.csv
python train\train_pump_lgbm.py
```

**Outputs**:
- `artifacts/pump_failure_lgbm.joblib` (failure classifier)
- `artifacts/pump_ttb_lgbm.joblib` (time-to-breakdown regressor)
- `artifacts/train_metrics.json` (performance metrics)

**Training Metrics** (see `EVALUATION_REPORT.md`):
- Failure classification: AUC 0.998, AP 0.965
- Time-to-breakdown MAE: 270 hours

---

## 🧪 Testing

### Automated Test Suite (pytest)

```powershell
# Run all 22 automated tests
pytest tests/api/test_api.py -v
```

### Manual Testing Scripts

| Test | Command/Script |
|------|----------------|
| Sensor-only | `Invoke-RestMethod -Uri "http://localhost:8000/predict" -Method Post -ContentType "application/json" -Body (Get-Content .\samples\request.json -Raw)` |
| Environmental | `.\tests\scripts\test_environmental.ps1` |
| Full Multimodal | `.\tests\scripts\test_multimodal_complete.ps1` |

Test images are located in `tests/images/rust/` and `tests/images/no_rust/`.

---

##  Web UI for Testing

A web-based testing interface with 9 pre-configured test cases.

```powershell
# Terminal 1: API
docker run --rm -p 8000:8000 oxmaint-api

# Terminal 2: UI
cd ui; python serve_ui.py

# Open: http://localhost:8080/ui/ui.html
```

**See [ui/README.md](ui/README.md) for full documentation.**

---

## ⚡ Performance

**Final Performance (Load Test, 200 requests, concurrency=10):**
- Throughput: **94.6 req/s** | P50 Latency: **88.3 ms** | P95 Latency: **213.1 ms** | Error Rate: **0%**

**Bottleneck**: ONNX image inference (42ms, 52% of total). Mitigate with GPU acceleration or INT8 quantization.

See [OPTIMIZATION_STUDY.md](OPTIMIZATION_STUDY.md) for detailed analysis.

---

## 📚 Documentation

**Essential:**
- [QUICK_START.md](QUICK_START.md) - Setup guide (5 min)
- [ARCHITECTURE.md](ARCHITECTURE.md) - System design + diagrams
- [EVALUATION_REPORT.md](EVALUATION_REPORT.md) - Model metrics + ablation studies

**Technical:**
- [AI_USAGE.md](AI_USAGE.md) - GenAI usage disclosure
- [OPTIMIZATION_STUDY.md](OPTIMIZATION_STUDY.md) - Performance analysis
- [data/DATASET_MANIFEST.md](data/DATASET_MANIFEST.md) - Dataset sources + licenses

**Specialized:**
- [docs/VLM_EVALUATION.md](docs/VLM_EVALUATION.md) - MobileNetV3 vs CLIP comparison
- [docs/ENVIRONMENTAL_MODALITY.md](docs/ENVIRONMENTAL_MODALITY.md) - Environmental risk logic
- [docs/CLOUD_DEPLOYMENT_GUIDE.md](docs/CLOUD_DEPLOYMENT_GUIDE.md) - Production deployment
- [Rust_Detection_Notebook/MODEL_TRAINING_GUIDE.md](Rust_Detection_Notebook/MODEL_TRAINING_GUIDE.md) - Model training guide
- [ui/README.md](ui/README.md) - Web UI guide 

---
