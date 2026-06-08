# 🚚 Truck Delay Predictor — Deployment App

A self-contained **Streamlit** dashboard that predicts whether a FreshBasket truck shipment is **at risk of delay**. It serves a pre-trained **XGBoost** classifier (from the Truck Delay Classification spine project) by loading model artifacts directly from the local `artifacts/` folder — **no database, S3, or MLflow connection required at runtime.**

This repository is the deployment unit used in the **AWS MLOps Master Course — Module 5, Lab C** (CI/CD & Production Deployment to ECS). It is intentionally lightweight and stateless so it can be containerized, pushed to ECR, and deployed behind an Application Load Balancer on ECS Fargate.

---

## What's in this repo

| Path | Purpose |
|------|---------|
| `app.py` | Streamlit app: input form → preprocessing → XGBoost prediction → delay risk + probability |
| `Dockerfile` | Builds the Streamlit image on `python:3.12-slim`, bundling `app.py` + `artifacts/` |
| `requirements.txt` | Pinned runtime dependencies |
| `artifacts/xgboost_model.pkl` | Trained XGBoost classifier |
| `artifacts/encoder.pkl` | Fitted one-hot encoder for categorical features |
| `artifacts/scaler.pkl` | Fitted scaler for continuous features |
| `artifacts/model_metadata.json` | Feature names + column groupings (continuous / categorical / binary-ordinal) used to assemble the model input in the exact training order |
| `.gitignore` | Standard Python ignore rules |

> **Why are the `.pkl` artifacts committed?** This app is a *deployment artifact*, not a training repo. Bundling the model keeps the container build hermetic and lets the image run anywhere with zero external dependencies — the pattern we want for an ECS deployment in Module 5.

---

## How it works

1. **Load artifacts** (cached) — model, encoder, scaler, and metadata are loaded once at startup.
2. **Collect inputs** — the form gathers trip/route, driver/truck, and weather details.
3. **Preprocess** — continuous features are scaled, categorical features one-hot encoded, binary/ordinal features passed through; columns are then reordered to match `metadata['feature_names']` exactly.
4. **Predict** — outputs a binary delay decision plus the delay probability (%).

---

## Run locally (Python)

> Requires **Python 3.12.10**.

```bash
# 🪟 Windows PowerShell
python -m venv .venv
.venv\Scripts\Activate.ps1

# 🍎 macOS / 🐧 Linux
python3 -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
streamlit run app.py
```

Open http://localhost:8501 in your browser.

---

## Run with Docker

```bash
# Build the image
docker build -t truck-delay-predictor .

# Run the container
docker run -p 8501:8501 truck-delay-predictor
```

Open http://localhost:8501.

The image exposes port **8501** and ships a `HEALTHCHECK` that polls Streamlit's `/_stcore/health` endpoint — the same path used by the ALB target group health check in Module 5.

---

## Module 5 (Lab C) deployment path

This repo is the source for the ECS deployment lab:

```
Local build  →  docker build  →  push to ECR  →  ECS Fargate task  →  behind ALB (target group on :8501, health check /_stcore/health)
```

CI/CD (GitHub Actions) automates the build → push → deploy steps. See the Module 5 lab guide for the full walkthrough.

---

## Conventions

- **Python:** 3.12.10
- **Container base:** `python:3.12-slim`
- **Service port:** 8501
- **Health check path:** `/_stcore/health`
- **Code style:** functional (no classes)
