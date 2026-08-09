# Assignment 2 Final Submission Report

This submission implements an end-to-end MLOps pipeline for Cats vs Dogs classification and is ready for evaluation, excluding only the demo video.

## Latest Verification Result

The current workspace was re-validated after fixing the DVC script execution path and output-tracking conflicts.

Verified results:

1. `python -m dvc repro` completes successfully.
2. `pytest tests -q` passes.
3. `python scripts/create_submission_zip.py` creates `submission_assignment2_ready.zip` and keeps it under 10 MB.

Explanation:

1. `src/train.py` and `src/evaluate.py` now support direct script execution from DVC by adding the repository root to `sys.path` when needed.
2. DVC-managed outputs are no longer tracked by Git, which prevents `dvc repro` from failing on `data/processed`, `models/cnn_latest.pt`, and the artifact images.
3. `src/evaluate.py` uses pure `matplotlib` for the confusion matrix plot, so evaluation no longer depends on the broken `seaborn`/`pandas` path in this environment.

## A. Rubric Compliance

### M1. Model Development and Experiment Tracking
1. Data and pipeline versioning with DVC: `dvc.yaml`, `dvc.lock`
2. Baseline model implementation and artifact: `src/model.py`, `models/cnn_latest.pt`
3. Training and evaluation with MLflow logging: `src/train.py`, `src/evaluate.py`, `mlruns/`

### M2. Packaging and Containerization
1. REST API endpoints:
	1. Health endpoint: `/health`
	2. Prediction endpoint: `/predict`
2. Dependencies pinned in `requirements.txt` and `requirements-dev.txt`
3. Containerization files: `Dockerfile`, `docker-compose.yml`

### M3. CI for Build, Test, and Image Creation
1. Unit tests:
	1. Preprocessing tests: `tests/test_preprocessing.py`
	2. Inference/model tests: `tests/test_inference.py`
2. CI workflow: `.github/workflows/ci-cd.yml`
3. Image build and registry push to GHCR in workflow on push to main

### M4. CD and Deployment
1. Deployment target: Docker Compose
2. Automated deployment in CI/CD workflow on push to main
3. Post-deploy smoke test: `scripts/smoke_test.py`

### M5. Monitoring and Post-Deployment Tracking
1. Request logging and latency/request counters in API: `app/main.py`
2. Prometheus monitoring config: `monitoring/prometheus.yml`
3. Post-deployment request simulation and summary outputs: `scripts/simulate_requests.py`

## B. Step-by-Step Execution Procedure

### 1. Environment Setup

```bash
pip install -r requirements-dev.txt
```

### 2. Reproduce Data, Model, and Evaluation Artifacts

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

### 3. Run Unit Tests

```bash
pytest tests -q
```

### 4. Run API Locally (Without Docker)

Terminal 1:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Terminal 2:

```bash
python scripts/smoke_test.py --url http://localhost:8000
```

Expected: `ALL TESTS PASSED`

### 5. Run API with Docker Compose

```bash
docker compose up --build -d
python scripts/smoke_test.py --url http://localhost:8000
docker compose down
```

### 6. Optional Monitoring Stack

```bash
docker compose -f monitoring/docker-compose.monitoring.yml up -d
```

### 7. Optional Post-Deployment Tracking Run

```bash
python scripts/simulate_requests.py --url http://localhost:8000 --num-requests 20
```

## C. CI/CD Execution Logic

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