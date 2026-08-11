# Telecom Site Health & Predictive Maintenance

> A portfolio demonstration of a predictive-maintenance system developed during an industry data science fellowship.

This project explores how **environmental sensor readings, SCADA alarms, and infrastructure event logs** can be combined to detect abnormal operating conditions, identify persistent faults, and continuously estimate the health of a telecommunications site.

Telecom infrastructure depends on supporting systems such as **air conditioning, UPS units, batteries, and power systems**. Traditional maintenance is often reactive: an alarm occurs, a technician investigates, and the infrastructure may already have been operating under abnormal conditions for some time.

We investigated a different approach:

**Can operational data help identify abnormal behaviour earlier and distinguish meaningful faults from transient events?**

The original project was developed using confidential operational data during an industry fellowship. **No proprietary data is included in this repository.** The public version uses synthetic/sanitized data and a reconstructed implementation to demonstrate the methodology.

---

## Project Overview

The pipeline combines three main sources of operational information:

```text
Environmental Sensors
Temperature / Humidity
        │
        ▼
Environmental Anomaly Detection
        │
        │
SCADA Alarm Logs ──────► Alarm Episode Analysis
        │                       │
        │                       ▼
Infrastructure Events ─► Fault Behaviour Analysis
        │
        └───────────────────────┐
                                ▼
                         Site Health Score
                                │
                                ▼
                     Future Health Forecast
                                │
                                ▼
                    Maintenance Decision Support
```

The project consists of four main components:

1. **Exploratory Data Analysis**
2. **Alarm & Fault Anomaly Detection**
3. **Environmental Anomaly Detection**
4. **Site Health Scoring & Forecasting**

---

## Repository Structure

```text
SCADAFaultPrediction/
│
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_alarm_anomaly_detection.ipynb
│   ├── 03_environmental_anomaly_detection.ipynb
│   └── 04_site_health_forecasting.ipynb
│
├── src/
│   ├── preprocessing.py
│   ├── feature_engineering.py
│   ├── anomaly_detection.py
│   └── health_scoring.py
│
├── data/
│   └── synthetic/
│
├── results/
│
├── requirements.txt
└── README.md
```

The notebooks are designed to explain both **what is being done and why**, with emphasis on the reasoning behind preprocessing, feature engineering, model selection, and evaluation.

---

# 1. Exploratory Data Analysis

The first stage investigates the quality and behaviour of operational telemetry.

### Environmental telemetry

The environmental feed contains measurements such as:

* temperature
* humidity
* timestamps
* sensor/site identifiers

The analysis investigates:

* missing measurements
* corrupted timestamps
* duplicate observations
* irregular sampling intervals
* communication gaps
* extreme environmental conditions
* sudden temperature/humidity changes

A key design decision is to avoid automatically interpolating long communication gaps.

A missing observation is not equivalent to a healthy observation.

### Alarm and infrastructure logs

Alarm logs contain timestamped events representing infrastructure state changes.

Examples in the synthetic dataset include:

```text
UPS BATTERY WARNING
AC HIGH TEMPERATURE
UTILITY POWER FAILURE
UPS ON BATTERY
GENERATOR START
ALARM CLEARED
```

Messages are mapped into infrastructure categories such as:

```text
UPS
BATTERY
COOLING
ENERGY
OTHER
```

This creates a structured representation that can be used for downstream anomaly detection and health scoring.

---

# 2. Alarm Anomaly Detection

Individual alarm rows do not always represent independent failures.

Operational systems may generate repeated events, rapid state changes, or multiple alarms caused by the same underlying incident.

The pipeline therefore converts raw events into **alarm behaviour and episodes**.

### Alarm frequency anomalies

Daily and hourly alarm volumes are monitored using robust statistical techniques including:

* rolling statistics
* z-scores
* Median Absolute Deviation (MAD)
* robust thresholding

These methods identify periods where alarm activity differs substantially from normal site behaviour.

### Flapping detection

An alarm may repeatedly activate and clear over a short period.

For example:

```text
10:00  ACTIVE
10:02  CLEARED
10:03  ACTIVE
10:05  CLEARED
10:07  ACTIVE
```

Rather than interpreting these as unrelated failures, the pipeline groups rapid retriggers into **flapping episodes**.

Useful features include:

```text
time_since_previous_alarm
episode_duration
number_of_retriggers
alarm_frequency
time_since_last_clear
subsystem
state_transition
```

### Alarm duration

Where activation and clearing events can be paired, the pipeline estimates how long a fault remains active.

This is important because:

> An alarm that clears almost immediately is operationally different from one that remains active for hours.

Persistence therefore becomes an important signal in later health scoring and predictive-maintenance models.

### Event-level anomaly detection

Two unsupervised approaches are demonstrated:

**Isolation Forest**

Detects events with unusual combinations of temporal, frequency, and episode-level features.

**Local Outlier Factor (LOF)**

Identifies observations whose local neighbourhood differs substantially from surrounding events.

Detected events can additionally receive:

* anomaly explanation
* severity category
* subsystem
* episode context

---

# 3. Environmental Anomaly Detection

Environmental conditions are analysed independently from alarm behaviour.

The pipeline looks for several types of abnormal behaviour:

### Threshold anomalies

Examples include:

* unusually high temperature
* unusually high/low humidity
* sudden environmental changes

Thresholds are estimated using the **training period only**.

This avoids leaking information from future observations into anomaly detection.

### Communication gaps

Large gaps between consecutive sensor readings break the time series into **continuous segments**.

Rolling statistics and episodes are never allowed to cross these gaps.

This prevents measurements separated by a long period of missing telemetry from being incorrectly treated as adjacent observations.

### Hysteresis episodes

Environmental anomalies often oscillate around a threshold.

Instead of generating many tiny episodes, hysteresis is used:

```text
Enter anomaly → threshold exceeded

Remain anomalous → while conditions remain elevated

Exit anomaly → only after several stable observations
```

This creates more realistic operational episodes.

### Change-point detection

Change-point algorithms are used to identify shifts in the statistical behaviour of the environment.

This helps detect situations where the site's environmental regime changes even when individual measurements are not extreme.

### Hidden Markov Model

A Gaussian Hidden Markov Model is explored for identifying latent environmental states such as:

```text
Normal
Elevated
Abnormal
```

Multiple random seeds are evaluated to reduce sensitivity to initialization.

Model selection is performed using validation data before final evaluation on an untouched test period.

### Isolation Forest

Isolation Forest is also evaluated using the same features and chronological splits, allowing a fair comparison with the state-based approach.

---

# 4. Site Health Score

The anomaly pipelines feed into a continuous **0–100 site health score**.

Instead of treating every alarm equally, the system combines multiple dimensions of operational risk.

Example subsystem scores include:

```text
Environmental Health
Energy Health
Battery / UPS Health
```

These are combined into an overall site health estimate.

```text
Environmental Risk ──┐
                     │
Energy Risk ─────────┼──► Weighted Risk ──► Health Score
                     │
Battery Risk ────────┘

Health = 100 - Risk
```

---

## Recency-Weighted Fault Burden

Recent events contribute more strongly than older events.

A decay function is used conceptually as:

```text
event_weight = exp(-λ × age)
```

This allows the health score to recover gradually after faults disappear rather than changing abruptly.

---

## Fault Persistence

Persistence receives additional importance.

The system distinguishes between:

```text
Alarm activates → clears quickly
```

and:

```text
Alarm activates ───────────────────► remains active
```

The second situation represents substantially greater operational risk even if both generated the same number of alarm messages.

---

## Configurable Weights

Health-score parameters are stored centrally so different assumptions can be tested.

For example:

```python
WEIGHTS = {
    "environment": ...,
    "energy": ...,
    "battery": ...
}
```

Weights are validated and versioned.

If a particular signal is unavailable, its contribution can be removed and redistributed rather than automatically making the site appear healthier.

---

## Smooth Risk Saturation

Risk terms use smooth saturation rather than simple clipping.

This preserves ordering between:

* mildly abnormal conditions
* severe conditions
* extremely severe conditions

even after a risk threshold has been crossed.

---

# 5. Health Forecasting

The final stage investigates whether current operational behaviour can predict **future site health**.

The target is the site's health score several hours into the future.

Because infrastructure health often changes slowly, a simple persistence model is used as the baseline:

```text
future_health ≈ current_health
```

A predictive model therefore needs to outperform a surprisingly strong baseline.

---

## Feature Engineering

Features include combinations of:

### Current state

```text
current_health
environmental_risk
energy_risk
battery_risk
```

### Lag features

```text
health_t-1
health_t-3
health_t-6
...
```

### Dynamic behaviour

```text
health_change
rolling_mean
rolling_std
alarm_rate
recent_anomaly_count
fault_persistence
time_since_last_fault
```

### Episode behaviour

```text
active_episode_count
episode_duration
flapping_frequency
subsystem_activity
```

These features allow models to learn not only the current state of the site but also **how quickly that state is changing**.

---

## Models Explored

Several approaches are compared, including:

* Persistence baseline
* AutoReg
* Isolation Forest
* Hidden Markov Models
* XGBoost
* LightGBM
* Hybrid models
* Classification models for major health deterioration

Hyperparameter optimization can be performed using Optuna.

---

# Preventing Data Leakage

Because this is a time-series problem, random train/test splitting would produce unrealistic results.

All modelling follows chronological splits:

```text
TRAIN ──────────► VALIDATION ──────────► TEST
past                                      future
```

The pipeline follows several rules:

* chronological train/validation/test splits
* preprocessing fitted on training data only
* training-period imputation statistics
* anomaly thresholds estimated from training only
* hyperparameters selected using validation data
* final test data evaluated once

This attempts to reproduce the information that would actually have been available at prediction time.

---

# Maintenance Decision Support

The final goal is not simply to produce an anomaly score.

Predictions are translated into operationally interpretable outputs such as:

```text
Health Score
Status
Primary Risk Driver
Maintenance Priority
Recommended Action
```

Conceptually:

```text
SCADA + Sensors + Infrastructure Logs
                │
                ▼
        Behaviour Detection
                │
                ▼
         Risk Estimation
                │
                ▼
          Health Scoring
                │
                ▼
        Future Health Risk
                │
                ▼
       Maintenance Decision
```

The broader objective is to move from:

**reactive maintenance**

toward:

**data-driven preventive maintenance.**

---

# Tech Stack

**Data & Analysis**

* Python
* Pandas
* NumPy
* Jupyter
* Apache Spark / Scala

**Machine Learning**

* Scikit-learn
* Isolation Forest
* Local Outlier Factor
* Hidden Markov Models
* XGBoost
* LightGBM
* Statsmodels
* Optuna

**Visualization**

* Matplotlib
* ipywidgets

**Engineering**

* Git / GitHub
* modular preprocessing
* reproducible chronological evaluation
* configurable and versioned scoring parameters

---

# Running the Demo

### Requirements

Python 3.11+ and Jupyter.

```bash
pip install pandas numpy matplotlib scikit-learn statsmodels hmmlearn xgboost lightgbm optuna ipywidgets
```

Clone the repository and launch:

```bash
jupyter lab notebooks/
```

Suggested order:

```text
01_eda.ipynb
      ↓
02_alarm_anomaly_detection.ipynb
      ↓
03_environmental_anomaly_detection.ipynb
      ↓
04_site_health_forecasting.ipynb
```

The included synthetic dataset is designed to reproduce the **structure and modelling challenges** of the original problem without reproducing confidential operational information.

---

# Data Confidentiality

⚠️ **Important**

This repository is a **portfolio reconstruction** of work completed as part of an industry data science fellowship.

The original project used proprietary telecommunications infrastructure data.

For confidentiality and intellectual-property protection:

* original operational datasets are **not included**
* real infrastructure identifiers have been removed
* exact operational incidents are not reproduced
* proprietary internal documentation is not included
* the demonstration data is synthetic/sanitized
* public code is intended to demonstrate the methodology rather than reproduce the organization's production environment

Any examples in this repository are therefore illustrative and should not be interpreted as real operational records.

---

# Key Takeaways

This project demonstrates how heterogeneous infrastructure data can be transformed into an end-to-end predictive-maintenance workflow:

**Raw telemetry → data-quality analysis → event modelling → anomaly detection → health scoring → forecasting → maintenance decision support**

More broadly, the project explores an important lesson in predictive maintenance:

> **Not every alarm is a failure, and not every abnormal condition deserves the same operational response.**

Combining alarm persistence, environmental behaviour, infrastructure state, recency and anomaly patterns can provide a much richer picture of site health than raw alarm counts alone.
