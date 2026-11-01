# Congestion-Length-Prediction-on-National-Freeway-No.1

Predicting the **congestion length** caused by traffic accidents on National Freeway No. 1 (10–99K), so that traffic control units can estimate the affected range at the moment an accident is reported.

> **Merit Award** — 2024 (ROC 113) National Freeway Intelligent Traffic Management Creative Competition, Ministry of Transportation and Communications, Taiwan.

## Introduction

Me and my teammates participate in a competition launched by Taiwanese government and get merit award in the end. This study employs machine learning techniques to predict the congestion length after an accident, aiming to reduce prediction errors and the underestimation rate.

### The problem

The target variable is **zero-inflated with a long tail**: most accidents cause no congestion at all (length = 0), while a small number cause very long queues. A single regression model is pulled toward zero by the majority class and performs poorly, which is why the problem is split into two stages.

## Methodology

```
Accident event
   │
   ├─ Stage 1 (classification): will this accident cause congestion?
   │        │
   │        ├─ predicted "no congestion" ──→ congestion length = 0
   │        │
   │        └─ predicted "congestion"
   │                 │
   └────────────────→ Stage 2 (regression): how long is the queue?
                             (predicted on log-transformed length)
```

### Stage 1 — Will congestion occur? (binary classification)

- **Candidates compared**: Logistic Regression, SVM, KNN, XGBoost, AdaBoost, CatBoost, Random Forest
- **Selected**: **CatBoost** — the dataset contains categorical features (`Direction`, `DateCategory`, `TimeCategory`) that CatBoost handles natively, and it scored best on accuracy and F1
- **Hyperparameters**: grid search over `iterations` and `depth`, inspected with a heatmap → `iterations=950`, `depth=4`
- **Asymmetric error cost**: a false negative (congestion occurs but is predicted as none) leads traffic control to under-deploy resources, which is worse than a false alarm. The model is therefore evaluated with **F2-score** (β=2, weighted toward recall) rather than accuracy alone, with class weights `{congestion: 0.87, no congestion: 0.13}`
- **Interpretability**: **SHAP** feature importance ranks, in order — average speed before the event, whether the accident is on a ramp, time of day, incident handling duration, whether it is in the inner lane, and the mileage marker

### Stage 2 — How long is the queue? (regression)

- **Candidates compared**: MLP, Linear Regression, Random Forest, SVR
- **Selected**: **MLP** (Keras Sequential, 64 → 32 → 32 → 1, with Dropout 0.3 / 0.3 / 0.2, trained for 100 epochs)
- **Preprocessing**: features standardized with `StandardScaler`; the target is **log-transformed** to compress the long tail

### Evaluation

The two stages are chained into a single pipeline and evaluated on the test set with:

- **MSE**
- **Underestimation rate** — the proportion of cases where the predicted length is shorter than the actual length

Underestimation is tracked separately because, for traffic control, underestimating a queue leads to insufficient response, which carries a higher operational cost than overestimating it. This makes the metric a closer match to the real use case than MSE alone.

## Feature Engineering

Four groups of features, built in `Extract_features/Extract_features_code.ipynb`:

| Group | Features | How they were derived |
|---|---|---|
| Weather | temperature, wind speed, humidity, pressure at the time of the accident | scraped with `requests` + `BeautifulSoup` + `selenium` |
| Traffic before | traffic volume and heavy-vehicle ratio in the 10 minutes before the accident | M03A |
| Traffic after | traffic volume in the 2 minutes after the accident | M03A, using a **weighted conversion** to handle the mismatch between the 5-minute aggregation and the accident timestamp (e.g. for an accident at minute 32, the 30–35 interval volume is scaled by 2/5) |
| Speed before | average speed in the 10 minutes before the accident | M06A, via a custom `M06_findspeed()` that locates the relevant gantry interval from the accident time and mileage |

M03A is aggregated in 5-minute windows while accidents occur at arbitrary timestamps inside those windows, so the post-accident volume is reconstructed by linear weighting.

## Data Source

This study utilizes weather factors from the [Time and Date website](https://www.timeanddate.com/), including temperature (°C), wind speed (km/h), humidity (%), and pressure (mbar). Additionally, it incorporates data from the [Taiwan Freeway Bureau database](https://freeway2024.tw/links#links), including Etag static data (v2.0), M03A (vehicle traffic statistics), M06A (trip path data), roadwork sections (Jan–Oct 2023 & Jan–Feb 2024), National Freeway A1, A2, A3 accident records (Jan–Oct 2023 & Jan–Feb 2024), and traffic incident reports. Historical files are published at the [TISV Cloud history list](https://tisvcloud.freeway.gov.tw/history-list.php).

## Repository Structure

```
Extract_features/
├── Extract_features_code.ipynb   # feature engineering: scraping + M03A/M06A processing
└── data/
    ├── original_data.xlsx         # raw accident records
    ├── 事故前+事故後_v2.xlsx       # accident records joined with traffic features
    ├── 新-測站資料.xlsx            # M06A gantry/station reference
    └── 門架總資訊.csv              # M03A gantry reference

Model_Training/
├── Model_Training_code.ipynb      # two-stage training and evaluation
├── data/Data_final.xlsx           # final modeling dataset
├── catboost_info/                 # CatBoost training logs
└── pic/flow_chart.png             # pipeline diagram

Project/
├── Essay_Chinese version.pdf      # paper (Chinese)
├── Essay_English version.pdf      # paper (English)
└── Presentation_Chinese version.pdf  # competition slides
```

## How to Run

Python 3.x. Main packages:

```
pandas, numpy, scikit-learn, catboost, xgboost, tensorflow (keras),
shap, matplotlib, seaborn, requests, beautifulsoup4, selenium, openpyxl
```

Run `Extract_features/Extract_features_code.ipynb` first, then `Model_Training/Model_Training_code.ipynb`.

> Note: the notebooks contain absolute local paths from development. Adjust them to relative paths before reproducing.

## Team

This project was a team effort.

<!-- Bo-Ye's scope is inferred from the 【Writer】 annotation in the notebooks; please confirm with the team and fill in any remaining members. -->

| Member | Scope |
|---|---|
| Wei-Jia Lin (林瑋珈) | Feature engineering and data processing (M03A traffic volume, M06A average speed, time-window alignment, weather data collection); Stage 2 regression model (MLP); paper writing and competition presentation |
| Bo-Ye Li (李博業) | Stage 1 classification model (model selection, tuning, class weighting, SHAP analysis); code integration |
| 〔other members〕 | 〔to be filled〕 |

## Data and Licensing

- M03A / M06A traffic data: [Taiwan Freeway Bureau](https://tisvcloud.freeway.gov.tw/) open data, under the Government Open Data License
- Accident records: provided by the competition organizer
- Weather data: scraped from a public website, used for academic research only

Code in this repository is shared for academic reference and portfolio purposes.
