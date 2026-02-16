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
3. **🎨 Try the [UI](ui/README.md)** - Test all features visually (10 min)
4. **📊 Review [EVALUATION_REPORT.md](EVALUATION_REPORT.md)** - See model performance
5. **🏗️ Read [ARCHITECTURE.md](ARCHITECTURE.md)** - Understand system design

---

## �📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Model Outputs](#model-outputs)
- [API Documentation](#api-documentation)
- [Training](#training)
- [Testing](#testing)
- [Web UI for Testing](#web-ui-for-testing)
- [Docker Deployment](#docker-deployment)
- [Performance](#performance)
- [Documentation](#documentation)
- [Deliverables Checklist](#deliverables-checklist)

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

### 🔬 Three Modalities

1. **Sensor Modality** (Mandatory)
   - LightGBM models trained on 220k pump sensor samples
   - Real-time feature extraction (mean, std, slope, min/max)
   - AUC 0.998 on failure classification (robust for imbalanced data)
   - Time-to-breakdown prediction (MAE: 270 hours)

2. **Image Modality** (Optional)
   - **MobileNetV3 CNN** (default, recommended for production)
   - 6MB model size, ~10-15ms inference, 66-100% accuracy in testing
   - Detects corrosion/rust with >94% confidence on positive cases
   - Processes base64-encoded JPEG/PNG images
   - Fusion triggers at 85% confidence threshold
   - Alternative CLIP VLM available but not recommended (see [VLM_EVALUATION.md](docs/VLM_EVALUATION.md))
   - Switchable via `RUST_MODEL_TYPE` environment variable (mobilenet|clip)
   - **Training**: See [MODEL_TRAINING_GUIDE.md](Rust_Detection_Notebook/MODEL_TRAINING_GUIDE.md) for full training process on Google Colab

3. **Environmental Modality** (Optional)
   - Rule-based risk scoring using operational context
   - Factors: operating hours, maintenance history, ambient conditions, load
   - Adjusts failure probability by 0.5-2.0x multiplier
   - Transparent, interpretable logic

### 🚀 Production Features

- ✅ RESTful API with `/predict`, `/predict/batch`, `/health` endpoints
- ✅ Pydantic schema validation with detailed field descriptions
- ✅ Docker containerization (one-command deployment)
- ✅ Multi-worker uvicorn (4 workers)
- ✅ Optimized NumPy feature extraction (4-5x faster than pandas)
- ✅ Graceful error handling for missing/malformed inputs
- ✅ Inference latency tracking per request
- ✅ **Human-readable explanations** for every prediction
- ✅ **Comprehensive pytest test suite** (22 automated tests)
- ✅ **Dataset manifest** with source, license, and data quality documentation

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

The system supports two rust detection models:

| Model | Speed | Accuracy | Size | Best For |
|-------|-------|----------|------|----------|
| **MobileNetV3** | ⚡ 19-55ms | ✅ 100% (10/10 test images) | 6 MB | **Production** (Recommended) |
| **CLIP** | 🐢 64-127ms | ⚠️ 70% (3/5 false positives on clean) | 335 MB | Research/Comparison only |

**⚠️ Important**: CLIP has severe false positive issues on clean surfaces (see [VLM_EVALUATION.md](docs/VLM_EVALUATION.md)). **MobileNetV3 is strongly recommended for production use.**

**Switch models via environment variable:**

```powershell
# Use MobileNetV3 (default)
$env:RUST_MODEL_TYPE="mobilenet"
uvicorn app.main:app --host 0.0.0.0 --port 8000

# Use CLIP
$env:RUST_MODEL_TYPE="clip"
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### 📥 CLIP Model Download (Optional)

The CLIP model files are **not included** in the repository due to size (335 MB). If you want to experiment with CLIP:

1. **Download the CLIP model files** from [Google Drive](https://drive.google.com/file/d/1F_SfsV89RgQvpJyD_w_Bd-_dJs-uuwuE/view?usp=sharing) or train your own using the notebook in `Rust_Detection_Notebook/`

2. **Place the files in the artifacts folder:**
   ```
   artifacts/
   ├── rust_clip.onnx           # CLIP model architecture
   └── rust_clip.onnx.data      # CLIP model weights (335 MB)
   ```

3. **Set the environment variable and restart:**
   ```powershell
   $env:RUST_MODEL_TYPE="clip"
   uvicorn app.main:app --host 0.0.0.0 --port 8000
   ```

> **Note**: CLIP is provided for research comparison only. MobileNetV3 outperforms CLIP on our test set (100% vs 70% accuracy) and is 10x smaller.

---

##  Project Structure

```
oxmaint-predictive-agent/
│
├── app/
│   ├── main.py                     # FastAPI app entrypoint
│   ├── schemas.py                  # Pydantic request/response schemas
│   └── modalities/
│       ├── sensor.py               # Sensor inference + feature extraction
│       ├── image.py                # ONNX rust detection
│       └── environmental.py        # Environmental risk scoring
│
├── artifacts/                      # Trained model artifacts
│   ├── pump_failure_lgbm.joblib    # Failure classification model
│   ├── pump_ttb_lgbm.joblib        # Time-to-breakdown regression
│   ├── rust_model.onnx             # Rust detection - MobileNetV3 (default)
│   ├── rust_model.onnx.data        # ONNX model data file
│   ├── rust_clip.onnx              # Rust detection - CLIP (optional)
│   ├── rust_labels.json            # Image model labels (shared)
│   ├── feature_schema.json         # Sensor feature names
│   └── train_metrics.json          # Training performance metrics
│
├── tests/                          # All test-related resources
│   ├── api/                        # Automated test suite
│   │   ├── conftest.py             # Pytest configuration
│   │   └── test_api.py             # API endpoint tests (22 tests)
│   ├── images/                     # Test images
│   │   ├── rust/                   # Rust detection test images
│   │   └── no_rust/                # Clean surface test images
│   └── scripts/                    # Testing utilities
│       ├── make_demo_batch.py      # Batch request generator
│       ├── load_test.py            # Performance load testing
│       ├── test_environmental.ps1  # Environmental modality tests
│       └── test_multimodal_complete.ps1 # Full 3-modality tests
│
├── train/
│   └── train_pump_lgbm.py          # LightGBM training script
│
├── Rust_Detection_Notebook/        # Rust detection model training notebooks
│   ├── MODEL_TRAINING_GUIDE.md     # Complete training documentation
│   ├── VLM_rust_detection_training.ipynb  # CLIP model training (Colab)
│   ├── rust_detection_model_training.ipynb # MobileNetV3 training (Colab)
│   └── What we did in Colab.pdf    # Training process screenshots
│
├── ui/                             # Web-based testing interface
│   ├── ui.html                     # Test UI (HTML+CSS+JS)
│   ├── serve_ui.py                 # Simple HTTP server with CORS
│   └── README.md                   # UI documentation
│
├── docker/
│   └── Dockerfile                  # Docker configuration
│
├── docs/
│   ├── ENVIRONMENTAL_MODALITY.md   # Environmental modality documentation
│   └── CLOUD_DEPLOYMENT_GUIDE.md   # Production deployment guide
│
├── data/                           # Datasets and data documentation
│   ├── sensor.csv                  # Pump sensor time series data
│   └── DATASET_MANIFEST.md         # Data sources, licenses, quality checks
│
├── samples/                        # Example API requests
│   ├── request.json                # Single prediction
│   ├── batch.json                  # Batch prediction
│   ├── env_critical.json           # Critical environmental conditions
│   └── env_favorable.json          # Favorable operating conditions
│
├── requirements.txt                # Python dependencies
├── .gitignore                      # Git ignore patterns
├── README.md                       # This file
├── AI_USAGE.md                     # GenAI usage documentation
├── OPTIMIZATION_STUDY.md           # Task 8 optimization analysis
├── ARCHITECTURE.md                 # System architecture + diagrams
├── EVALUATION_REPORT.md            # Metrics, ablation, error analysis
└── FINAL_SUBMISSION_CHECKLIST.md   # Pre-submission verification
```

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

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `operating_hours` | float | Total hours since last maintenance | 2500.0 |
| `days_since_last_maintenance` | float | Days elapsed since last service | 90.0 |
| `ambient_temperature_c` | float | Ambient temperature (Celsius) | 48.0 |
| `ambient_humidity_percent` | float | Ambient humidity (%) | 88.0 |
| `load_factor` | float | Current load vs rated capacity (0-1) | 0.98 |
| `maintenance_overdue` | bool | Whether maintenance schedule is overdue | true |

See [docs/ENVIRONMENTAL_MODALITY.md](docs/ENVIRONMENTAL_MODALITY.md) for detailed environmental risk logic.

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

# Run specific test class
pytest tests/api/test_api.py::TestPredictEndpoint -v

# Run with coverage (if installed)
pytest tests/api/test_api.py --cov=app --cov-report=html
```

**Test Categories:**
- `TestHealthEndpoint`: Health check endpoint validation (3 tests)
- `TestPredictEndpoint`: Single prediction functionality (8 tests)
- `TestInputValidation`: Schema validation and error handling (5 tests)
- `TestBatchEndpoint`: Batch prediction functionality (4 tests)
- `TestFeatureExtraction`: Sensor feature handling (2 tests)

### Test Sensor-Only

```powershell
$body = Get-Content .\samples\request.json -Raw
Invoke-RestMethod -Uri "http://localhost:8000/predict" `
  -Method Post -ContentType "application/json" -Body $body | ConvertTo-Json -Depth 10
```

### Test Image Modality

**Note**: These tests use **MobileNetV3** by default. To test with CLIP, set `$env:RUST_MODEL_TYPE="clip"` before starting the API.

```powershell
# Test all rust/no_rust images (uses MobileNetV3 by default)
$base = "C:\oxmaint-predictive-agent\tests\images"
$endpoint = "http://localhost:8000/predict"

# Load sensor window once
$req = Get-Content ".\samples\request.json" -Raw | ConvertFrom-Json

function Invoke-ImagePredict($imgPath, $assetId) {
  $b64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes($imgPath))
  $bodyObj = @{
    asset_id = $assetId
    timestamp = (Get-Date).ToString("o")
    sensor_window = $req.sensor_window
    image_base64 = $b64
  }
  $body = $bodyObj | ConvertTo-Json -Depth 50
  Invoke-RestMethod -Uri $endpoint -Method Post -ContentType "application/json" -Body $body
}

# Test rust images
Write-Host "=== Testing RUST images ==="
Get-ChildItem "$base\rust" -Filter *.jpg | ForEach-Object {
  $resp = Invoke-ImagePredict $_.FullName "rust_$($_.BaseName)"
  $resp | ConvertTo-Json -Depth 10
}

# Test no_rust (clean) images
Write-Host "=== Testing NO_RUST images ==="
Get-ChildItem "$base\no_rust" | ForEach-Object {
  $resp = Invoke-ImagePredict $_.FullName "clean_$($_.BaseName)"
  $resp | ConvertTo-Json -Depth 10
}
```

### Test Environmental Modality

```powershell
# Test environmental risk scenarios
.\tests\scripts\test_environmental.ps1

# Or test individual scenarios
$body = Get-Content .\samples\env_critical.json -Raw
Invoke-RestMethod -Uri "http://localhost:8000/predict" `
  -Method Post -ContentType "application/json" -Body $body | ConvertTo-Json -Depth 10
```

### Test Full Multimodal (Sensor + Image + Environmental)

```powershell
.\tests\scripts\test_multimodal_complete.ps1
```

---

## 🎨 Web UI for Testing

A simple web-based testing interface is available for easy visual testing of all API endpoints.

### Quick Start

**Terminal 1 - Start API:**
```powershell
docker build -t oxmaint-api -f docker\Dockerfile .
docker run --rm -p 8000:8000 --name oxmaint-api oxmaint-api
```

**Terminal 2 - Start UI:**
```powershell
cd ui
python serve_ui.py
```

**Open in browser:** http://localhost:8080/ui/ui.html

### Features

- ✅ **9 Pre-configured Test Cases** - All modality combinations including VLM
- ✅ **Visual Results** - See predictions, metrics, and explanations
- ✅ **Image Preview** - View uploaded images with rust detection
- ✅ **Batch Testing** - Run all tests or individual tests
- ✅ **Live Health Check** - Real-time API status

### Available Tests

1. **Health Check** - Verify API is running
2. **Sensor Only** - Basic prediction using sensor data
3. **Environmental - Critical** - High-risk conditions
4. **Environmental - Favorable** - Normal operating conditions
5. **Image - Rust Detected** - Visual corrosion detection
6. **Image - Clean Surface** - No rust detected
7. **Multimodal Complete** - All three modalities combined
8. **VLM Multimodal (CLIP)** - Vision-language model testing
9. **Batch Prediction** - Process multiple assets

**See [ui/README.md](ui/README.md) for full documentation and troubleshooting.**

---

## 🐳 Docker Deployment

### Build Image

```powershell
docker build -t oxmaint-api -f docker\Dockerfile .
```

### Run Container

```powershell
docker run --rm -p 8000:8000 --name oxmaint-api oxmaint-api
```

### Test Dockerized API

```powershell
# Health check
Invoke-RestMethod "http://localhost:8000/health" | ConvertTo-Json -Depth 10

# Verify artifacts loaded
docker exec -it oxmaint-api ls -lah /app/artifacts

# Test prediction (see testing section above)
```

### Docker Verification Results

| Test | Result | Details |
|------|--------|---------|
| Health | ✅ | `status: ok`, all models loaded |
| Sensor-only | ✅ | ~21ms, low risk predicted |
| Environmental (critical) | ✅ | ~11ms, `environmental_stress` fault |
| Image (rust) | ✅ | ~45ms, `corrosion_rust` fault, ~99.5% confidence |
| Image (clean) | ✅ | ~70ms, `no_rust`, no fault triggered (below threshold) |
| Batch (3 assets) | ✅ | ~34ms total, all predictions correct |

### What Docker Does

- **Packages**: API code, model artifacts, dependencies into portable image
- **Installs**: Python dependencies, Linux libraries (libgomp1 for LightGBM)
- **Exposes**: Port 8000 for API access
- **Starts**: Uvicorn server with 4 workers (multi-process)

---

## ⚡ Performance

### Load Test Results

**Methodology**: Locust load testing at 10-50-200 request levels, concurrency=10

| Optimization Stage | Throughput (req/s) | P50 Latency | P95 Latency |
|--------------------|-------------------|-------------|-------------|
| **Baseline** (single worker, pandas) | 11 | 850 ms | 960 ms |
| **Multi-worker** (4 workers) | 42 | 220 ms | 280 ms |
| **NumPy features** | 101 | 89 ms | 147 ms |

**Final Performance** (200 requests):
- Throughput: **94.6 req/s**
- P50 Latency: **88.3 ms**
- P95 Latency: **213.1 ms**
- Error Rate: **0%**

### Bottleneck Analysis

| Component | Latency (ms) | % of Total |
|-----------|--------------|------------|
| ONNX image inference | 42 | 52% |
| Sensor feature extraction | 7 | 9% |
| LightGBM inference | 6 | 8% |
| Base64 decode | 8 | 10% |
| Image preprocessing | 13 | 15% |
| Environmental logic | <1 | <1% |
| Fusion + serialization | 4 | 5% |

**Current Bottleneck**: ONNX image inference (42ms)  
**Mitigation**: GPU acceleration (5-10x faster) or INT8 quantization (2-3x faster)

See [OPTIMIZATION_STUDY.md](OPTIMIZATION_STUDY.md) for detailed analysis.

---

## 📚 Documentation

### 🎯 Start Here

| Document | Purpose | When to Read |
|----------|---------|-------------|
| **[README.md](README.md)** | Project overview and features | You are here! Read first |
| **[QUICK_START.md](QUICK_START.md)** | Step-by-step setup guide | Getting started (5 min) |

### 📖 Core Documentation

| Document | Purpose |
|----------|---------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design, data flow, and architecture diagrams |
| [EVALUATION_REPORT.md](EVALUATION_REPORT.md) | Model performance metrics and ablation studies |
| [docs/VLM_EVALUATION.md](docs/VLM_EVALUATION.md) | Vision-language model comparison (MobileNetV3 vs CLIP) |

### 🔧 Technical References

| Document | Purpose |
|----------|---------|
| [AI_USAGE.md](AI_USAGE.md) | GenAI usage disclosure (required for submission) |
| [DATASET_MANIFEST.md](data/DATASET_MANIFEST.md) | Data sources, licenses, and quality checks |
| [OPTIMIZATION_STUDY.md](OPTIMIZATION_STUDY.md) | Performance optimization and bottleneck analysis |

### 📂 Specialized Guides

| Document | Purpose |
|----------|---------|
| [docs/ENVIRONMENTAL_MODALITY.md](docs/ENVIRONMENTAL_MODALITY.md) | Environmental risk scoring logic |
| [docs/CLOUD_DEPLOYMENT_GUIDE.md](docs/CLOUD_DEPLOYMENT_GUIDE.md) | Production deployment (AWS/GCP/Azure) |
| [Rust_Detection_Notebook/MODEL_TRAINING_GUIDE.md](Rust_Detection_Notebook/MODEL_TRAINING_GUIDE.md) | Complete guide for training MobileNetV3 and CLIP models |
| [ui/README.md](ui/README.md) | Web UI testing interface guide |

---

## ✅ Deliverables Checklist

### Task 1: Data Pipeline ✅
- [x] Ingestion scripts for sensor, image, environmental data
- [x] Unified schema keyed by `asset_id` and `timestamp`
- [x] Data quality checks (see `app/schemas.py` Pydantic validation)
- [x] Dataset manifest (see `EVALUATION_REPORT.md` section 1)

### Task 2: Preprocessing ✅
- [x] Sensor: windowing, scaling, derived features (NumPy-based)
- [x] Image: base64 decode, resize, normalize for ONNX
- [x] Environmental: structured context parsing

### Task 3: Modeling ✅
- [x] Sensor-only baseline (LightGBM, AUC 0.998)
- [x] Multimodal model (sensor + image + environmental)
- [x] Fusion logic combining modality outputs
- [x] Graceful handling of missing modalities

### Task 4: Agent Orchestration ✅
- [x] Inference orchestrator routing inputs to models
- [x] Structured output with confidence signals
- [x] Machine-generated explanations (`top_signals`)

### Task 5: API Service ✅
- [x] `POST /predict`, `POST /predict/batch`, `GET /health`
- [x] Schema validation (Pydantic)
- [x] Error handling with clear messages
- [x] Inference time tracking per request

### Task 6: Deployment ✅
- [x] Docker containerization
- [x] One-command local startup
- [x] Deployment instructions (see above + `OPTIMIZATION_STUDY.md`)

### Task 7: Scale and Performance ✅
- [x] Load tests at 3 traffic levels (10/50/200 requests)
- [x] Throughput, p50/p95 latency metrics
- [x] Bottleneck identification (ONNX inference)
- [x] Optimization implemented (NumPy features, multi-worker)

### Task 8: Optimization Study ✅
- [x] Fine-tuning evaluation (see `OPTIMIZATION_STUDY.md` Q1)
- [x] Structured reasoning/ensemble analysis (Q2)
- [x] Deployment approach recommendation (Q3)
- [x] Database connection analysis (Q4)
- [x] Latency bottleneck mitigation (Q5)

### Task 9: Generative AI Usage ✅
- [x] AI-assisted code generation (70-80% of initial code)
- [x] Verification methods documented (`AI_USAGE.md`)
- [x] Transparency about AI contribution

---

## 🎓 Datasets Used

1. **Sensor Data**: [Kaggle Pump Sensor Data](https://www.kaggle.com/datasets/nphantawee/pump-sensor-data)
2. **Image Data**: 
   - **Training**: [Roboflow Rust Detection Dataset](https://universe.roboflow.com/test-stage/rust-detection-t8vza/dataset/8)
   - **Testing**: Manual photos of rust/no-rust pumps (`tests/images/`)
3. **Environmental Data**: Synthetic (rule-based generation for testing)

---

## 🚦 Next Steps for Production

1. **Deploy to Cloud**: AWS ECS Fargate or GCP Cloud Run (see `OPTIMIZATION_STUDY.md`)
2. **Add Prediction Logging**: TimescaleDB for model monitoring
3. **Collect Ground Truth**: Label actual failures for model retraining
4. **Model Fine-Tuning**: Hyperparameter optimization, add frequency-domain features
5. **GPU Acceleration**: Enable CUDA for ONNX inference (5-10x faster)

---
