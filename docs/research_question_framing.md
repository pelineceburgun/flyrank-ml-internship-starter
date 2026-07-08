# Research Question Framing

## 1. Provisional Lane

**Lane 2: Refresh / Content Opportunity Scoring** (Core lane, per the FlyRank ML Internship Intern Dataset and Lane Guide)

This is provisional and may be confirmed or changed by the end of Week 4, pending exploratory analysis of the approved warehouse release (`flyrank_pseudonymized_warehouse_release_v20260703`) or continued work on the starter dataset.

**Why this lane, provisionally:**
- It has the strongest data depth of the four core lanes: `fact_content_daily_performance` (78.8M rows) carries dense GSC impressions (28.9M rows) and meaningful click/session coverage, versus thinner signal in AI-referral or engagement-only fields.
- The starter pipeline already demonstrates that a learned ranking can outperform a transparent rule on this problem shape (baseline rules: Precision@50 = 0.240; random forest: Precision@50 = 0.740), which gives a concrete, evidence-backed answer to "why not just train a model" rather than a speculative one.
- The lane's output — a ranked review queue with reason codes — maps directly onto a real FlyRank workflow (content strategist review), so the decision/action link is not hypothetical.
- I now have Hugging Face access to the approved warehouse release (`FlyRank/internship-warehouse`, build `flyrank_pseudonymized_warehouse_release_v20260703`). An initial look at `dim_clients` (104 rows) surfaces a concrete constraint this framing must account for: clients carry an `access_profile` of `gsc_and_ga4`, `gsc_only`, `no_search_or_analytics_access`, `ga4_only`, or `source_only_missing_client_dimension` (the last being rows with null client metadata — a data-quality artifact, not a usable client). Only `gsc_and_ga4` clients provide the full feature set (search + engagement/session signal); `gsc_only` clients can support search-signal features but not engagement ones; `no_search_or_analytics_access` and the malformed rows should be excluded. The true usable client count is therefore smaller than 104 and has not yet been finalized — this will be confirmed with an actual filter/count pass early in Week 1–2, not assumed here.
- `gsc_data_start` and `ga4_data_start` vary substantially by client (confirming the lane guide's "unbalanced panel" warning) — some clients have GSC history back to January 2025, others have no GSC start date at all despite `has_gsc_access = true`. Any feature window (e.g., prior 90 days) must be checked against each client's actual data start before use, so that a lack of history isn't misread as a lack of signal.
- The specific feature/label window, minimum-volume thresholds, final target definition, and the confirmed usable-client subset are not yet fixed and may change at the Week 4 checkpoint.

## 2. Search / Discoverability Question

Given content and search-performance signals for a client's page inventory, **which pages should be prioritized for human review first — for refresh, expansion, protection, pruning, or monitoring — and can a learned ranking do this more reliably than a simple transparent rule?**

This is phrased as a ranking/prioritization question, not a "will this page recover" question. Per the lane guide, "right page to fix" means *right page to review first, given evidence and limited review capacity* — not a guarantee that action on it will produce recovery. Establishing recovery causally would require an experiment, which is out of scope here.

## 3. Unit of Analysis

The **content item** (`content_hash_id`), scored at a defined decision point using a feature window prior to that point (e.g., prior 90 days). Where a future-outcome label is used instead of the starter's current-window `trend_direction` proxy, the corresponding target window (e.g., next 30 days) is defined separately and does not overlap the feature window — this separation will be explicit in the data contract before any modeling begins.

## 4. Output

A **ranked review queue**, where each content item has:
- A priority/opportunity score
- One or more human-readable reason codes (e.g., `stale_visible_page`, `declining_with_demand`, `thin_visible_page`) drawn from observable signals, not from FlyRank's own product decision flags
- A confidence label, gated on minimum evidence (sufficient impressions/sessions), not just a raw score

This is decision-support output, meant for a reviewer to inspect and act on — not an autonomous action trigger.

## 5. Action Someone Could Take

A content strategist or SEO reviewer at FlyRank could use the ranked queue to decide, per page: refresh the content, expand it, protect it (leave as-is, monitor), prune it, or flag it for deeper investigation (e.g., possible consolidation/cannibalization). The reason codes let the reviewer sanity-check *why* a page was surfaced before committing review time or content-production budget to it.

## 6. Cost of a Wrong Recommendation

- **False positive (page flagged as a priority, but isn't actually declining or underperforming):** wastes reviewer time and possibly content-production budget on a page that didn't need it — a direct, measurable cost since review capacity is limited by design (this is why the lane guide frames the task around top-K capacity, not full-inventory scoring).
- **False negative (a genuinely declining, high-value page not surfaced):** the page continues losing visibility/traffic unnoticed, which is a real business cost for the client, though harder to measure directly than a false positive's wasted effort.
- **Mistaking consolidation, seasonality, or noise for real decline:** per Section 7 of the lane guide, a drop in one page's metrics can be explained by demand being absorbed by a sibling page, calendar-driven demand shifts, or simple low-volume noise. Recommending a "fix" for a page that is not actually declining, but merely consolidating or seasonally quiet, wastes effort and can even prompt a strategist to unnecessarily disturb a page that's fine.
- **Leakage-driven overconfidence:** if the feature window overlaps the target window, or a product decision flag (e.g. `health_score`) slips in as a feature, the model will look artificially strong on validation but fail on real future decisions — this is a costly failure mode because it would not be visible until after the recommendation is acted on.

Given these costs, this project treats the ranked queue as a **decision-support signal for time-limited human review**, not an autonomous action trigger. The pass bar for a usable recommendation is intentionally set above "is this technically a valid score" — it must also survive the consolidation/seasonality/noise check and a leakage audit.

## 7. Why This Is Not Just "Train a Model"

- **The label is not given — it has to be defined and defended.** The starter's `trend_direction == "down"` is an explicitly beginner, current-window proxy. A stronger capstone target (prior 90 days → next 30-day decline/recovery) requires defining magnitude, window, persistence, and minimum-volume thresholds as policy choices — this is a definitional and evidentiary task before it's a modeling task.
- **Ground truth for "should be reviewed" doesn't exist independent of the definition above.** There's no external ledger of "correct" review priorities; evaluation depends on how well the chosen label and features avoid leakage and actually reflect future business-relevant movement, not on maximizing a metric against a fixed answer key.
- **Distinguishing real decline from consolidation, seasonality, or noise is itself the hard part**, and it requires inspecting related content/keyword groups, site-level trends, and volume thresholds — not something a classifier does automatically without a human-defined check.
- **The existing baseline already works somewhat** (Precision@50 = 0.240 isn't zero) — so the real question is whether a learned method adds enough lift to justify the added complexity and validation burden, evaluated at the top-K scale a review team can actually act on, not on generic accuracy.
- **Output only has value if it's interpretable and appropriately gated.** A raw probability score with no reason code and no confidence gating is not something a strategist can safely act on — building that interpretability and evidence-gating is as much of the work as the ranking itself.

## 8. Notes on Language and Confidence

This framing avoids performance or ROI promises. At this stage, the accurate claims are:
- The starter pipeline shows learned ranking *can* outperform a transparent baseline rule on a 30,000-row anonymized slice with client-holdout validation — this is not yet shown on the full warehouse release, and has to be re-earned with proper validation there.
- "Right page to review first" is a prioritization claim under limited review capacity, not a claim that action will cause recovery; causal claims about refresh outcomes are explicitly out of scope without an experimental design.
- The lane, label definition, and feature/target windows above are provisional and are expected to be refined — or possibly changed — as I continue inspecting the warehouse release in Weeks 1–4. Warehouse access is confirmed as of this writing; the usable-client subset (filtered by `access_profile` and history depth) and its resulting row counts are not yet confirmed and should not be assumed from the raw 104-client / 78.8M-row headline figures.
