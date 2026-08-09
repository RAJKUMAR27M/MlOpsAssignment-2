# Assignment 2 Final Submission Report

This report documents the implemented MLOps pipeline for Cats vs Dogs classification and the exact procedure used to reproduce the results. The demo video is the only remaining external deliverable.

## 1. What Was Implemented

### 1.1 Model development and tracking
1. DVC pipeline for preprocessing, training, and evaluation: `dvc.yaml`, `dvc.lock`
2. Baseline CNN model and serialized artifact: `src/model.py`, `models/cnn_latest.pt`
3. MLflow experiment logging for parameters, metrics, and artifacts: `src/train.py`, `src/evaluate.py`, `mlruns/`

### 1.2 Packaging and containerization
1. FastAPI inference service with `/health` and `/predict`: `app/main.py`, `app/utils.py`
2. Pinned dependencies: `requirements.txt`, `requirements-dev.txt`
3. Container and compose setup: `Dockerfile`, `docker-compose.yml`

### 1.3 CI and deployment
1. Unit tests for preprocessing and inference: `tests/test_preprocessing.py`, `tests/test_inference.py`
2. GitHub Actions workflow: `.github/workflows/ci-cd.yml`
3. Docker Compose deployment and smoke test flow: `docker-compose.yml`, `scripts/smoke_test.py`

### 1.4 Monitoring and post-deployment tracking
1. API request logging and latency counters: `app/main.py`
2. Prometheus config: `monitoring/prometheus.yml`
3. Post-deployment request simulation: `scripts/simulate_requests.py`

## 2. Step-by-Step Assignment Execution Procedure

### 2.1 Set up the environment

```bash
pip install -r requirements-dev.txt
```

### 2.2 Reproduce the DVC pipeline

```bash
python -m dvc repro
```

If DVC reports that outputs are already tracked by Git, run this once and retry:

```bash
git rm -r --cached data/processed
git rm --cached models/cnn_latest.pt artifacts/loss_curves.png artifacts/confusion_matrix.png
python -m dvc repro
```

Expected outputs:

1. `models/cnn_latest.pt`
2. `artifacts/loss_curves.png`
3. `artifacts/confusion_matrix.png`

### 2.3 Run the test suite

```bash
pytest tests -q
```

Expected result: all tests pass.

### 2.4 Run the API locally without Docker

Terminal 1:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Terminal 2:

```bash
python scripts/smoke_test.py --url http://localhost:8000
```

Expected result: `ALL TESTS PASSED`

### 2.5 Run the API with Docker Compose

```bash
docker compose up --build -d
python scripts/smoke_test.py --url http://localhost:8000
docker compose down
```

### 2.6 Run monitoring and request simulation

Optional monitoring stack:

```bash
docker compose -f monitoring/docker-compose.monitoring.yml up -d
```

Optional post-deployment tracking:

```bash
python scripts/simulate_requests.py --url http://localhost:8000 --num-requests 20
```

## 3. CI/CD Execution Logic

File: `.github/workflows/ci-cd.yml`

1. Pull request to main:
   1. Install dependencies
   2. Run tests
   3. Build Docker image
2. Push to main:
   1. Run tests
   2. Build and push image to GHCR
   3. Deploy with Docker Compose
   4. Wait for healthy service
   5. Execute smoke tests
   