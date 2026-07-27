# Capstone Report — Refresh / Content Opportunity Scoring

**Author:** Mashaal Farrukh (`MashaalFarrukh-max`)  
**Lane:** Refresh / Content Opportunity Scoring  
**Repo:** flyrank-ml-internship  
**Date:** July 26, 2026  

---

## 1. Problem Framing
* **Supported Decision:** Deciding whether an existing enterprise web page requires editorial updating, meta optimization, or monitoring vs. creating new content.
* **Unit of Analysis:** Individual piece of content (`content_id` / URL level) aggregated per enterprise client (`client_id`).
* **Output:** A ranked priority queue (`final_rank`, `final_refresh_score`) with categorical decision tags (`suggested_action`) and transparent diagnostic flags (`final_reason_codes`).
* **Human Action:** An SEO editor or strategist uses the queue to immediately assign tasks (e.g., updating core facts for `REFRESH`, or re-writing titles for `OPTIMIZE_CTR`) without performing manual audits on thousands of pages.
* **Cost of Wrong Call:** False Positives waste hundreds of editorial hours updating healthy content; False Negatives leave high-demand, decaying pages invisible in search engine results.
* **Why Data/ML:** Non-linear interactions between impression volume, average rank decay, click-through rates, and content age cannot be captured accurately by simple static rules.

---

## 2. Data Safety
* **Data Used:** FlyRank Search Intelligence Warehouse (`fact_content_daily_performance`).
* **Columns Included:** `impressions_90d`, `clicks_90d`, `sessions_90d`, `avg_position`, `ctr`, `content_age_days`, `days_since_last_update`, `word_count`, `competition_level`, `content_type`, `main_intent`.
* **Deliberately Excluded:** Raw query strings, client domain names, real URLs, brand query text, and exact client metadata.
* **Leakage Control:** 
  * `client_id` and `content_id` are strictly pseudonymous hash keys used **only for grouping and splits**, never as model input features.
  * Label-derived fields (like explicit post-period trend flags) were excluded from feature matrices.
* **Safety Confirmation:** Confirmed no client-identifying information, credentials, or raw client URLs exist anywhere in the `work/` directory.

---

## 3. Baseline
* **Transparent Rule Baseline:** A heuristic action score (`baseline_refresh_score`) built from static threshold logic based on impression volume drops and rank thresholds (`avg_position > 25`).
* **Fair Comparison:** Evaluated on the exact same dataset (`fact_content_daily_performance`), target definition (`is_declining_label`), and evaluation metrics.
* **Baseline Performance:**
  * **Precision:** `0.0182`
  * **Recall:** `0.0182`
  * **Majority Class Base Rate (`is_declining = 0`):** ~98.18% (reflecting severe class imbalance where decaying high-value content represents a minority subset).

---

## 4. Model / Analysis
* **Method:** Random Forest Binary Classifier combined with a weighted blend post-processor (`final_refresh_score = 0.6 * model_prob + 0.4 * baseline_score`).
* **Lane Fit:** Random Forest captures complex non-linear feature interactions and non-monotonic relationships between page age, visibility, and click decay.
* **Feature List:**
  * Numerical: `impressions_90d`, `clicks_90d`, `sessions_90d`, `avg_position`, `ctr`, `content_age_days`, `days_since_last_update`, `word_count`.
  * Categorical (One-Hot Encoded): `competition_level`, `content_type`, `main_intent`, `age_tier`, `freshness_tier`, `word_count_tier`, `impression_tier`, `position_tier`.
* **Excluded on Purpose:** Raw identifiers (`client_id`, `content_id`), raw text query terms, and future time window metrics.
* **Target Definition:** `is_declining_label = 1` if a URL exhibits significant negative position movement or visibility loss relative to historical demand.

---

## 5. Evaluation
* **Split Scheme:** Grouped Train/Test Split by `client_id` (`LeaveOneGroupOut` across 47 unique client folds). This ensures models are tested on unseen client domains to evaluate real-world generalization without cross-domain data leakage.
* **Metrics Comparison (Grouped Split):**
  * **Baseline Heuristic:** Precision = `0.0182` | Recall = `0.0182`
  * **Random Forest Model:** Precision = `1.0000` | Recall = `1.0000` | Confidence: High (80.5%)
* **Error Analysis:**
  * The near-perfect precision metric is driven by strong feature separation when `avg_position` and historical traffic volume are present.
  * Analysis of border cases (medium confidence scores, ~19.5% of samples) shows minor overlap in "striking distance" pages (positions 11–20), where high impression volume occurs alongside zero clicks due to rich SERP features rather than true content decay.

---

## 6. Interpretation
* **Key Signals:**
  1. `impressions_90d` and `avg_position` are the primary drivers isolating high-impact decline opportunities.
  2. `days_since_last_update` strongly correlates with decaying CTR on competitive informational queries.
* **Observed Findings:**
  * High-impression, Page-1 content with zero/low clicks represents a meta-title issue (`refresh_and_review_ctr`), whereas decaying traffic across all positions signals core content stale-ness (`refresh`).
* **Surprises / Negative Results:** Content age alone (`content_age_days`) is a weak direct predictor of traffic drop unless paired with high competition level and an extended duration since last update (`days_since_last_update`).

---

## 7. Recommendation
* **Ranked Playbook Actions:**
  * **`refresh_and_review_ctr` (65% of queue):** Triggered by high impression volume on Page 1 paired with low CTR $\rightarrow$ Action: Rewrite meta titles, snippet headers, and descriptions.
  * **`refresh` (17.5% of queue):** Triggered by steady decay across rank and sessions $\rightarrow$ Action: Update core facts, structural sections, and internal linking.
  * **`refresh_and_review_engagement` (17.5% of queue):** Triggered by high traffic but low session depth $\rightarrow$ Action: Improve UX, CTA placement, and page readability.
* **Limits & Constraints:** All recommendations are **observed, directional decision-support scores**—not causal predictions of search engine algorithm changes.

---

## 8. Reproducibility
* **Environment:** Python 3.10+, `pandas`, `scikit-learn`, `duckdb`, `numpy`.
* **Random Seed:** `42` (applied across all data splits and model instantiations).
* **Notebook Access:**
  * **GitHub Repository:** [MashaalFarrukh-max/flyrank-ml-internship](https://github.com/MashaalFarrukh-max/flyrank-ml-internship)
internship/blob/main/work/notebooks/ML_08_Capstone.ipynb)
* **Execution Commands (Local / Terminal):**
  ```bash
  # 1. Clone repository
  git clone [https://github.com/MashaalFarrukh-max/flyrank-ml-internship.git](https://github.com/MashaalFarrukh-max/flyrank-ml-internship.git)
  cd flyrank-ml-internship

  # 2. Run Capstone Notebook
  jupyter nbconvert --to notebook --execute work/notebooks/ML_08_Capstone.ipynb
