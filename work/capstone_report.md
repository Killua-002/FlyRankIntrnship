# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Abdullah Hany Ahmed Raafat
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Killua-002/FlyRankIntrnship.git
- **Date:** 25/7/2026

## 1. Problem framing

This work supports a content editor's weekly triage decision: given a client's full page
inventory, which pages should be reviewed first with limited hours available? The unit of
analysis is one content page, summarized over its trailing 90 days. The output is a ranked
queue with a reason code and a recommended action (review meta/snippet, content refresh
review, monitor, or manual review). The action a human takes is picking items off the top of
that queue and deciding whether to refresh, rewrite, or leave a page alone.

The cost of a wrong call is asymmetric. A false positive — flagging a healthy page — costs an
editor roughly an hour reviewing a page that didn't need it. A false negative — missing a page
that's genuinely declining with real demand behind it — costs traffic that keeps compounding
the longer it goes unnoticed. That asymmetry is why this project optimizes precision at the
top of a ranked list (precision@K) rather than overall accuracy, and why data/ML helps at all:
no single hand-written threshold separated decliners from the rest cleanly (see Section 3),
but a model combining several weak signals does.

## 2. Data safety

This project uses the anonymized starter dataset (`data/raw/content_refresh_anonymized.csv`):
30,000 pseudonymized content rows across 32 clients, one snapshot, with trailing-90-day
aggregates and static content metadata.

Features used: `content_age_days`, `days_since_last_update`, `impressions_90d`,
`avg_position`, `ctr`, `word_count`, `engagement_rate`. `client_id` and `content_id` are used
only as pseudonymous grouping keys for the train/test split — never as model features.

**Deliberately excluded:** `trend_pct`, the exact value the label (`trend_direction`) is
bucketed from. Including it hands the model the answer — demonstrated directly rather than
just asserted: adding it back in as a feature pushes precision@50 to a clean 1.000 (see
Section 5), confirming the leakage risk is real and the exclusion is load-bearing, not
theoretical. No product/decision flags exist in this starter release, so there's nothing else
of that category to accidentally include.

Confirmed: no client names, domains, raw queries, or credentials appear anywhere in `work/` —
`client_id`/`content_id` are opaque pseudonyms throughout every notebook and export.

## 3. Baseline

The baseline is a transparent, hand-written rule, built only after checking its underlying
signal against real data rather than assuming it:

- **Signal check 1 (staleness, behind refresh flags):** MIXED. Decline rate barely moved across
  staleness buckets (0.47–0.61 against a 0.542 base rate) — staleness alone does not separate
  decliners here.
- **Signal check 2 (CTR-vs-position, behind CTR-fix logic):** CONFIRMED. Weighted CTR drops
  sharply from 0.355 (page 1) to 0.055 (deep), with real sample sizes per tier (533–8,633 rows).

The baseline rule is built on the confirmed signal: `score = (expected CTR for a page's
position tier − its actual CTR) × impressions_90d`, applied to visible pages
(impressions_90d ≥ 200). This is a fair comparison because it's scored on the *same held-out
test rows* as the model, not the full dataset it was built and shown on — re-using training
rows for a "baseline" comparison would inflate it artificially.

| method | precision@20 | precision@50 |
|---|---|---|
| baseline (rule) | 0.400 | 0.540 |
| dummy (stratified) | 0.550 | 0.580 |
| random forest | 0.850 | 0.660 |

## 4. Model / analysis

**Method:** Random Forest (300 trees, `class_weight="balanced"`). This is a ranking/scoring
task — "which pages first" — evaluated at precision@K, implemented via a classifier's
probability output read as a ranking score rather than a hard label.

**Feature list:** `content_age_days`, `days_since_last_update`, `impressions_90d`,
`avg_position`, `ctr`, `word_count`, `engagement_rate` — all pre-decision, observable before
any editorial action. Left out on purpose: `trend_pct` (label-derived), `trend_direction`
(the label itself), and any product/decision flag (none exist in this release).

**Target / proxy, in one sentence:** `is_declining_label = (trend_direction == "down")`, a rule
computed from the current 30-day-vs-previous-30-day impression window — a proxy for "this page
looks like it's declining right now," not an observed future outcome.

## 5. Evaluation

**Split:** `GroupShuffleSplit`, grouped by `client_id`, 75/25 (8 of 32 clients held out,
n=7,115 test rows). A plain random row split was tested and rejected: it let 31 of 32 clients
appear in both train and test, letting the model partly learn "which client is this" instead
of "is this page declining." Under a random split, precision@50 reads 0.880; under the
client-grouped split it drops to 0.660 — a ~0.22 gap attributable to that memorization risk,
not to a worse model. The grouped number is the one reported as this project's result.

**Leakage check:** adding `trend_pct` back in as a feature pushes precision@50 to a clean
1.000 — confirming the test harness would catch real leakage if it happened by accident, and
confirming why `trend_pct` stays excluded.

**Errors:** false positives in the top-50 model picks cluster among pages with poor average
position (20–39) and near-zero CTR that are actually trending `stable` or `up` — the model
appears to partly learn "this page performs badly" as a proxy for "this page is declining,"
which are related but not identical. Named as a real limitation, not swept under the rug.

## 6. Interpretation

Feature importances: `impressions_90d` (0.256) and `avg_position` (0.233) dominate, followed
by `content_age_days` (0.156) and `word_count` (0.145); `days_since_last_update` is weakest
(0.037). That last point is a negative result worth keeping, not hiding — it's consistent with
the staleness signal check in Section 3 coming back MIXED. Two independent methods (a manual
bucket check and a trained model) arriving at the same conclusion about staleness is a good
sign neither result was a fluke.

## 7. Recommendation

The validated model, scored honestly with out-of-fold predictions (`GroupKFold`, grouped by
client, so no row is scored by a model that trained on it), combines with the baseline signal
to produce four archetypes, each mapped to one action:

| Archetype | Action | Decline rate (vs. 0.542 base) |
|---|---|---|
| `high_value_ctr_gap` | review_meta_and_snippet | 0.685 |
| `declining_low_ctr_signal` | content_refresh_review | 0.599 |
| `ctr_gap_stable` | monitor_ctr_gap | 0.547 |
| `stale_but_visible` | manual_review_stale | 1.000 (n=4 — too small to trust) |
| `no_action` | — | 0.399 |

An editor works down the resulting 19,308-row queue (of 30,000 total pages), ranked by
`decline_score × impressions_90d` so limited review hours go to the highest-stakes pages
first. Confidence: this is decision-support from one snapshot and one client-grouped
holdout — not a guarantee, and not evidence that acting on a recommendation will cause
traffic to recover. Nothing here should be automated into an actual content change without
human review; the no-go list (Week 7 / capstone Section 6) names three specific failure
patterns found by inspecting real top-ranked picks by hand.

## 8. Reproducibility

**Fresh clone → same numbers:**
```bash
git clone https://github.com/Killua-002/FlyRankIntrnship.git
cd FlyRankIntrnship
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute work/notebooks/capstone.ipynb
```

**Random seed:** `SEED = 42`, used consistently for every split and every model in the
capstone notebook and every weekly notebook it consolidates.

**Environment:** pandas, numpy, scikit-learn, matplotlib (see `requirements.txt` in the repo
root for exact versions).

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
>
> **Metrics vs. base rate:** test-set base rate is 0.517 — every precision@K number above
> should be read against that, not in isolation.
