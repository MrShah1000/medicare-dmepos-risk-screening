# Medicare DMEPOS Risk Screening Pipeline

Statistical outlier screening for Medicare Durable Medical Equipment, Prosthetics, Orthotics, and Supplies (DMEPOS) billing patterns using public CMS data, peer-adjusted analytics, and a governance layer for flagging anomalies for review.

## Problem Statement

Durable medical equipment (DME) fraud and billing abuse cost Medicare billions annually. Identifying high-risk providers requires analyzing provider billing patterns against peer baselines — but the data is fragmented, updated irregularly, and non-risk-adjusted at the aggregate level. This project builds an end-to-end analytics pipeline to:

- Ingest multi-year Medicare DMEPOS utilization and payment data from public sources
- Profile providers against peer cohorts defined by specialty, geography, and service code
- Flag statistical outliers using robust z-scores and unsupervised ML
- Cross-reference flagged providers against HHS-OIG's List of Excluded Individuals and Entities (LEIE)
- Surface results through a self-service tool with transparent data governance and limitations

**Critical caveat:** This analysis identifies *statistical outliers requiring review*, not fraud findings. Public CMS files are aggregate, non-risk-adjusted, and cover Medicare FFS only. See [Data Limitations](#data-limitations) below.

## Data Sources

| Source | Description | Rows | Frequency |
|--------|-------------|------|-----------|
| **Medicare Physician & Other Practitioners PUF** | Part B utilization, payment, and charges by NPI, HCPCS, place of service | ~10M/year | Annual |
| **Medicare DMEPOS PUF** | DMEPOS claims by referring provider, HCPCS, and state | ~2M/year | Annual |
| **HHS-OIG LEIE** | Active exclusions (individuals and entities) | ~100K | Monthly |
| **NPPES NPI Registry** | Provider taxonomy, practice location, specialty | ~25M | Ongoing |
| **Medicaid State Drug Utilization Data** | Optional: state-level drug utilization for multi-program context | Varies | Quarterly |

All sources are public and freely downloadable.

## Data Limitations

- **No risk adjustment:** CMS files contain aggregate counts and payments, not individual claim details. Billing intensity differences due to case-mix, geography, or patient acuity are not isolated.
- **Medicare FFS only:** Analysis covers traditional Medicare, not Medicare Advantage or other payers.
- **Suppression rules:** CMS suppresses cells with ≤10 claims or ≤5 providers to protect privacy; flagged providers may reflect imputed or missing values.
- **LEIE fuzzy-matching:** The LEIE contains provider name and location, not NPI. Name/location matching produces false positives and false negatives; see [docs/leie_matching.md](docs/leie_matching.md) for methodology and accuracy.
- **Aggregate outliers:** Statistical flags identify provider-code-year combinations with unusual billing, not individual claims. Investigators must review actual claim-level data through official channels.

## Project Structure
medicare-dmepos-risk-screening/
├── README.md # This file
├── LICENSE # MIT license
├── pyproject.toml # Python project metadata and dependencies
├── .gitignore # Git exclusions
├── .github/
│ └── workflows/
│ └── tests.yml # CI/CD: pytest on PR
├── sql/ # Versioned SQL queries
│ ├── 01_peer_groups.sql
│ ├── 02_outlier_engine.sql
│ ├── 03_risk_signals.sql
│ └── README.md # Query documentation
├── src/ # Python modules (not notebooks)
│ ├── init.py
│ ├── ingest.py # Download and profile data sources
│ ├── scoring.py # Outlier scoring logic
│ ├── ml.py # Unsupervised/supervised models
│ └── tests/
│ └── test_scoring.py
├── notebooks/ # Exploratory analysis only
│ └── 01_eda.ipynb
├── data/
│ ├── raw/ # (gitignored) Raw downloaded files
│ ├── interim/ # (gitignored) Processed intermediate tables
│ └── .gitkeep
├── docs/
│ ├── data_profiles/ # Output from Phase 1 profiling
│ ├── query_optimization.md # Phase 2: before/after query timings
│ ├── data_dictionary.md # Phase 5: field-level definitions
│ ├── data_catalog.md # Phase 5: asset risk ranking
│ ├── lineage.md # Phase 5: source-to-score transformations
│ ├── data_use_statement.md # Phase 5: use & limitations for end users
│ ├── model_card.md # Phase 4: ML model assumptions & limits
│ ├── leie_matching.md # Fuzzy matching methodology
│ └── README.md # Documentation index
└── app/ # Phase 6: Streamlit self-service tool
└── app.py

## Quick Start

### Prerequisites
- Python 3.10+
- DuckDB (local dev) or Snowflake (Phase 3+)
- Git

### Installation

```bash
git clone https://github.com/MrShah1000/medicare-dmepos-risk-screening.git
cd medicare-dmepos-risk-screening

python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

pip install -e .
```

### Running the pipeline (Phase 1+)

```bash
# Download and profile data
python src/ingest.py --years 2020 2021 2022 2023

# Run SQL analytics (DuckDB)
duckdb < sql/01_peer_groups.sql

# Score outliers
python src/scoring.py --output data/interim/flags.parquet

# Launch dashboard (Phase 6+)
streamlit run app/app.py
```

See [docs/README.md](docs/README.md) for detailed phase-by-phase instructions.

## Key Features (By Phase)

### Phase 1: Data Ingestion & Profiling
- Reproducible multi-year downloads from CMS, NPPES, LEIE
- Data quality profiling: null rates, cardinality, join-key integrity
- Documented data defects and handling decisions

### Phase 2: SQL Analytics
- Peer cohorts defined by specialty × geography × HCPCS code
- Robust outlier detection using window functions and median absolute deviation
- Multi-signal risk scoring (utilization ratios, concentration, volume jumps)
- Query optimization documentation

### Phase 3: Snowflake & Snowsight
- Production-grade SQL dialect and micro-partition optimization
- RBAC and column-level masking for governance
- Interactive dashboard for exploratory analysis

### Phase 4: Python & Machine Learning
- Unsupervised outlier detection (Isolation Forest)
- Supervised classification (gradient boosting on LEIE matches)
- Precision/recall evaluation and SHAP explainability
- Model card documenting intended use and limitations

### Phase 5: Data Governance
- Data dictionary and catalog with risk ranking
- End-to-end lineage documentation
- Data use statement and access control model

### Phase 6: Self-Service Tool
- Streamlit app for filtering and drilling into flagged providers
- Filter correctness tests
- Non-technical user interface

## Development

### Running Tests

```bash
pytest src/tests/
```

Tests run automatically on every PR via GitHub Actions (see `.github/workflows/tests.yml`).

### Git Workflow

All work happens on feature branches and merges via pull request, even for solo development:

```bash
git checkout -b feature/phase-1-ingestion
# ... make changes, commit
git push origin feature/phase-1-ingestion
# Open PR, review, merge via GitHub UI
```

This documents your ability to work with Git version control in a collaborative environment — the commit graph is your evidence.

## Interview Notes

This project demonstrates the specialized experience required for the HHS-OIG IT Specialist (DATAMGT) role:

- **Analyzing healthcare data:** Medicare DMEPOS utilization and payment patterns, peer-adjusted anomaly detection
- **Querying large-scale databases:** Multi-year CMS files (~10M rows/year) with SQL window functions and performance optimization
- **Python scripting + Git:** Data ingestion, scoring, and ML modules with version control and CI/CD
- **Presenting to leadership:** Five-minute brief on findings, methodology, and limitations

See [docs/interview_prep.md](docs/interview_prep.md) for STAR stories and cold-question prep.

## License

MIT. See [LICENSE](LICENSE) for details.

## Contact

Questions? Open an issue or reach out at suhaibshah@hotmail.com.

---

**Last updated:** [today's date]  
**Status:** Phase [X] in progress
