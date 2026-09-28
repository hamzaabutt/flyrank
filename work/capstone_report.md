# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** FlyRank ML Intern
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/hamzaabutt/flyrank
- **Date:** September 2026

## 0. Abstract

How do SEO content teams decide which decaying articles are worth a $300–$500 editorial refresh? Using the FlyRank search intelligence dataset (30,000 anonymized URLs), we evaluate a transparent rule baseline against machine learning models to predict organic traffic decay (`is_declining_label`) and rank update priorities. Features were constructed strictly prior to label observation, and models were evaluated on a client-aware holdout split (20% unseen client domains). Our Random Forest classifier achieves a **74.0% Precision@50** and **0.750 ROC-AUC**, delivering a **3.08x lift over the deterministic rule baseline** (24.0% Precision@50). The output is deployed as an automated action engine mapping ranked URLs to concrete editorial playbooks. Built on the FlyRank ML Internship dataset (https://flyrank.ai).

## 1. Problem framing

- **Decision Supported**: Prioritizing monthly editorial content refresh budgets across large catalogs of indexable pages.
- **Unit of Analysis**: A single unique content URL aggregated over a 90-day pre-decision window.
- **Output**: An action score ($0.0 - 1.0$), probability score, and reason codes mapped to specific action playbooks (`expand_and_refresh`, `refresh_and_review_ctr`, `refresh`, `monitor`).
- **Cost of Wrong Call**:
  - *False Positive*: Wasting $300–$500 rewriting high-ranking, stable articles.
  - *False Negative*: Ignoring decaying high-demand content, allowing competitors to permanently capture organic SERP market share.
- **Why Data/ML**: Search performance involves non-linear interactions between SERP position decay, keyword search volume, content length, and CTR expectations that fixed heuristic rules cannot capture.

## 2. Data safety

- **Data Source**: `data/raw/content_refresh_anonymized.csv` (30,000 anonymized URLs).
- **Public-Safety Rules**: No client names, domain names, target URLs, or raw user search queries are stored or exposed.
- **Excluded Columns**: `trend_direction` and `trend_pct` were excluded from feature matrices to prevent target leakage, as they directly define the target label `is_declining_label`. `client_id` was strictly used as a grouping variable for the client-holdout split and never used as a learned feature.

## 3. Baseline

- **Deterministic Baseline Rule**:
  $$\text{Baseline Score} = 0.40 \cdot \text{Visibility} + 0.30 \cdot \text{Freshness Risk} + 0.25 \cdot \text{Position Opp} + 0.05 \cdot \text{Depth Gap}$$
- **Baseline Evaluation (Client-Holdout Split)**:
  - Precision@50: **24.0%**
  - Precision@20: **15.0%**
  - ROC-AUC: **0.627**
  - PR-AUC: **0.468**
  - Accuracy: **60.9%**

## 4. Model / analysis

- **Target Definition**: `is_declining_label` ($1 = \text{downward traffic trend}$, $0 = \text{stable/upward}$).
- **Feature Matrix**: 52 numeric and categorical features including `days_with_impressions`, `log_impressions_90d`, `avg_position`, `content_age_days`, `word_count`, `ctr`, `scroll_rate`, and `engagement_rate`.
- **Model Toolkit**: Logistic Regression (linear baseline), Decision Tree (depth=5), and Random Forest (200 estimators, balanced subsampling).

## 5. Evaluation

- **Validation Split**: `client_holdout` split (27,675 train rows across 80% client domains; 2,325 test rows across 20% unseen client domains).

### Performance Matrix (Evaluated on Held-Out Test Clients)

| Model | Precision@50 | Precision@20 | Precision@100 | PR-AUC | ROC-AUC | Accuracy |
|---|---|---|---|---|---|---|
| **Baseline Rule** | 24.0% | 15.0% | 36.0% | 0.468 | 0.627 | 60.9% |
| **Logistic Regression** | 40.0% | 35.0% | 44.0% | 0.521 | 0.700 | 66.1% |
| **Decision Tree** | 62.0% | 55.0% | 60.0% | 0.575 | 0.742 | 67.7% |
| **Random Forest (Selected)** | **74.0%** | **65.0%** | **72.0%** | **0.618** | **0.750** | **67.2%** |

- **Error Analysis**: False positives occur primarily on pages with recent organic keyword cannibalization or temporary macro search volume drops. False negatives occur on low-volume long-tail pages where traffic decay manifests gradually over extended periods.

## 6. Interpretation

- **Top Feature Drivers**:
  1. `days_with_impressions` (15.8% importance)
  2. `log_impressions_90d` (12.9% importance)
  3. `avg_position` (10.9% importance)
  4. `content_age_days` (9.5% importance)
  5. `char_count` & `word_count` (8.2% combined importance)
- **Insight**: Search visibility consistency (`days_with_impressions`) combined with rank position decay is significantly more predictive of traffic decline than content staleness (`days_since_last_update`) alone.

## 7. Recommendation

- **Action Engine Playbook**:
  - `expand_and_refresh`: Recommended for thin visible pages (`word_count < 1200`). Expand coverage by adding structured FAQ sections and missing subtopics.
  - `refresh_and_review_ctr`: Recommended for visible pages underperforming on CTR (`ctr < 0.5%`). Rewrite title tags and meta descriptions.
  - `refresh`: Recommended for stale decaying pages. Update facts, outbound links, and publish date.
- **Human Review Protocol**: Mandatory editorial review of search intent and brand compliance prior to content updates. High-converting transactional URLs are on the no-go list for automated restructuring.

## 8. Reproducibility

- **Environment**: Python 3.13, duckdb, scikit-learn, pandas, numpy, reportlab.
- **Random Seed**: `RANDOM_STATE = 42`.
- **Rerun Pipeline Command**:
  ```bash
  $env:PYTHONIOENCODING="utf-8"; python scripts/run_all.py
  ```
- **Receipt Files**: Committed in `outputs/model_results.json`, `outputs/summary.json`, `outputs/refresh_queue.csv`, and `data/processed/model_predictions.csv`.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset (https://flyrank.ai).
