# Refresh Opportunity Scoring: Ranking Pages for Human Review

- **Author:** FlyRank ML Internship participant
- **Lane:** Refresh / Content Opportunity Scoring
- **Repository:** https://github.com/zgander/flyrank_ml
- **Analysis window:** February 2026 features; March 2026 outcome

## 0. Abstract

Can observable page-performance signals help prioritize content pages for a human refresh review? This analysis uses the FlyRank ML Internship warehouse, aggregating February 2026 Google Search Console (GSC) and available Google Analytics 4 (GA4) measurements and defining a March 2026 zero-click outcome for eligible pages. A Random Forest classifier was evaluated using a client-grouped holdout, with a random-row split reported only as a diagnostic comparison and a transparent W04 rule as the baseline. On held-out clients, the Random Forest achieved Precision@50 of 0.760, Average Precision of 0.473, and ROC-AUC of 0.853, versus 0.340 Precision@50 for the W04 baseline reported on the same W05 test cohort; the grouped test positive rate was 15.3%. The resulting rank is intended to support human review of pages, not to prove that a refresh will recover clicks or to predict a search engine's ranking algorithm.

## 1. Problem framing

**Decision:** which eligible pages should an editor or SEO analyst inspect first for a possible content refresh or opportunity review?

**Unit of analysis:** a pseudonymized content page aggregated across the specified monthly window. **Output:** a ranked review queue based on the model's estimated probability of the defined outcome. **Human action:** inspect the page's content, intent match, technical status, and context before deciding whether to update it. A false positive consumes review time; a false negative may leave a potentially useful page unreviewed.

The prediction target is a narrow operational proxy, not refresh success: `went_dark = 1` when a page has at least one measured GSC day in March 2026 and zero measured GSC clicks in that month. This can help triage pages with prior visibility, but does not say why clicks were zero or whether editing would change the outcome.

## 2. Data and safety

The analysis queries the FlyRank ML Internship warehouse's `fact_content_daily_performance` table for February and March 2026. February is the feature window; March is the outcome window. Aggregates are constructed per `client_hash_id` and `content_hash_id`; the modeling population is restricted to pages with at least 500 February GSC impressions and measured GSC coverage in both months.

The W06 eligible population contained **46,152 rows across 31 clients**, with an overall positive rate of **15.50%** before the holdout split. The grouped test set's positive rate was **15.3%**. These figures describe the selected, measured population, not all pages in the warehouse or all web pages.

Features used:

- `feb_impressions`
- `feb_clicks`
- `feb_ctr`
- `feb_avg_position`
- `feb_gsc_available_days`
- `feb_ga4_sessions`
- `feb_ga4_available_days`

Client/content IDs were excluded from the feature matrix. The client ID was used only to keep all pages from a client together during grouped validation. March clicks and March availability were used only to construct the outcome/population eligibility, not as predictors. Product-derived scores and actions, trend fields, and the target itself were excluded. No client names, domains, URLs, raw search queries, credentials, or row-level review queue are included in this public paper.

**Selection caveat:** requiring measured GSC coverage in March uses outcome-window availability to define the evaluation population. The reported findings are conditional on measurement coverage and do not automatically apply to pages without March data.

## 3. Baseline

The transparent W04 baseline ranks eligible pages using the existing rule-based score and was reconstructed on the W05 held-out test cohort. It achieved **Precision@50 = 0.340** (17 relevant pages in the top 50). The Random Forest result from the W06 client-grouped evaluation is **Precision@50 = 0.760** (38 of 50 in the top 50). This is an absolute difference of **0.420** in Precision@50 on the held-out test cohort.

The baseline comparison is useful because both methods are evaluated on the same eligible test population and top-50 budget. The W06 grouped split is the primary estimate; the random-row split is a diagnostic, because it can place pages from the same client in both train and test. Verify baseline/model outputs from the committed W05 and W06 notebooks when rerunning the analysis.

## 4. Methodology

A Random Forest classifier was trained on February-only features to rank pages by their predicted probability of the March `went_dark` outcome. The target is a measured zero-click proxy, not a causal label for whether a page needs a refresh. The W06 notebook rebuilt the eligible frame from the warehouse and compared:

1. **Random-row diagnostic split:** a random page-level split where clients may occur in both partitions.
2. **Client-grouped primary split:** `GroupShuffleSplit(test_size=0.20, random_state=42)`, keeping every page for a client in only one partition.

The grouped split tests transfer to held-out clients more directly than a random-row split. The model was evaluated with Precision@50, Average Precision (AP), ROC-AUC, and test positive rate. Precision@50 measures how many of the top 50 ranked pages had the measured outcome; AP and ROC-AUC assess ranking/discrimination across thresholds. The base rate is reported to contextualize precision.

### Leakage audit

All six executable checks passed in W06:

1. Features were February-only.
2. No future, target-derived, or product-action fields appeared in the feature list.
3. Client/content identifiers were excluded from features.
4. The target matched the March zero-click rule (zero label mismatches).
5. The grouped split had zero client overlap.
6. Every eligible row met the 500-impression threshold.

These checks reduce common leakage risks but do not establish that the target is a complete or causal measure of content quality.

## 5. Results

### Validation results

| Split / method | Precision@50 | Average Precision | ROC-AUC | Test positive rate |
|---|---:|---:|---:|---:|
| Random-row split (diagnostic) | 0.760 | 0.478 | 0.857 | 15.3%* |
| Client-grouped split (primary) | 0.760 | 0.473 | 0.853 | 15.3% |
| W04 rule baseline (W05 held-out cohort) | 0.340 | — | — | 15.25% in W05 test set |

\*The random-row split has its own test partition; the displayed positive rate is rounded as reported in the notebook output. Use the executed notebook's validation table as the source of truth if exact split-specific rates are needed.

The two W06 split designs produced the same Precision@50 at the reported precision, while AP and ROC-AUC were slightly lower under client-grouped validation (0.005 and 0.004 absolute, respectively). This is not evidence of a large random-split inflation in these metrics, but grouped validation remains the primary design because it avoids client overlap. The top-50 result is substantially above the 15.3% grouped test positive rate, indicating useful ranking enrichment in this selected sample.

### Model vs baseline at the top 50

| Method | Relevant pages in top 50 | Precision@50 |
|---|---:|---:|
| W04 rule baseline | 17 / 50 | 0.340 |
| Random Forest, client-grouped W06 | 38 / 50 | 0.760 |

The difference is +0.420 absolute Precision@50, or 21 additional measured positives in a 50-page review queue. This comparison should be read as retrospective ranking performance for the defined proxy, not as proof of business impact.

### Error interpretation

At a 0.50 probability threshold in the W05 error analysis, the model produced 3,062 true positives, 15,937 true negatives, 5,291 false positives, and 758 false negatives on 25,048 held-out rows. The thresholded confusion matrix is not the same as the top-50 ranking decision. False positives represent pages flagged by the probability threshold that did not meet the March zero-click target; false negatives are pages with the target outcome that fell below that threshold. In practice, the top-50 queue is the intended review mechanism, and human verification is essential.

## 6. Interpretation and limitations

The model's ranking is useful as a **triage signal** within the selected population. In the W05 Random Forest, impurity-based feature importance was concentrated in February clicks (0.423), impressions (0.266), and CTR (0.195); other features had smaller importance. Permutation importance similarly ranked clicks and impressions highest, while GA4 sessions had slightly negative mean AP contribution in that run. This suggests that the model mainly exploits prior search-performance scale and click-through signals; importance does not establish causality or feature necessity in every client context.

Limitations:

- `went_dark` means zero measured March GSC clicks, not proven content decay, low quality, or a refresh opportunity.
- The results are conditional on at least 500 February impressions and measured GSC coverage in both months; selection and coverage can bias which pages are evaluated.
- Only one feature month and one outcome month are evaluated; seasonality, longer trends, and repeated temporal validation are not addressed.
- There are 31 clients in the eligible frame and 7 clients in the W05 grouped test partition, so client-level generalization evidence is limited.
- GA4 coverage is sparse; availability flags describe measurement coverage, not user behavior itself.
- Retrospective metrics do not measure whether a human refresh decision was correct or whether a refresh caused clicks to recover. A prospective experiment or well-designed causal evaluation would be required for that claim.
- Feature importance is model-specific and can be affected by correlated predictors.

## 7. Ranked recommendations

1. **Use the ranked queue as a review aid, not an automatic refresh instruction.** Start with the top-ranked pages, then manually verify indexability, query/page intent match, content accuracy, and technical health.
2. **Prioritize pages with substantial prior visibility for contextual review.** The 500-impression floor focuses review effort on pages with observable search exposure; do not interpret it as a universal threshold.
3. **Add reasons and coverage context to every recommendation.** Show February clicks, impressions, CTR, average position, and GSC/GA4 availability so reviewers can understand what evidence drove the ranking.
4. **Create a “not enough data” path.** Pages lacking measured GSC coverage should be routed to measurement checks rather than assigned a low or high risk score by default.
5. **Evaluate prospectively before claiming impact.** Track reviewer decisions and subsequent outcomes; where feasible, compare refreshed pages with a credible control group and account for seasonality and regression to the mean.
6. **Revalidate across later time windows and additional clients.** Repeat past-to-future evaluation and report uncertainty before operationalizing the model broadly.

**Confidence:** moderate that the model enriches the top-50 queue for the defined retrospective label in this measured sample; low that this directly identifies pages whose refresh will improve performance.

## 8. Reproducibility

- Repository: https://github.com/zgander/flyrank_ml
- W05 model notebook: [`work/notebooks/w05_model.ipynb`](https://github.com/zgander/flyrank_ml/blob/main/work/notebooks/w05_model.ipynb)
- W06 validation audit: [`work/notebooks/w06_validation_audit.ipynb`](https://github.com/zgander/flyrank_ml/blob/main/work/notebooks/w06_validation_audit.ipynb)
- Capstone notebook: [`work/notebooks/capstone.ipynb`](https://github.com/zgander/flyrank_ml/blob/main/work/notebooks/capstone.ipynb)
- Validation seed: `42`; grouped split: `GroupShuffleSplit(test_size=0.20, random_state=42)`.
- Run in Google Colab with a Hugging Face token stored privately as a Colab Secret named `HF_TOKEN`; never paste the token into a notebook or commit it.
- Run the W05 and W06 notebooks from top to bottom. Dependencies include DuckDB, pandas, NumPy, and scikit-learn; use the repository's `requirements.txt` where applicable.
- The warehouse queries and metric calculations live in the notebooks. The W06 notebook rebuilds the frame and prints the validation comparison; do not treat this paper's manually transcribed metrics as a substitute for rerunning those cells.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai). The analysis uses the internship warehouse's content-performance aggregates for February and March 2026. All reported outcomes are retrospective measurements and should be interpreted within the selection rules and limitations described above.
