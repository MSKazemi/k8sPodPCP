# Project Overview

This project is a machine learning pipeline for predicting the power consumption of Kubernetes pods. It uses a combination of Python scripts to collect data from a Kubernetes cluster, process it, train a model, and make predictions.

The main technologies used are:
- Python
- Kubernetes API
- Prometheus
- Kepler
- Pandas
- Scikit-learn (KNeighborsRegressor)
- Sentence-Transformers (for text embeddings)

The project is structured as a pipeline of scripts that pass data to each other via files.

## Building and Running

### 1. Installation

Install the required Python packages:

```bash
pip install -r requirments.txt
```

### 2. Data Collection and Training

The following scripts are run in sequence to train a power prediction model:

1.  **`k8s_collect.py`**: Collects workload specifications from the Kubernetes API.

    ```bash
    python k8s_collect.py watch --kinds Deployment Job CronJob --emit-initial > ./data/inference_requests.ndjson
    ```

2.  **`k8s_encode.py`**: Encodes the collected data into feature vectors. First, fit an encoder:

    ```bash
    python k8s_encode.py fit --input ./data/inference_requests.ndjson --out ./artifacts/encoder.joblib
    ```

    Then, transform the data:

    ```bash
    python k8s_encode.py transform --input ./data/inference_requests.ndjson --encoder ./artifacts/encoder.joblib --out ./data/features.parquet
    ```

3.  **`kepler_labels.py`**: Fetches power consumption labels from Prometheus/Kepler.

    ```bash
    python kepler_labels.py \
      --prom <your-prometheus-url> \
      --start $(date -u -d '6 hours ago' +%s) \
      --end   $(date -u +%s) \
      --out ./data/kepler_labels.parquet
    ```

4.  **`join_features_labels.py`**: Joins the features and labels to create a training set.

    ```bash
    python join_features_labels.py \
      --features ./data/features.parquet \
      --labels   ./data/kepler_labels.parquet \
      --out      ./data/train_rows.parquet
    ```

5.  **`train_power.py`**: Trains a KNN model on the training data.

    ```bash
    python train_power.py \
      --train ./data/train_rows.parquet \
      --target avg_power_w \
      --out ./artifacts/knn_power.joblib
    ```

### 3. Prediction

To make predictions on new workloads:

```bash
python predict_k8s.py \
  --encoder ./artifacts/encoder.joblib \
  --model   ./artifacts/knn_power.joblib \
  --input   ./data/inference_requests.ndjson
```

## Development Conventions

- The project uses a virtual environment for dependency management (see the `.venv` directory).
- The machine learning pipeline is modular, with each script performing a specific task.
- Data is passed between scripts using files (NDJSON and Parquet formats).
- Trained models and encoders are saved as joblib files in the `artifacts` directory.
