# MLOps Batch Signal Processing Pipeline (Task 0 Assessment)

A deterministic, observable, and containerized Python batch job simulating production quantitative signal pipelines (e.g., trading-signal calculations in MetaStackerBandit).

---

## Technical Features & Architecture

- **Reproducibility**: Enforces global random seeding (`np.random.seed(seed)`) sourced directly from validated YAML configuration.
- **Observability**: Dual-channel logging via Python's standard `logging` module. Execution logs stream to `sys.stderr` and write to `--log-file`. Machine-readable structured outputs stream to `sys.stdout` and write to `--output` (`metrics.json`).
- **Deployment Readiness**: Lightweight, containerized packaging via Docker using `python:3.9-slim`. Zero hardcoded paths; flexible CLI parameters with production defaults.
- **Robust Error Handling**: Exception boundaries trap invalid configurations, missing files, corrupted CSV schemas, or missing columns, producing a guaranteed error payload JSON with exit code `1`.

---

## Signal Generation & Rolling Window Conventions

1. **Rolling Mean**: Calculated on the `close` column using a rolling window of size `window` specified in `config.yaml`.
2. **Binary Signal**: 
   $$\text{signal}_i = \begin{cases} 1 & \text{if } \text{close}_i > \text{rolling\_mean}_i \\ 0 & \text{otherwise} \end{cases}$$
3. **Initial `window - 1` Rows**: For the first `window - 1` rows where `rolling_mean` is `NaN`, evaluating `close > NaN` resolves to `False` (`0`). This produces a complete binary series of length equal to `rows_processed` without breaking determinism or mean calculations.

---

## File Structure

```text
.
├── config.yaml          # YAML configuration (seed, window, version)
├── data.csv             # Input OHLCV dataset (10,000 rows)
├── Dockerfile           # Multi-stage/Slim Docker image manifest
├── metrics.json         # Output metrics artifact (JSON)
├── README.md            # Pipeline documentation & instructions
├── requirements.txt     # Python dependency specifications
├── run.log              # Detailed execution logs
└── run.py               # Main CLI execution entrypoint
```

---

## Prerequisites & Installation

### Local Python Environment

Python 3.9+ is required. Install dependencies via `pip`:

```bash
pip install -r requirements.txt
```

---

## Usage Instructions

### 1. Local Execution

Run the batch job using the required CLI signature:

```bash
python run.py --input data.csv --config config.yaml --output metrics.json --log-file run.log
```

Alternatively, default CLI options allow execution without flags:

```bash
python run.py
```

### 2. Docker Container Execution

Build the Docker image:

```bash
docker build -t mlops-task .
```

Run the containerized batch job:

```bash
docker run --rm mlops-task
```

---

## Output Specifications

### Success Output (`metrics.json`)

On successful execution (exit code `0`), `metrics.json` is generated and printed to `stdout`:

```json
{
  "version": "v1",
  "rows_processed": 10000,
  "metric": "signal_rate",
  "value": 0.4989,
  "latency_ms": 31,
  "seed": 42,
  "status": "success"
}
```

### Error Output (`metrics.json`)

On failure (exit code `1`), the error payload is written to `metrics.json` and printed to `stdout`:

```json
{
  "version": "v1",
  "status": "error",
  "error_message": "Required column 'close' missing from dataset"
}
```

---

## Error Handling & Edge Cases

The application handles the following edge cases cleanly:
- **Missing Input / Config Files**: Trapped with `FileNotFoundError`, logging the missing path.
- **Malformed YAML / Empty Config**: Validated prior to execution; missing keys (`seed`, `window`, `version`) or invalid types raise descriptive errors.
- **Empty / Corrupted CSV Data**: Verified using `os.path.getsize` and DataFrame validation. Missing `close` or non-numeric values trigger structured error metrics.
