# Adversarial Geopolitical Risk Agent

**AI-driven energy supply chain resilience: a two-tier, dual-agent (Hawk / Dove) risk engine for India's crude oil import corridors.**

The pipeline turns millions of noisy GDELT news events, vessel (AIS) data and oil-price fundamentals into a per-corridor disruption risk score. A cheap quantitative tier runs continuously. An expensive LLM debate (an *Escalation* analyst vs a *De-escalation* analyst, then a neutral *Adjudicator*) runs only on the ambiguous cases.

> **Status:** research prototype / hackathon-style demo. All numbers below come from a 13-month backfill (Jul 2025 – Jul 2026). They are pipeline diagnostics, **not** a validated forecasting accuracy claim. See [Limitations](#limitations--known-issues).

---

## Table of contents

1. [Why this exists](#why-this-exists)
2. [Architecture](#architecture)
3. [Repository layout](#repository-layout)
4. [Notebook guide (run order)](#notebook-guide-run-order)
5. [Data requirements](#data-requirements)
6. [Installation](#installation)
7. [Configuration](#configuration)
8. [Running the pipeline](#running-the-pipeline)
9. [Outputs](#outputs)
10. [How the scoring works](#how-the-scoring-works)
11. [Safety guardrails](#safety-guardrails)
12. [Results snapshot](#results-snapshot)
13. [Limitations & known issues](#limitations--known-issues)
14. [Troubleshooting](#troubleshooting)
15. [Roadmap](#roadmap)

---

## Why this exists

India imports roughly 88% of its crude oil, and a large share of that passes through the Strait of Hormuz and nearby chokepoints. Conventional monitoring struggles to separate a real disruption signal from the news noise around it. This project tackles that in three ways:

- **Noise reduction.** Only about 10% of raw GDELT rows are relevant to energy, shipping or geopolitical risk. A documented, auditable filter funnel removes the rest.
- **Cost-aware design.** Tier 1 is free and always on. Tier 2 (LLM calls) fires only when the quantitative signals are ambiguous.
- **Adversarial reasoning instead of one black-box score.** Two LLM analysts argue opposite sides over the *same* cited evidence, rebut each other once, and a separate adjudicator justifies the final number.

**Corridors scored:** `hormuz`, `bab_el_mandeb`, `suez`, `red_sea`, `persian_gulf`, `india` (the last is a domestic-policy and demand bucket).

---

## Architecture

```mermaid
flowchart LR
    RAW["Raw GDELT CSVs<br/>13 months, 4.86M rows"] --> S1["1. Preprocessing<br/>dedup + keyword / TF-IDF filter"]
    S1 --> CORP[("Relevant corpus<br/>151,922 stories")]

    AIS["Vessel / AIS history"] --> T1
    FUND["Brent / WTI / Indian basket"] --> T1
    CORP --> T1["2. Tier 1 quant scores<br/>news + vessel + fundamentals x sanctions"]

    CORP --> RAG["3. RAG store<br/>chunks, TF-IDF, credibility tiers"]

    T1 --> TRIG{"Trigger?<br/>composite high OR<br/>sub-scores disagree"}
    TRIG -- "no" --> FEED
    TRIG -- "yes (episode)" --> DEB
    RAG --> DEB["4. Tier 2 debate<br/>Hawk + Dove, rebuttal, Adjudicator"]
    DEB --> FEED["7. Unified risk feed"]
    FEED --> ALERTS["Disruption alerts"]

    T1 --> CAL["5. Calibration & backtest"]
    DEB --> CAL
    CAL -. "tuned thresholds" .-> TRIG
```

**Tier 1 (always on, no LLM):** per-corridor News, Vessel and Fundamentals sub-scores, blended into a composite on a 6-hour grid.

**Tier 2 (conditional, local LLM):** triggered when the composite is high or the news and vessel sub-scores disagree. Consecutive triggered points collapse into *episodes*, so the debate runs once per episode rather than once per timestamp.

---

## Repository layout

```
.
├── README.md
├── .gitignore
├── demo.ipynb                          # narrated walkthrough of the finished pipeline
├── gdelt_preprocessing.ipynb           # Step 1: raw GDELT -> relevant story corpus
├── tier1_quant_subscores_complete.ipynb  # Step 2: news / vessel / fundamentals / composite
├── tier2_rag_complete.ipynb            # Step 3: RAG chunk store + retrieval
├── tier3_dove_complete.ipynb           # Step 4: trigger -> Hawk/Dove -> Adjudicator
├── tier4_calibration_backtest.ipynb    # Step 5: calibration, backtest scaffolding
├── tier5_deploy.ipynb                  # Step 5 (apply) + Step 7: unified feed + alerts
└── outputs/                            # generated artifacts (git-ignored)
```

The `tierN_*` filenames do **not** match the step numbers one-to-one. Use the table below.

---

## Notebook guide (run order)

| Order | Notebook | Plan step | What it does | Key outputs |
|---|---|---|---|---|
| 1 | `gdelt_preprocessing.ipynb` | Step 1 | Loads every `gdelt_*.csv`, normalizes URLs, dedups across files into stories, Stage A keyword filter (anchor + context terms), Stage B TF-IDF rescue, corridor reassignment | `gdelt_relevant_corpus.csv` / `.json` |
| 2 | `tier1_quant_subscores_complete.ipynb` | Step 2 | Story severity, rolling News sub-score, Vessel sub-score, Fundamentals sub-score, static sanctions multiplier, composite | `tier1_news_subscore.csv`, `tier1_vessel_subscore.csv`, `tier1_fundamentals_subscore.csv`, `tier1_composite.csv` |
| 3 | `tier2_rag_complete.ipynb` | Step 3 | Credibility tiering, chunking, TF-IDF vector store, retrieval and evidence bundles with injection sanitization | `rag_chunks_metadata.csv`, `rag_tfidf_matrix.npz`, `rag_tfidf_vectorizer.pkl` |
| 4 | `tier3_dove_complete.ipynb` | Step 4 | Trigger flag, episode collapsing, Hawk / Dove round 1 + rebuttal, Adjudicator, guardrails | `step4_trigger_episodes.csv`, `step4_round1_hawk_dove.json`, `step4_final_risk_scores.json` |
| 5 | `tier4_calibration_backtest.ipynb` | Step 5 | Mines backtest candidates, lead-time metric, trigger-threshold sweep, human-review threshold, retrieval-layer injection test | `step5_backtest_candidates.csv`, `step5_lead_time_report.csv`, `step5_threshold_sweep.csv`, `step5_calibrated_config.json` |
| 6 | `tier5_deploy.ipynb` | Step 5 apply + Step 7 | Applies calibrated thresholds, merges Tier 1 + Tier 2 into one feed, exports alerts | `step5_final_calibrated_config.json`, `unified_risk_feed.csv`, `disruption_alerts.csv` |
| 7 | `demo.ipynb` | n/a | Presents the real outputs above; runs one live retrieval guardrail check and an optional LLM briefing and Q&A | (display only) |

Each section of `demo.ipynb` checks that its input file exists and prints a "run notebook X first" message instead of crashing, so a partial pipeline still demos.

---

## Data requirements

Raw data is **not** included in this repository (it is large and git-ignored). You need three inputs:

| Input | Expected file(s) | Required columns |
|---|---|---|
| GDELT news signals | `gdelt_YYYY-MM.csv` (one per month) | `signal_id`, `corridor_id`, `as_of`, `raw_reference` (article URL), `entities`, `value`, `confidence`, ... |
| Vessel / AIS history | `vessel_full_history.csv` | `corridor_id`, `as_of`, `metric` (`congestion_index`, `vessel_count_total`, `tanker_capacity_transiting_dwt`), `value` |
| Fundamentals | `fundamentals_full.csv` | `corridor_id` (`global` / `india`), `as_of`, `metric` (`brent_price`, `wti_price`, `indian_basket_price`), `value` |

There is no historical sanctions feed. The sanctions multiplier is a static `1.0` (see [Limitations](#limitations--known-issues)).

---

## Installation

**Prerequisites**

- Python 3.9+ (3.10+ recommended) and Jupyter (JupyterLab, VS Code or Colab)
- [Ollama](https://ollama.com) running locally, for the Tier 2 debate and the demo briefing
- Enough RAM for a ~4.9M-row CSV load in pandas (several GB for Step 1)

```bash
# 1. Clone and enter the project
git clone <your-repo-url>
cd <repo-folder>

# 2. Create an isolated environment
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install pandas numpy scipy scikit-learn matplotlib jupyter ollama requests beautifulsoup4

# 4. Pull the local LLM used by Tier 2 and the demo
ollama pull mistral
ollama serve                         # keep running in another terminal
```

`requests` and `beautifulsoup4` are only needed if you enable full-text fetching (`ENABLE_FULLTEXT_FETCH = True`) in the RAG notebook. It is off by default.

---

## Configuration

Every notebook has a **Config** cell at the top. Paths are currently hard-coded to the original author's Windows machine (`D:\python time\ET\code\risk_agent\...`), so **edit them before running**:

| Notebook | Variables to change |
|---|---|
| `gdelt_preprocessing` | `INPUT_DIR`, `OUTPUT_DIR` |
| `tier1_quant_subscores_complete` | `GDELT_CORPUS_PATH`, `VESSEL_INPUT`, `FUNDAMENTALS_INPUT` |
| `tier2_rag_complete` | `GDELT_CORPUS_PATH` |
| `tier3_dove_complete` | `TIER1_COMPOSITE_PATH` |
| `tier4_calibration_backtest` | `TIER1_COMPOSITE_PATH`, `FUNDAMENTALS_PATH` |
| `tier5_deploy` | `TIER1_COMPOSITE_PATH` |
| `demo` | `STEP1_CORPUS_JSON`, `TIER1_COMPOSITE_PATH` |

The notebooks from Step 2 onward read and write the relative `outputs/` folder, so **launch them all from the same working directory** so they share it.

**Important tuning knobs**

| Setting | Notebook | Default | Meaning |
|---|---|---|---|
| `GRID_FREQ` | Tier 1 | `"6H"` | Scoring grid per corridor |
| `HALF_LIFE_HOURS` / `LOOKBACK_HOURS` | Tier 1 | `60` / `336` | News recency decay and window |
| `COMPOSITE_WEIGHTS` | Tier 1 | `news 0.40, vessel 0.35, fundamentals 0.25` | Renormalized over the available sources |
| `RUN_LIVE_CALLS` | Tier 2 debate | `True` | Set `False` for a dry run with no LLM calls |
| `MAX_EPISODES_THIS_RUN` | Tier 2 debate | `50` | Cost cap. Set to `None` for all episodes |
| `OLLAMA_MODEL` | Tier 2 debate, demo | `"mistral"` | Any model you have pulled in Ollama |
| `ENABLE_FULLTEXT_FETCH` | RAG | `False` | Fetch article bodies instead of URL-slug pseudo-text |
| `COMPOSITE_TRIGGER_THRESHOLD` | Tier 2 debate | `0.65` | Composite level that triggers a debate |
| `SUBSCORE_DISAGREEMENT_THRESHOLD` | Tier 2 debate | `0.35` (calibrated: `0.45`) | Sub-score spread that triggers a debate |
| `HUMAN_REVIEW_DISAGREEMENT_THRESHOLD` | Tier 2 debate | `0.30` (calibrated: `0.20`) | Hawk-Dove gap that flags human review |
| `HIGH_CRED_SCORE_FLOOR` | Tier 2 debate | `0.30` | Minimum final score when Hawk cites high-credibility evidence |
| `ALERT_SCORE_THRESHOLD` | Deploy | `0.60` | Effective score that creates an alert |

---

## Running the pipeline

Run the notebooks in order (table above), each top to bottom.

1. **Preprocess GDELT.** Long step: roughly 4.9M rows, regex over every row.
2. **Tier 1 scores.** Cheap and deterministic.
3. **Build the RAG store.** Offline TF-IDF, no network unless full-text fetch is on.
4. **Tier 2 debate.** The only step that costs LLM time. Every call is cached under `outputs/step4_cache/`, so an interrupted run resumes where it stopped. Start with `MAX_EPISODES_THIS_RUN = 50`, check the output, then raise or remove the cap.
5. **Calibrate** (`tier4_calibration_backtest`), then **apply and deploy** (`tier5_deploy`).
6. **Present** with `demo.ipynb`.

> **After calibrating**, copy the final values from `step5_final_calibrated_config.json` into the Tier 2 debate notebook's config. Also change `compute_trigger()` there so the sub-score spread uses only `news_subscore` and `vessel_subscore`. Then re-run the debate so the Tier 2 results match the shipped thresholds.

---

## Outputs

All generated files go to `outputs/` and are git-ignored.

| File | Produced by | Description |
|---|---|---|
| `gdelt_relevant_corpus.csv` / `.json` | Step 1 | One row per unique relevant story, with every raw file/row location kept as provenance |
| `tier1_composite.csv` | Step 2 | Composite score plus news / vessel / fundamentals sub-scores and `data_status` per corridor per timestamp |
| `rag_*` (`.csv`, `.npz`, `.pkl`) | Step 3 | Chunk metadata, sparse TF-IDF matrix and fitted vectorizer. Row order of the matrix and CSV must match |
| `step4_trigger_episodes.csv` | Step 4 | Collapsed trigger episodes (max span 7 days) |
| `step4_final_risk_scores.json` | Step 4 | Per-episode Hawk, Dove and Adjudicator results with cited evidence IDs |
| `step5_final_calibrated_config.json` | Step 5 | The thresholds actually shipped, with rationale |
| `unified_risk_feed.csv` | Step 7 | One row per corridor-timestamp: `effective_score` and `score_source` (`tier2_adjudicated` or `tier1_composite_only`) |
| `disruption_alerts.csv` | Step 7 | Rows whose effective score crosses `ALERT_SCORE_THRESHOLD` |

---

## How the scoring works

**News sub-score (per corridor, every 6h).** Each story gets a severity from a keyword-weight table (0.55), normalized TF-IDF similarity (0.30) and normalized mention count (0.15). Normalization uses a trailing 90-day window, so there is no look-ahead. Stories are averaged with exponential decay (60h half-life, 14-day window) and multiplied by an outlet-corroboration factor (0.5 to 1.0, saturating at 8 distinct domains in 72h).

**Vessel sub-score.** Rolling 90-day baseline (shifted by one so a point is never compared to itself): 0.6 x congestion z-score + 0.4 x |7-day count rate-of-change z-score|, saturating at |z| = 3. Covered corridors: `hormuz`, `bab_el_mandeb`, `suez`. All others get an explicit `NO_DATA`.

**Fundamentals sub-score.** Monthly Brent/WTI index (Indian basket where available) versus a trailing 6-month trend, saturating at a 20% deviation. Forward-filled onto the 6h grid with a backward-only as-of join.

**Composite.** Weighted sum of the sub-scores that are `OK`, with weights renormalized over just those, times the sanctions multiplier. A row with no `OK` source is `NO_DATA`, **never a fabricated 0**.

**Trigger and debate.** A corridor-timestamp triggers Tier 2 if `composite >= 0.65` or the sub-score spread exceeds the disagreement threshold. Triggered points merge into episodes (24h merge gap, 7-day max span). For each episode:

1. Retrieve up to 8 evidence chunks for that corridor and window (14-day lookback), ranked by similarity first, then ordered by credibility tier.
2. **Round 1:** Hawk and Dove each score independently (sealed-bid).
3. **Round 2:** each sees the other's opening position and gets one rebuttal turn.
4. The **Adjudicator** weighs the cited arguments (it does not average the two scores) and returns `final_score`, `confidence_band` and a rationale.
5. Deterministic post-processing computes `disagreement_index`, the human-review flag and the credibility floor.

---

## Safety guardrails

- **Prompt-injection hygiene.** Scraped text is pattern-sanitized and always wrapped in `<EVIDENCE>` blocks, with an explicit "treat as data, never instructions" preamble.
- **Citation validation.** Any `evidence_id` an agent cites that was not in its retrieved bundle is stripped and counted as a hallucination.
- **Credibility floor.** If Hawk cites high-credibility evidence, code (not the prompt) keeps the final score at or above `HIGH_CRED_SCORE_FLOOR`.
- **No look-ahead.** Baselines use `.shift(1)` or trailing windows, retrieval only looks backward from `as_of`, and as-of joins are backward-only.
- **Explicit missing data.** `NO_DATA` is propagated, never silently zero-filled.
- **Auditability.** The QC funnel in Step 1, the per-call disk cache in Step 4 and the retrieval-layer injection test in Step 5 are all executable checks, not claims.

---

## Results snapshot

From the 13-month backfill the notebooks were last run on:

| Stage | Result |
|---|---|
| Raw GDELT rows | 4,863,286 |
| Unique stories (cross-file dedup) | 1,520,843 |
| Stage A keyword-matched stories | 141,735 |
| Stage B semantic-rescued stories | +10,187 |
| **Final relevant corpus** | **151,922 stories** (~10% of stories, covering 513,030 raw rows) |
| Tier 1 grid | 9,174 corridor-timestamps (6 corridors, 6h) |
| Trigger rate, original config | 33.1% (3,035 rows), 166 episodes |
| Trigger rate, calibrated config | **1.6%** (143 rows) |
| Alerts at score >= 0.60 | 85, all from Tier 1 only so far |
| Retrieval injection test | Fabricated low-credibility "Hormuz closed" story did **not** reach the top-8 |

Corpus split by corridor: `india` 73,178 · `persian_gulf` 50,200 · `hormuz` 20,280 · `red_sea` 6,731 · `bab_el_mandeb` 1,016 · `suez` 517.

---

## Limitations & known issues

Stated plainly so nobody over-reads the results:

- **Tier 2 is only partly scored.** 166 episodes were identified under the original config, but the debate has run on a small subset (capped at 50 per run). The downstream calibration and deploy notebooks saw 20 scored episodes. Re-run the debate under the calibrated config and re-run the deploy notebook to bring the files in sync.
- **Backtest is unlabeled.** `step5_backtest_candidates.csv` has blank `is_genuine_event` / `expected_severity` columns for you to fill. Until then there are no precision/recall numbers. The only lead-time result so far is 2 of 24 corridor-shock pairs (12 and 27 days), which is too small to conclude anything.
- **No article text.** Filtering and retrieval use a URL-slug plus entities pseudo-text, and TF-IDF is a proxy for real embeddings. Both are drop-in upgrades once full text or a sentence-embedding model is available.
- **Thin signal coverage.** Vessel data covers only 3 of 6 corridors. Fundamentals are monthly (Brent/WTI). Sanctions are a static `1.0` multiplier (no historical feed).
- **Data caveat.** The Mar–Apr 2026 Brent spike in the fundamentals file was flagged in the project notes as possibly synthetic. Confirm it before treating it as a real event.
- **Small local LLM.** Mistral via Ollama is cheap but weak. Sample debates show loose reasoning (for example, migrant-incident evidence treated as a vessel disruption signal), so treat Tier 2 scores as exploratory.
- **Unvalidated guardrail constants.** `HIGH_CRED_SCORE_FLOOR = 0.30` and the hand-written keyword severity weights are assumptions, not fitted values.
- **Windows path bug in Step 1.** `gdelt_preprocessing.ipynb` builds `source_file` with `f.split('/')[-1]`, which leaves the full path on Windows. Use `os.path.basename(f)` so provenance stays portable.
- **Not built (by design):** Knowledge Graph, Procurement Orchestrator and Reserve Optimisation agents. They need real supplier, refinery, route and reserve data that this project does not have. `unified_risk_feed.csv` is the intended input if they are added.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `FileNotFoundError` / `No files matched ...` | A hard-coded `D:\...` path. Update the Config cell (see [Configuration](#configuration)) |
| `ValueError: Row mismatch: matrix has N rows, chunks_df has M` | `rag_chunks_metadata.csv` was re-sorted or re-filtered after the matrix was saved. Re-run the RAG notebook's export |
| `ConnectionError` / Ollama calls fail | Run `ollama serve` and confirm `ollama pull mistral` finished |
| Debate cell finishes instantly | `RUN_LIVE_CALLS = False`, or every call is already cached in `outputs/step4_cache/` |
| `Invalid frequency: 6H` | Newer pandas dropped uppercase `H`. Set `GRID_FREQ = "6h"` |
| Demo section prints `[MISSING] ... run X first` | Run the earlier notebook that produces that file |
| Memory error in Step 1 | Load fewer months at a time (the file pattern is configurable) or use a machine with more RAM |
| Flat or empty Tier 1 charts | You are probably pointed at a tiny sample corpus instead of the real export |

---

## Roadmap

- [ ] Fill in the backtest labels and report precision, recall and lead time
- [ ] Re-run Tier 2 on all episodes under the calibrated config
- [ ] Replace TF-IDF with sentence embeddings and fetch full article text
- [ ] Add AIS coverage for `red_sea`, `persian_gulf` and `india`
- [ ] Wire a live OFAC/UN/EU sanctions feed into `get_sanctions_multiplier()`
- [ ] Try a stronger LLM for Hawk, Dove and Adjudicator
- [ ] Move paths into a single config file and extract notebook code into a package
- [ ] Add the downstream agents once real supplier and route data exists

---