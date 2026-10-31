# ZEW 4.0 — Zombie Early Warning

**The Financial Lifespan of Zombie Companies: integrating machine learning and survival analysis on Indian listed firms.**

ZEW 4.0 is an end-to-end Python framework that identifies "zombie" companies (firms that cannot cover their interest bills from operating profit and survive only on forbearance or cheap credit), scores how severe their distress is, tracks how long they survive, and predicts which firms are likely to enter insolvency (IBC) within 24 months.

It covers the full chain: data panel → classification gate → severity score → state machine → published benchmarks → validation → survival analysis → machine learning → forensic accounting → charts and reports.

> Scope of the reference run: Nifty 500 + Nifty Microcap 250, FY2015–FY2026. Death event = IBC admission.

---

## Table of contents

- [Why this project](#why-this-project)
- [Framework at a glance](#framework-at-a-glance)
- [Repository structure](#repository-structure)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Input data](#input-data)
- [Outputs](#outputs)
- [Configuration](#configuration)
- [Methodology notes](#methodology-notes)
- [Testing](#testing)
- [Limitations](#limitations)
- [Author](#author)
- [License](#license)

---

## Why this project

Zombie firms tie up credit, depress productivity and hide stress inside balance sheets. Most published definitions use a single rule (for example, interest coverage below 1 for three years). ZEW 4.0 runs **seven definitions side by side**, adds a continuous 0–100 severity score and a fuzzy variant, and then asks three questions:

1. Who is a zombie, and how severe is the distress? (Layers 0–3)
2. How long does a zombie survive before insolvency? (survival analysis)
3. Can we predict insolvency early, and do zombies manipulate earnings? (ML and forensic analysis)

## Framework at a glance

| Layer | What it does |
|---|---|
| **Layer 0 — Universe** | Loads the firm-year panel, drops financial firms, requires at least 5 usable years, buckets small sectors into "Other", winsorises within sector-year |
| **Layer 1 — Gate** | Veto → condition A (persistent low coverage) → B/C (profitability / cash flow) → D (subsidy signal). Labels: Zombie, Subsidised Zombie, At Risk |
| **Layer 2 — Severity** | Robust sector-year z-scores → logistic 0–100 score, five root-cause pillars, non-compensatory override |
| **Layer 3 — States** | Persistence state machine (Healthy, Watch, At Risk, Zombie, Subsidised Zombie, Recovered), trajectories and transition matrix |
| **Layer 4 — Benchmarks** | Six published definitions (AAM18, BH20, CHK08, SKSW17, ACEE20, SST17) plus Altman Z″, Beneish M and Piotroski F |
| **Fuzzy variant** | Continuous membership scores μA, μB, μC, μZ as a robustness check |
| **Validation (V1–V6)** | Null universe, known-good firms, known-distressed firms, sector balance (χ²), definition overlap (Jaccard), parameter grids and bootstrap |
| **Survival** | Kaplan–Meier, log-rank, time-varying Cox, Shumway hazard, "death clock" (median, S(t), RMST) |
| **Machine learning** | 24-month probability of default: logit, XGBoost, XGBoost + SMOTE, out-of-time test, TreeSHAP |
| **Forensic** | Earnings quality by state, fixed-effects regressions, dose–response, event-time analysis |

## Repository structure

```
zew_analysis/
├── README.md
├── requirements.txt
├── ZEW_Analysis.ipynb          # whole project as a runnable notebook
├── zew_analysis/               # the Python package
│   ├── __init__.py
│   ├── __main__.py             # command-line entry point
│   ├── config.py               # every threshold and constant, in one place
│   ├── data.py                 # Layer 0: load and build the panel
│   ├── ratios.py               # derived variables (G1–G9, P1–P5, Altman/Beneish/Piotroski inputs)
│   ├── gate.py                 # Layer 1
│   ├── severity.py             # Layer 2
│   ├── states.py               # Layer 3
│   ├── benchmarks.py           # Layer 4
│   ├── fuzzy.py                # fuzzy variant
│   ├── validation.py           # V1–V6
│   ├── survival.py             # KM, log-rank, Cox, Shumway, death clock
│   ├── ml.py                   # PD models and SHAP
│   ├── forensic.py             # earnings-quality tests
│   ├── plots.py                # 36 charts (F01–F36)
│   ├── report.py               # Excel, CSV and HTML report
│   ├── synthetic.py            # null universe and demo data
│   └── run.py                  # runs everything end to end
└── tests/
    └── test_analysis.py        # 6 self-tests
```

## Installation

Requires Python 3.9 or later.

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Main dependencies: `pandas`, `numpy`, `scipy`, `scikit-learn`, `statsmodels`, `lifelines`, `xgboost`, `imbalanced-learn`, `matplotlib`, `openpyxl`.

## Quick start

**1. Try it on synthetic data first (about 1–2 minutes):**

```bash
python -m zew_analysis --demo
```

**2. Run on your real data:**

```bash
python -m zew_analysis --data zew_data --out zew_results
```

**Command-line options**

| Option | Default | Meaning |
|---|---|---|
| `--data` | `zew_data` | Folder with the data pipeline output |
| `--out` | `zew_results` | Folder where results are written |
| `--boot` | from `config.py` (200) | Bootstrap repetitions for V6; `0` skips it |
| `--demo` | off | Generate and analyse synthetic demo data |
| `--no-prices` | off | Skip weekly-price beta estimation |

**3. Or use it from Python:**

```python
from zew_analysis.run import run_all

R = run_all(data_dir="zew_data", results_dir="zew_results")
panel = R["panel"]            # firm-year table with labels, states, scores and flags
```

**4. Or use the notebook:** open `ZEW_Analysis.ipynb` and run it top to bottom. Set `USE_DEMO = True` in Part 2 to run on synthetic data. The notebook rebuilds the package modules itself and shows every chart inline.

## Input data

The analysis reads the output folder of the data-collection pipeline. The key file is:

| File | Content |
|---|---|
| `zew_data/09_abt_firm_year.csv` | Analytical base table, one row per firm per fiscal year (`ISIN_Root`, `FY`, revenue, EBIT, interest, CFO, total assets, total debt, event flags and so on) |

Optional price and index files enable the market-based features (beta, Merton distance-to-default). The raw data is **not** included in this repository. If the file is missing, the program stops with a message telling you to run the pipeline or use `--demo`.

## Outputs

Everything is written to `zew_results/`:

| Output | Description |
|---|---|
| `ZEW_report.html` | Self-contained report with all charts and headline numbers |
| `ZEW_results.xlsx` | Every result table, one sheet each, plus a Methodology sheet |
| `zew_firm_year_results.csv` | Label, state, scores and flags for every firm-year |
| `figures/F01…F36.png` | All 36 charts |

## Configuration

Every threshold lives in [`zew_analysis/config.py`](zew_analysis/config.py), marked either **(spec)** for values from the framework specification or **(impl)** for implementation choices. Examples:

| Setting | Default | Meaning |
|---|---|---|
| `ICR_CUT` | 1.0 | Interest-coverage break-even |
| `PERSIST_YEARS` | 3 | Years of persistence (AAM18) |
| `AGE_MIN` | 10 | Minimum firm age |
| `VETO_ICR`, `VETO_ND_EBITDA` | 3.0, 2.0 | Healthy-firm veto thresholds |
| `PILLAR_WEIGHTS` | 0.35 / 0.25 / 0.20 / 0.15 / 0.05 | Severity pillar weights |
| `PD_HORIZON_MONTHS` | 24 | Insolvency prediction horizon |
| `TRAIN_LAST_FY`, `TEST_FIRST_FY` | 2021, 2022 | Out-of-time split |
| `SEED` | 2026 | Random seed |

Paths can also be set with the environment variables `ZEW_OUT` (data folder) and `ZEW_RESULTS` (results folder).

## Methodology notes

- **Reportable classification.** The gate's "At Risk" label follows the specification literally and includes many capex-heavy growth firms with negative 3-year free cash flow. The **State** column from Layer 3 is the classification to report.
- **Calibration.** XGBoost uses class weights, so raw probabilities are inflated. Judge the models by ranking metrics (ROC-AUC, PR-AUC, top-decile capture), not by raw PD levels.
- **No immortal-time bias.** Survival spells start at the firm's first classifiable filing date, and a zombie spell starts at zombie entry.
- **Filing lag.** Financials are treated as public two months after year-end (SEBI LODR rule: audited results within 60 days) to avoid look-ahead bias.
- **Association, not causation.** Survival and forensic results describe associations only.
- **Demo data is synthetic.** Never report numbers from `--demo` runs.

## Testing

```bash
python tests/test_analysis.py
```

Six self-tests: null universe returns zero zombies, gate and veto rules, age filter, Cash-ICR extension, severity anchoring, and an end-to-end demo run.

## Limitations

- Depends on the quality and coverage of the upstream data pipeline.
- The IBC death events are relatively few, so survival and ML estimates carry wide confidence intervals.
- Sector-year benchmarks need enough firms per bucket (minimum 8; smaller sectors are merged into "Other").
- Results are specific to the Indian listed-firm universe and the chosen sample window.

## Team

Group project, MSc Business Analytics, Manipal Academy of Higher Education (MAHE).

| Name |
|---|---|---|
| Bhagath S G | 
| Aditya S |
| Manasa M |
| Apoorva Kulkarni |

Supervised by: _faculty name (Dr. Sathyabhan Sahoo)

# Corporate-Default-Prediction-using-Machine-Learning-Survival-Analysis
