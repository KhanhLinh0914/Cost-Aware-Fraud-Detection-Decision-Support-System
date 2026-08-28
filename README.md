# Cost-Aware Fraud Detection & Decision-Support System
 
## Overview
This is an **end-to-end capstone project** that builds a cost-aware fraud detection and decision-support framework for card-not-present (CNP) credit card transactions, using the IEEE-CIS Fraud Detection dataset (Vesta Corporation).

Instead of optimizing for conventional classification accuracy, the project frames fraud detection as a **financial optimization problem**, balancing detection performance against the real dollar cost of false positives and false negatives. The project follows the full lifecycle: business problem definition → IT architecture → solution design (Agile) → data cleaning → modeling & cost optimization → testing → final Figma/dashboard delivery.

**Team:**  Neha Kataria, Thi Khanh Linh Pham (Sylvia), Muskan, Ke Ping Lo, Karyn Denise Pang, Gull Qazi
 
## Business Requirements
1. Cost-aware fraud scoring  
2. Dollar-value threshold optimization
3. Deliver a Figma-prototyped dashboard exposing the analytical outputs as user-facing functions: Live Risk Feed, Detection Performance, geographic/device risk views, staffing support, and cost comparison so that stakeholders can act on results without touching the underlying model
 

## Data
- **Transaction table:** 590,540 records
- **Identity table:** 144,233 records
- Integrated via `TransactionID`; only ~24.4% of transactions matched an identity record
- Overall fraud rate: ~3.5% (class imbalance: 96.5% legitimate / 3.5% fraud)
- ~434 columns after merging, many anonymized (V-, D-, M-, ID-series)

## 1. IT Architecture

- Documented the **existing** bank fraud-analytics architecture: data sources (transaction, identity/device, customer DB, historical fraud logs, external email/IP risk APIs) → core processing (payment gateway, transaction authorization engine, rule-based fraud module) → data storage (SQL Server EDW, ETL logs/audit trails) → integration & connectivity (nightly ETL, ODBC/JDBC, TLS, API endpoints) → analytics & reporting (Power BI, analyst workstations)
- Designed how the **new FraudWatch solution fits into this architecture**: transaction/identity data processed in the existing Databricks environment → LightGBM model scores each transaction → predictions and risk classifications stored in the existing data environment → surfaced via Power BI / the Figma-based dashboard prototype
- Documented security & connectivity considerations (Port 1433/TLS for SQL Server, role-based access control, firewalls, API/ODBC-JDBC access)
<img width="1182" height="1002" alt="IT Architecture_updated (2)" src="https://github.com/user-attachments/assets/239c7142-b415-47ac-88d5-d4eed48527c9" />

## 2. Solution Design (Agile)
Delivered through an iterative Agile process, organized as descriptive → diagnostic → predictive → prescriptive analytics sprints, each following a consistent workflow: **analytics question → feature coding (notebook) → Figma wireframe → interactive dashboard**.
 
- **Descriptive**: fraud distribution, transaction-amount patterns, fraud by hour of day
- **Diagnostic**: fraud by geographic region (region 299: 28.3% fraud rate vs. 3.5% baseline) and by device (new devices: 21.4% fraud rate vs. 1.8% for returning devices)
- **Predictive**: Live Risk Feed (ranks transactions by fraud score, suggests Block/Manual Review/Monitor) and Detection Performance (model caught 85.3% of fraud cases — 2,644 of 3,100 — at 13.4% precision)
- **Prescriptive**: riskiest hours for staffing, safe blocking range vs. review-team capacity, and cost comparison across strategies

## 3. Implementation
### 3.1 Database
- Built in **MySQL** (`FraudDetectionDB2`) with a full ERD covering all fraud-detection feature tables
- Sprint-based build-out:
  1. **Database Environment Setup**:  schema design, primary/foreign keys, ERD validation
  2. **Data Loading Pipeline**: imported raw CSVs into MySQL, resolved delimiter/format issues, validated row counts and NULL distributions
  3. **Environment Integration & Pipeline Merging**: Python↔MySQL connection via SQLAlchemy, chunk-based extraction into pandas, merged on `TransactionID` into the unified modeling dataset
     
<img width="946" height="1149" alt="fraudDB2_ERD" src="https://github.com/user-attachments/assets/f23af65b-1aa7-4753-ba71-437d0542dc7f" />


### 3.2 Data Cleaning 
Three reusable, independently-tested validation functions, each validated via synthetic error injection:
- **Duplicate Guard**: removes fully duplicated transaction records
- **Error Guard**: removes rows with invalid business-critical values (e.g., negative/null amounts, out-of-range fraud labels)
- **Missing Value Guard**: sentinel-value imputation (`-999`) for the heavily-missing V/D/M feature blocks, chosen over statistical imputation so the tree-based model can treat missingness as an informative signal

### 3.3 Feature Engineering & Modeling
- Temporal, customer, device, and email-based behavioral features engineered from the cleaned dataset
- Chronological split: 70% train / 15% validation / 15% test (~413K / 88.5K / 88.5K)
- Three models trained and compared:

| Model | Validation AUC | Test AUC |
|---|---|---|
| **LightGBM (final)** | 0.9355 | 0.9071 |
| CatBoost | 0.9159 | 0.9016 |
| Neural Network | 0.8474 | 0.7182 |

### 3.4 Cost-Based Threshold Optimization
Applied a Bahnsen-style cost-sensitive framework (Bayes Minimum Risk) with alpha sensitivity analysis (1%/2%/5%) and a capacity-constrained scenario. Cost basis: Ca = $2.69 (alpha = 2% of mean transaction amount, $134.60).

| Strategy | Total Cost | Savings vs. Baseline | Fraud Recall |
|---|---|---|---|
| No model (baseline) | $469,608.50 | — | — |
| Default threshold (0.5) | $344,621 | 26.6% | 35.6% |
| **Cost-optimal fixed threshold (0.017)** | **$105,777.78** | **77.5%** | **85.3%** |
| Bayes Minimum Risk (per-transaction) | $86,267 | 81.6% | — |

The fixed threshold (0.017) was recommended for production over the per-transaction Bayes Minimum Risk approach due to greater simplicity, transparency, and auditability.

### 3.5 Testing
- **Master test suite: 40 test cases**   
- Validated that expected outputs are correctly represented across the Figma/dashboard application's analytical screens

### 3.6. Final Deliverable
Figma-prototyped & dashboard ("FraudWatch") built on top of the model and cost-analysis outputs, including:
- **Live Risk Feed**: transaction-level fraud score, risk level, and suggested action
- **Detection Performance**: recall/precision and transactions-under-review view
- **Geographic & device risk views**, **staffing/scheduling support**, and **cost-comparison view** (new system vs. traditional approach)

Figma prototype:  https://less-harp-55003755.figma.site/   Password: requires uppercase + lowercase + number + special character

## Key Challenges
- High dimensionality and largely anonymized features (~434 columns after merge)
- Structural missingness in identity data (only 24.4% of transactions had a matching identity record)
- Block-wise missingness across the V-series features
- Balancing data cleaning against preservation of genuine fraud signals (e.g., near-duplicate transactions can indicate card-testing rather than errors)
- Severe class imbalance (96.5% legitimate / 3.5% fraud)

## Future Enhancements (from report)
- Formal, validation-driven optimization of risk thresholds (rather than manually selected cut-offs)
- Transaction-amount-aware risk decisions (combining fraud probability with dollar exposure)
- Automated three-tier fraud-risk categories (low/medium/high)
- Real-time fraud scoring integrated into the live transaction pipeline

## Repository Structure
```
├── CAPSTONE.ipynb                                # Main notebook: cleaning, modeling, cost optimization
├── Capstone_Final_Report_Group_3.docx            # Full final report  
├── sql/                                          # MySQL schema / ERD for FraudDetectionDB2
└── README.md
```

## Dataset
This project uses the [IEEE-CIS Fraud Detection dataset](https://www.kaggle.com/c/ieee-fraud-detection) from Kaggle. Due to file size, the raw data is not included in this repository — download it directly from Kaggle to reproduce the notebook.

## References
1. Bahnsen, A. C., Stojanovic, A., Aouada, D., & Ottersten, B. (2013). Cost sensitive credit card fraud detection using Bayes minimum risk. *ICMLA 2013*.
2. Khalili, N., & Rastegar, M. A. (2023). Optimal cost-sensitive credit scoring using a new hybrid performance metric. *Expert Systems with Applications, 213*.
3. Vesta Corporation. (2019). *IEEE-CIS Fraud Detection* [Data set]. Kaggle.
