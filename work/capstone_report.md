# Capstone Report — Organic search intelligence: CTR-fix review queue

- **Author:** Md. Imdadul Haque Somik
- **Lane:** Refresh / content opportunity scoring, narrowed to a CTR-fix action queue (scoring and ranking)
- **Repo:** https://github.com/ihSomik/flyrank-ml-internship
- **Date:** 2026-10-09

All results are observed on one sampled month (March 2026) and are directional decision-support.

## 1. Problem framing

**Decision:** which pages should an SEO/content specialist review first? **Unit of analysis:** one page (client + content hashed id) over a 21-day feature window. **Output:** a ranked review queue with one reason code and one suggested action per page. **Human action:** review, then rewrite title/description, refresh, improve links, hold, or leave alone. **Wrong-call cost:** a false alarm wastes editor time; a miss leaves clicks unclaimed. **Why data/ML:** many signals interact (impressions, position, CTR, momentum), and a single rule cannot weigh them, but any model must first beat a transparent rule.

## 2. Data safety

Data: the FlyRank ML Internship warehouse, March 2026 daily page-performance table (9,841,378 rows, grain date x client x page, verified unique) joined to the content dimension table. Search Console rows only (36.7% of rows).

Deliberately excluded: paid-channel columns (ad budgets, not organic SEO), AI-referral columns (measured after the fact), same-day clicks/impressions/position/engagement (too close to the label; position is filled only when impressions exist, a leak by availability), and client and page ids as features (grouping only). The Week 1-2 starter CSV fields `trend_direction` and `trend_pct` are label-derived and were not used in the final model.

Leakage risks considered: feature window (days 1-21) ends before the label window (days 22-31); the CTR norm is pooled across clients (disclosed, and re-learned from training clients only: baseline AUC 0.785 vs 0.786); a deliberate label-derived trap raised AUC from 0.905 to 1.000, so a real leak would show; a random-noise column scored AUC 0.509. All checks passed in this sample, which does not prove leakage is absent. Only hashed ids exist in the data; no client names, URLs or private queries appear in `work/` or the paper (confirm with a final search before submitting).

## 3. Baseline

Rule: score = max(expected CTR for the page's position band - actual CTR, 0) x impressions (estimated missed clicks). Eligible: at least 100 impressions and average position 1-20 in days 1-21. Expected CTR is the pooled CTR of eligible pages in the same band. It is a fair comparison because it uses the same pages, the same window and the same folds as the model, and it is what an editor could compute by hand. Same-split numbers (5 held-out client folds): AUC 0.785, average precision 0.668, precision at top 5% 0.904, captured later missed clicks at top 5% 38.8%. Label base rate: 25.4% overall (12.7% to 34.5% by fold). A second floor, impressions only, gives AUC 0.837.

## 4. Model / analysis

Models: logistic regression, shallow random forest, small gradient boosting (depth 3). Best by AUC: gradient boosting. Target in one sentence: a page is positive when its later missed clicks (expected CTR frozen from days 1-21, applied to days 22-31) fall in the top quarter of training pages and are at least 1 click, which is a proxy and not a measured result of a rewrite.

Features (16, all known by day 21): log impressions, CTR, average position, active days, one-day spike share, position wobble (std), last-7-day impression share, last-7-day CTR, log search volume, log word count, log char count, competition level, and four missing-value flags. Left out on purpose: client id, paid, AI-referral, GA4, product flags, the Week-4 score and expected CTR.

## 5. Evaluation

Split: 5-fold GroupKFold by client (38 clients), so test clients are unseen; time-forward by construction; thresholds learned on training pages only; identical folds for every method.

| Method | AUC | Avg precision | Precision @ top 5% | Captured @ top 5% |
|---|---|---|---|---|
| Week-4 rule (baseline) | 0.785 | 0.668 | 0.904 | 38.8% |
| Impressions only | 0.837 | 0.552 | 0.635 | 33.8% |
| Logistic regression | 0.876 | 0.687 | 0.830 | 39.8% |
| Random forest (shallow) | 0.891 | 0.726 | 0.868 | 33.8% |
| Gradient boosting (small) | 0.906 | 0.761 | 0.914 | 37.4% |

Base rate 25.4% (precision at top 5% must be read against it). Paired against the rule, gradient boosting gains +0.121 AUC (5 of 5 folds) but -0.014 captured clicks (better in 2 of 5 folds). A naive random split gave AUC 0.915 vs 0.906 grouped, a small inflation here. The best model was chosen on the folds it is scored on, so its edge is slightly optimistic.

Error analysis (top 5% queue, gradient boosting): queue precision 0.92 vs base rate 0.254; 275 false alarms and 14,295 misses against 3,171 hits. False alarms often had a high one-day spike share (a spike inflated the first window); misses were mostly smaller pages (median 1,875 impressions vs 6,552 for hits). Precision was similar across position bands (0.87 to 0.94).

## 6. Interpretation

The model ranks the whole list better than the rule, but the top of the queue, where an editor works, is about as good under the rule. Impressions carry most of the signal (permutation importance 0.33 AUC drop, then CTR 0.06 and last-week impression share 0.02), so a large part of the skill is "big pages matter". The model's queue overlaps 73-85% with the rule's. Importance is not cause. Negative or modest results: the model did not capture more clicks at the top; CTR by position is only partly monotonic (verdict MIXED); and among 8,131 fading pages, the next 10 days ran at 1.56x the last-7-day rate, so fading weeks often rebound (regression to the mean is likely part of this). Combined ranking (rule x model probability) gave AUC 0.820 vs 0.797 for the rule with captured clicks level (41.4% vs 41.3%), so it is the ranking used in the playbook.

## 7. Recommendation

Of 68,868 eligible pages: 33,160 ACT (48.2%), 19,858 HOLD (28.8%), 15,850 MONITOR (23.0%). Ranked actions (clicks at stake are an upper bound, effort hours are assumptions for the editorial team to replace):

1. Review title and description on page-1 pages with CTR below the norm: 20,838 pages, about 165,331 clicks at stake per 30 days, 0.5 h each, 15.9 per effort hour.
2. Verify, then refresh fading pages: 5,031 pages, about 44,646, 3 h each, 3.0 per hour.
3. Improve relevance and links for positions 11-20: 7,291 pages, about 30,553, 1.5 h each, 2.8 per hour.
4. Verify HOLD pages (thin evidence, one-day spikes) before any action.
5. Monitor the rest monthly.

Tomorrow an editor opens the ACT list from the top, checks the live results page for branded queries and answer boxes, drafts a change, has a second person approve, and logs the outcome. A 25-page week touches about 2.7% of the value, so capacity is the constraint. Confidence: moderate for ordering, low for any size of gain. Limits: one month, proxy label, page-level data, observational, no causal claim, no promised click gain. Nothing is safe to automate. Monitoring: re-score monthly, re-learn the norm if a band moves over 25%, drop the model ranking if its AUC gain falls below 0.02, revisit rules if reviewers reject over 30%.

## 8. Reproducibility

Notebooks are in `work/notebooks/`; run in Google Colab (Runtime -> Run all) in this order, with a Hugging Face read token stored as the Colab secret `HF_TOKEN` and access to the internship dataset:

1. `w03_data_contract.ipynb`
2. `w04_baseline_score.ipynb` (writes `work/outputs/baseline_metrics.json`)
3. `w05_model.ipynb` (writes `work/outputs/model_metrics.json`)
4. `w06_validation_audit.ipynb` (writes `work/outputs/validation_audit.json`)
5. `w07_action_playbook.ipynb` (writes `work/outputs/action_playbook.json`, `playbook_metrics.json`, `work/figures/*.png`; the row-level CSV stays out of git)
6. `capstone.ipynb`

Seeds: `random_state=42` for sampling, splits and models; noise control uses `default_rng(0)`; fold assignment is deterministic (GroupKFold). Environment: Python 3 on Colab with `huggingface_hub`, `pandas`, `pyarrow`, `scikit-learn`, `matplotlib` (installed with `pip install -q`; versions not pinned). TODO before submitting: paste your `pip freeze | grep -iE "pandas|scikit|pyarrow|huggingface|matplotlib"` output here. Paper: `docs/index.html`, deployed via GitHub Pages; URL in `submission/paper_url.txt`.

---

Claims checklist: observed / measured / directional / decision-support language used throughout; no causal claims; no "predicted Google's algorithm"; no client-identifying details; re-run the notebooks and confirm the numbers above match before submitting.
