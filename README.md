# Dynamic Fare Estimation for NYC Yellow & Green Taxis 🚕💸

This project builds a **cloud-hosted fare estimator** for NYC yellow and green taxis focused on **Manhattan**.  

Users can select **pickup/dropoff zones**, **date & time**, and **number of passengers** and get an **estimated taxi fare** before taking the trip – similar to modern ride-hailing apps.

Under the hood:

- Historical **NYC TLC trip data** is ingested and stored in **PostgreSQL**
- A **LightGBM regression model** is trained on engineered features
- The trained model is stored in **DigitalOcean Spaces** (S3-compatible)
- A **FastAPI web app**:
  - Uses **Mapbox Directions** to compute realistic driving distance
  - Builds the same features used in training
  - Predicts the fare and displays it via a simple web UI
  - Caches repeated predictions in Postgres

---

## 🔎 Table of Contents

1. [Architecture Overview](#-architecture-overview)
2. [Repository Structure](#-repository-structure)
3. [Data Sources](#-data-sources)
4. [Tech Stack](#-tech-stack)
5. [Setup & Installation](#-setup--installation)
6. [Environment Variables](#-environment-variables)
7. [Step 1 – Ingest TLC Data to Postgres](#step-1--ingest-tlc-data-to-postgres)
8. [Step 2 – Train the LightGBM Model](#step-2--train-the-lightgbm-model)
9. [Step 3 – Run the Web App](#step-3--run-the-web-app)
10. [Optional – Run with Docker](#optional--run-with-docker)
11. [Model Details](#-model-details)
12. [Database Tables](#-database-tables)
13. [Limitations & Future Work](#-limitations--future-work)
14. [License](#-license)

---

## 🏗 Architecture Overview

High-level flow:

1. **Data ingestion (`taxi_ingest.py`)**
   - Downloads monthly NYC TLC **yellow** and **green** trip Parquet files
   - Cleans & filters trips
   - Engineers **temporal** and **spatial** features
   - Stores features in a Postgres table: `taxi_trips_manhattan_ml`

2. **Model training (`train_model.py`)**
   - Loads features from Postgres
   - Trains a **LightGBM** regression model on `log1p(total_amount)`
   - Evaluates metrics (RMSE, MAE, R²)
   - Saves model as `.joblib` and uploads it to **DigitalOcean Spaces**
   - Logs metrics and model location to `taxi_model_metrics` in Postgres

3. **Model serving (`app.py` + `model_loader.py` + `preprocess.py`)**
   - On startup, downloads the latest model from Spaces
   - Loads **Manhattan zone centroids** from `manhattan_zone_lookup.csv`
   - Exposes a **FastAPI** endpoint `/` with a form-based UI (via `templates/index.html`)
   - When a user submits a request:
     - Mapbox Directions API is used to compute realistic **driving distance**
     - Features are built to match the training pipeline
     - A fare is predicted by the model (and cached in Postgres)

4. **Caching (`fare_prediction_cache`)**
   - Feature combinations are used as a composite key
   - Predictions are cached in Postgres to avoid recomputing for identical queries

---

## 📁 Repository Structure

> Exact layout may vary slightly, but these are the main components.

```text
MIS547_Group3_project/
├── app.py                     # Main FastAPI application with caching + DB
├── app-Copy1.py               # Simpler FastAPI version (no DB/cache)
├── taxi_ingest.py             # Ingest TLC data → Postgres feature table
├── train_model.py             # Train LightGBM model and upload to Spaces
├── preprocess.py              # Online preprocessing + Mapbox distance
├── model_loader.py            # Download model from Spaces and load via joblib
├── model_metadata.json        # High-level metadata about the model
├── manhattan_zone_lookup.csv  # TLC Manhattan zones + centroids (lat/lon)
├── requirements.txt           # Python dependencies
├── style.css                  # Stylesheet (used by the HTML template)
├── Dockerfile                 # Container image for deployment
├── .gitignore
└── (expected)
    ├── templates/
    │   └── index.html         # Jinja2 HTML template for the UI
    └── static/
        └── style.css          # CSS served by FastAPI static files
