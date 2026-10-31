# Capstone Report: Ranking pages that keep under-capturing clicks

- **Author:** Muhammad Abdullah Nazeer
- **Lane:** CTR / Engagement Opportunity Scoring
- **Repo:** https://github.com/M-Abdullah-Nazeer/Flyrank-Ml-Pipeline
- **Date:** October 31, 2026

## 0. Abstract

I ask which already-visible pages will keep earning fewer clicks than expected for their search position, so an editor can review them first. I use the FlyRank pseudonymized warehouse (daily Search Console rows, February to April 2026), with page features from days 1-15 of each month and a label from days 16 onward. A gradient-boosting ranker trained on February and March is compared with a transparent missed-clicks rule on held-out April and under stricter splits. On April (n = 59,986, base rate 49.7%) it reaches AUC 0.789 against 0.680 for the rule, and 92% of its top 500 pages under-captured clicks; with a later month and unseen clients the lead narrows (AUC 0.769 vs 0.679) on a small sample. The output is a ranked, human-reviewed action queue for decision support, an observed association that does not show why pages under-capture or that a rewrite would raise clicks.

## 1. Problem framing

- **Decision supported:** which pages a content editor should review first for snippet, content or refresh work.
- **Unit of analysis:** one page (client × content item) per month, with features from days 1-15 and an outcome from days 16 to month end.
- **Output:** a ranked queue with one reason code and one action per page.
- **Human action:** an editor checks the page (search features, intent, position moves) and decides whether to edit it.
- **Cost of a wrong call:** a wasted review (a false positive costs about 20 minutes of editor time, an assumption), or a missed page whose CTR collapses later (false negative). Nothing is changed automatically.
- **Why data helps:** a team cannot review every page, and CTR gaps at a given position persisted in my data (pages far below the expected CTR for their position kept getting 0.32× the expected clicks later).

## 2. Data safety

- **Data:** `FlyRank/internship-warehouse`, table `fact_content_daily_performance`, Search Console columns only. One row is one page on one day for one client; grain verified (March: 9,841,378 rows, 9,841,378 distinct keys). Rows kept where `gsc_data_available IS TRUE` (3,611,061 of 9,841,378 March rows, 36.7%).
- **Windows:** train 2026-02 and 2026-03, evaluate 2026-04; the final month (2026-06) was never used.
- **Excluded on purpose:** GA4, session-channel and AI-referral columns (another lane); pages under 100 impressions in days 1-15 or average position beyond 20; product flags and health scores such as `trend_direction`, `trend_pct` and `recommended_action` (leakage risk, never used as inputs); pseudonymous ids (used only for grouping and joins, never as features).
- **Leakage risks considered:** future-window columns, label-derived fields, and the expected-CTR curve (computed from training months only). A deliberate leak (adding future clicks) raised AUC to 0.985, confirming the audit would catch one.
- **Public safety:** no client names, domains, URLs or queries anywhere. The exported queue file has no ids and no exact counts. Raw data is not redistributed.

## 3. Baseline

- **Rule (Week 4):** score = missed clicks = impressions × (expected CTR − actual CTR), where expected CTR is the pooled CTR of the page's position bucket (1-3, 4-6, 7-10, 11-20), from days 1-15 of the training months. Higher score means review first.
- **Why it is fair:** it uses the same features, the same eligible pages and the same metrics as the model, and it needs no training labels. I also report the CTR gap ratio alone as a stronger simple reference.
- **Numbers on April (n = 59,986, base rate 49.7%):** rule AUC 0.680, precision@100/500/1000 = 0.79 / 0.786 / 0.763; CTR gap ratio alone AUC 0.745, precision@500 0.712.

## 4. Model / analysis

- **Method:** gradient boosting (`HistGradientBoostingClassifier`), compared with logistic regression (AUC 0.750, precision@500 0.824). I prefer gradient boosting only because it won clearly on held-out data (precision@500 0.918).
- **Features (all days 1-15):** log impressions, impression-weighted average position, CTR, CTR gap ratio (CTR ÷ expected CTR), impression trend (days 8-15 vs 1-7) and days with data.
- **Left out on purpose:** everything from days 16+, product flags and scores, ids, GA4 and AI-referral fields.
- **Target:** a page is labelled 1 if its clicks in days 16+ are below 50% of the clicks expected for its position bucket (needs at least 100 impressions in days 16+). The 50% cut-off is my choice.

## 5. Evaluation

- **Split and why:** train on February and March, test on April, so the model never sees a later month. Robustness checks: random rows (the easy version), clients held out, and later month plus unseen clients.
- **Model vs baseline, same April pages (base rate 49.7%):**

| Method | AUC | precision@100 | @500 | @1000 |
|---|---|---|---|---|
| Missed-clicks rule | 0.680 | 0.79 | 0.786 | 0.763 |
| CTR gap ratio alone | 0.745 | 0.78 | 0.712 | 0.735 |
| Logistic regression | 0.750 | 0.87 | 0.824 | 0.803 |
| Gradient boosting | 0.789 | 0.90 | 0.918 | 0.911 |

- **Stricter splits (rule → model):** random rows AUC 0.695 → 0.787; clients held out 0.697 → 0.792; later month 0.680 → 0.788; later month plus unseen clients 0.679 → 0.769 (n = 7,657, base rate 0.537, one draw, so directional). The Week 6 re-run gives 0.788 for the main split against 0.789 in Week 5, a rounding difference between runs.
- **Errors:** 8.9% of the model's top 1,000 picks were wrong. Top-3 pages had the highest wrong rate (13.5%), likely because a search feature can take the click; mid-volume pages (1k-5k impressions) 13.1%. Many false positives were near misses just above the cut-off. False negatives were pages whose CTR fell sharply later in the month, which no days 1-15 feature can foresee.
- **Leakage audit:** features end on day 15 (checked in code); no single feature predicts near-perfectly (strongest is CTR at AUC 0.75); about 80% of April pages also appeared in training months, and AUC on never-seen pages was 0.769 against 0.788 overall.

## 6. Interpretation

- Permutation importance is dominated by the CTR gap ratio (AUC drop 0.177), far ahead of log impressions (0.044) and CTR (0.015); position, trend and days with data each add under 0.006. A CTR that is low for the page's position is the main warning sign, which matches the signal audit.
- Signal audit verdicts (April-style checks): CTR gap persistence CONFIRMED; impression volume FALSE; trend and position MIXED.
- Pages with a shrinking impression trend under-captured 67% of the time against 49% for flat and growing pages, so a falling trend adds risk while growth gives no protection. This is directional; my data has no page age or update date.
- Negative result worth keeping: volume alone did not tell which pages keep under-capturing, and a plain position-to-CTR curve was only roughly monotonic.

## 7. Recommendation

Ranked by `priority = model score × missed clicks` (days 1-15), with one reason code per page (top 500 of 46,330 qualifying April pages):

1. `UNDER_CLICKED_PAGE1_REWRITE_SNIPPET` (324 pages): rewrite title and meta description.
2. `UNDER_CLICKED_TOP3_CHECK_SERP` (116): check search features first; test the snippet only if none.
3. `DECLINING_AND_UNDER_CLICKED` (44): review for refresh, then fix the snippet.
4. `STRIKING_DISTANCE_UNDER_CLICKED` (16): improve snippet and content depth.

- **How an editor uses it tomorrow:** start at the top, check each page for search features, how far below expected the CTR is, sudden position or impression moves, intent fit, and facts and brand voice; stop where extra review hours stop paying (about 426 missed clicks per review hour for the top 100, 146 for the top 1,000, under an assumed 20 minutes a page).
- **Do not automate:** publishing edits, deleting or merging pages, top-3 edits without the search-feature check, causal promises to clients, or use outside positions 1-20 and 100+ impressions.
- **Confidence and limits:** observed, directional, decision support. The priority queue's top 500 had 0.853 precision, below the 0.924 of ranking by model score alone, because large pages come first. Labels and the expected-CTR curve are my choices. Unseen-client evidence is one small draw. Monitoring proposals: retrain if precision@500 falls below base rate + 0.15, AUC below 0.70, label rate shifts by more than 0.10, or PSI of CTR gap ratio or position exceeds 0.25.

## 8. Reproducibility

- **Environment:** Google Colab, Python with `duckdb`, `pandas`, `numpy`, `scikit-learn`, `matplotlib`, `huggingface_hub`. I did not pin versions; they are the Colab defaults at run time.
- **Data access:** request access to `FlyRank/internship-warehouse` on Hugging Face, create a Read token and store it as a Colab secret named `HF_TOKEN`.
- **Run order from a fresh clone (open each in Colab and Run all):**
  1. `work/notebooks/w03_data_contract.ipynb`
  2. `work/notebooks/w04_baseline_score.ipynb`, `work/notebooks/w04_signal_audit.ipynb`
  3. `work/notebooks/w05_model.ipynb`
  4. `work/notebooks/w06_validation_audit.ipynb`
  5. `work/notebooks/w07_action_playbook.ipynb`
- **Seeds:** `random_state=0` for the model and the grouped splits; `np.random.RandomState(0)` for the random-row split and `RandomState(1)` for the unseen-client draw.
- **Outputs:** `work/outputs/` (model vs baseline, before/after splits, ranked queue, playbook, archetypes, decay, cost/value, monitoring, figures).
- **Sealed month:** the final month (2026-06) was not evaluated at all, so I make no sealed-evaluation claim.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai). I thank the FlyRank team for the pseudonymized warehouse release and the internship curriculum.

---

> **Claims checklist:** observed / measured / directional / decision-support language everywhere · no causal claims without an experiment · no "predicted Google's algorithm" · no client-identifying details · base rate shown next to every precision@K (49.7% on April) · numbers match the executed notebooks.
