# Predicting Content Decay for Proactive Refreshing

**Author:** Koushal Karthik
**Lane:** Refresh / Content Opportunity Scoring
**Repo:** [mlflyrankkarthik](https://github.com/koushalkarthik15/mlflyrankkarthik)
**Date:** September 2026

## Abstract
This paper investigates methods for identifying high-traffic webpages that are actively decaying in Google Search performance, allowing editors to proactively refresh them before traffic is permanently lost. We utilized a Random Forest classifier to rank pages based on their risk of decline over a 30-day window. The model was trained and evaluated on the FlyRank ML Internship dataset using a client-grouped split to ensure generalization. The resulting model outperforms a naive rule-based baseline in ranking accuracy, though it struggles to differentiate between actively decaying content and stable "evergreen" references. These findings provide a directional ranking engine that enables content teams to triage the highest-value opportunities efficiently.

## 1. Introduction & Problem Framing
Content decay is a natural process where previously high-performing pages gradually lose organic search visibility. For publishers and brands, identifying which pages to refresh is often done retroactively—after the traffic is already gone. 

This project aims to support the **content triage decision**: identifying which high-value pages are currently decaying so editorial teams can prioritize them for updates. 
*   **Unit of analysis:** One content item (page) per day.
*   **Output:** A ranked probability score predicting near-term decline.
*   **Action:** A human editor reviews the top-ranked pages and decides whether to rewrite, merge, or update the content.
*   **Cost of a wrong call:** If a page is flagged incorrectly (false positive), an editor wastes time reviewing or tweaking a page that didn't need it.

Machine Learning is uniquely suited for this problem because decline is rarely driven by a single threshold (e.g., age). Instead, it involves the complex interaction of age, historical impressions, and keyword positions, which a model can weigh simultaneously.

## 2. Data & Safety
This analysis was performed on the **FlyRank ML Internship starter dataset** (`content_refresh_anonymized.csv`), encompassing performance data for March 2026. 

**Exclusions & Leakage Prevention:**
Label-derived fields such as `trend_direction` (the target) and `trend_pct` (the exact calculation used to derive the target) were strictly excluded from the feature set. Including them would result in massive data leakage, as the model would effectively be looking at future outcomes to make its predictions. 

**Safety:**
All data used is pseudonymized. `client_id` and `content_id` were used exclusively for grouping and splitting data, never as predictive features. No client-identifying details, raw URLs, or private queries are present in the repository or this analysis.

## 3. Baseline
Before training a complex model, we established a transparent rule-based baseline:
**Rule:** A page is worth refreshing if it has high visibility (impressions > 1000) AND it is stale (older than 180 days). 
**Score:** `(age >= 180) * (impressions >= 1000) * impressions`

This acts as an honest floor. By running this baseline on our hold-out test set, we establish the precision our ML model must beat to justify its complexity. 

## 4. Methodology
**Method:** We utilized a **Random Forest Classifier** limited to a maximum depth of 5.
**Why it fits:** This lane requires a "which first?" ranking. Random Forests naturally output probabilities rather than rigid binary labels, and handle non-linear thresholds effortlessly. Limiting the depth prevents overfitting and maintains feature interpretability.

**Features Used:**
*   `content_age_days`
*   `impressions_90d`
*   `clicks_90d`
*   `avg_position`
*   `ctr`
*   `word_count`

**Label Definition:** 
A binary indicator `is_declining` where `trend_direction == 'down'`.

## 5. Evaluation
**Validation Design:** 
We used a **GroupShuffleSplit** grouped by `client_id` (80% train, 20% test). A standard random split is dangerous because the model might memorize a specific client's website architecture or performance patterns. Splitting by client forces the model to be evaluated on *completely unseen clients*, proving true generalization.

**Results:**
The Random Forest model and the Baseline were evaluated on the exact same hold-out test set using **Precision@50** (of the top 50 pages flagged, what percentage actually declined?).

| Metric | Score |
| :--- | :--- |
| **Base Rate (Random guessing)** | 0.536 |
| **Baseline Precision@50** | 0.640 |
| **Model Precision@50** | **0.820** |

The Random Forest model correctly identified decaying pages in the top 50 at a rate of 82%, a massive improvement over both random guessing and the rigid baseline.

## 6. Interpretation & Error Analysis
**What the model learned:**
The Random Forest leaned heavily on `content_age_days` and `impressions_90d` as its primary drivers. This confirms the baseline intuition (old, high-traffic pages decay), but the model was able to weave in `avg_position` to find deeper nuance without rigid cutoffs.

**Error Analysis:**
Where is the model wrong? A review of the model's top False Positives (pages it confidently flagged that were actually stable) revealed a pattern: the model aggressively flags extremely old pages with moderate traffic. However, some of these pages are "evergreen" references (e.g., privacy policies, historical glossaries) that naturally accumulate age without losing relevance. The model currently lacks the semantic understanding to differentiate between a stale news article and an evergreen reference document.

## 7. Recommendations
**The Action Playbook:**
Content teams should pull the Top 100 highest-scored pages from this model weekly. 
1. **Filter:** Editors must rapidly filter out "evergreen" or corporate reference pages that the model falsely flags due to high age.
2. **Review:** For the remaining articles, review the search intent and competing pages to determine if a content refresh, metadata tweak, or full rewrite is required.

**Limitations:**
The output of this model is **directional decision-support**, not an automated truth. It highlights risk based on observed historical patterns, but a human editor must ultimately decide if the page's content is actually outdated. 

## 8. Reproducibility
The code for this analysis is available in the `work/notebooks/` directory of the project repository.
- Baseline implementation: `w04_baseline_score.ipynb`
- Model training and evaluation: `w05_model.ipynb`

To reproduce:
1. Clone the repository.
2. Install dependencies (e.g., `pandas`, `scikit-learn`).
3. Run `w05_model.ipynb` top-to-bottom. The random seed is fixed (`random_state=42`) to guarantee identical splits and feature importances.

## Acknowledgments
Built on the FlyRank ML Internship dataset.
[https://flyrank.ai](https://flyrank.ai)
