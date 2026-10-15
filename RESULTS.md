# Experiment log

One entry per experiment: date, configuration, and results. Newest last.

---

## 2026-10-15 · Phase 1 · Data preparation

**Config:** `nl2vis.ipynb` section 2: seed 42, chart types bar/line/scatter, split by database with test 0.1 and val 0.1 targets, 500 candidate splits, 3 sample values per column (cut at 20 characters). nvBench pinned at commit `544fbc21`.

**Source:** nvBench `NVBench.json` has 7,247 visualisations and 25,752 non-empty questions over 152 Spider databases (the README says 25,750).

**Filtering funnel (visualisations / questions)**

| Stage | Bar | Line | Scatter | Pie | Total |
|---|---|---|---|---|---|
| All of nvBench (all 7 chart labels) | 5523 / 19406 | 380 / 1563 | 264 / 1037 | 520 / 1752 | 7247 / 25752 |
| Bar, line, scatter or pie | 5523 / 19406 | 380 / 1563 | 264 / 1037 | 520 / 1752 | 6687 / 23758 |
| Single table | 3033 / 11801 | 267 / 1126 | 169 / 747 | 295 / 1154 | 3764 / 14828 |
| Expressible in our spec | 2251 / 8946 | 99 / 452 | 106 / 475 | 268 / 1068 | 2724 / 10941 |
| Spec reproduces the SQL result | 2251 / 8946 | 93 / 424 | 105 / 473 | 268 / 1068 | 2717 / 10911 |
| **In scope (bar, line, scatter)** | **2251 / 8946** | **93 / 424** | **105 / 473** | 0 / 0 | **2449 / 9843** |

The largest losses are joins (2,382 visualisations), nested queries (541) and binning (660, mostly by weekday or year).

**Converter validation:** 2,717 of 2,724 converted specs (99.7%) give the same chart data as nvBench's SQL run in SQLite. The 7 failures all come from `bike_1.weather`, where SQLite ranks empty-string temperatures above every number. Separately, only 86.6% of the kept visualisations match nvBench's own `vis_obj`, because `vis_obj` was post-processed and partly computed on older data. So the gold chart data is the SQLite execution, not `vis_obj`.

**Splits (by database, no overlap)**

| | train | val | test |
|---|---|---|---|
| databases | 114 | 13 | 13 |
| visualisations | 1819 | 336 | 292 |
| questions | 7115 | 1434 | 1294 |
| line / scatter questions | 286 / 387 | 60 / 29 | 78 / 57 |
| min / max aggregation | 127 / 137 | 25 / 15 | 33 / 39 |

**Schema serialisation:** 250 tables. Length in characters: median 270, 90th percentile 453, maximum 1,519 (`soccer_1/Player_Attributes`).

Files: `results/phase1_*.csv|md`, `data/processed/{train,val,test}.jsonl`, `data/processed/schemas.json`.

---

## 2026-10-15 · Phase 1b · Add binning (supersedes the splits above)

**Change:** the spec gains `"bin": null | "year" | "month" | "weekday"`, so binned nvBench queries (`BIN date BY YEAR`) are now converted instead of dropped. SQLite cannot bin, so a binned spec is kept only if our executor reproduces nvBench's `vis_obj` (allowing for `Thur`/`Sept` labels and zero-filled empty bins). `split_candidates` goes from 500 to 2000. Pie is still out of scope.

**Binned validation:** 289 of 466 binned visualisations kept. Dropped: 73 where nvBench merged years into ranges (`2010~2012`), and 104 where `vis_obj` disagrees with the data. Inspected examples show nvBench computed those on a subset or older data, e.g. a weekday cloud-cover sum of 8 where the full table gives about 1,400. Unbinned specs still reproduce the SQL result in 2,717 of 2,724 cases (99.7%).

**Funnel (visualisations / questions), new rows**

| Stage | Bar | Line | Scatter | Total |
|---|---|---|---|---|
| Expressible in our spec | 2599 / 10118 | 217 / 883 | 106 / 475 | 3190 / 12544 (incl. pie) |
| Reproduces nvBench's result | 2501 / 9796 | 132 / 604 | 105 / 473 | 3006 / 11941 (incl. pie) |
| **In scope** | **2501 / 9796** | **132 / 604** | **105 / 473** | **2738 / 10873** |

Line charts: 93 → 132 visualisations, 424 → 604 questions.

**Splits (by database; largest relative category imbalance in val/test 82%)**

| | train | val | test |
|---|---|---|---|
| databases | 107 | 21 | 14 |
| questions | 8540 | 1217 | 1116 |
| bar / line / scatter | 7763 / 459 / 318 | 1078 / 60 / 79 | 955 / 85 / 76 |
| none / count / sum / avg / min / max | 2108 / 3687 / 1101 / 1360 / 147 / 137 | 132 / 927 / 25 / 89 / 20 / 24 | 192 / 562 / 121 / 193 / 18 / 30 |
| bin year / month / weekday | 214 / 135 / 395 | 42 / 18 / 63 | 46 / 21 / 96 |

---

## 2026-10-15 · Phases 2-3 · Models and evaluation

**Systems:**
- `rule_based`: keywords + RapidFuzz column matching.
- `few_shot_t5`: Flan-T5-base, schema + 4 fixed examples, prompt primed with `chart:`.
- `few_shot_llm`: Qwen2.5-1.5B-Instruct, schema + the same 4 examples as chat turns, JSON repair.
- `ft_no_schema` and `ft_schema`: Flan-T5-base, fully fine-tuned.

T5 models read and write a lossless spec text, because `{`, `}` and `<` are not in T5's vocabulary. All 8,540 training targets survive tokenisation unchanged.

**Fine-tuning config:**
- AdamW, lr 3e-4, linear decay, warm-up 5%, weight decay 0.01.
- Batch 16, 8 epochs, bf16, seed 42.
- Max input 512 / target 128 tokens.
- Best epoch chosen by validation loss.
- Hardware: one RTX 5070 Ti; `ft_schema` took 15.8 min and `ft_no_schema` 8.3 min.

| Run | Best epoch | Best val loss | Val loss by epoch (1→8) | Inputs cut at 512 |
|---|---|---|---|---|
| ft_schema | 1 | 0.0429 | 0.043, 0.048, 0.044, 0.048, 0.050, 0.045, 0.047, 0.054 (flat) | 0 train, 53 val |
| ft_no_schema | 1 | 0.1301 | 0.130 → 0.282 (rising: overfits training schemas) | 0 / 0 |

Training loss drops below 0.01 within the first epoch for both runs.

**Validation (1,217 questions, 21 unseen databases), %**

| System | Chart | x | y | Agg | Valid | Exact | Exec | Raw invalid |
|---|---|---|---|---|---|---|---|---|
| Rule-based | 97.6 | 82.2 | 86.7 | 94.1 | 100.0 | 49.6 | 56.1 | 0.0 |
| Few-shot Flan-T5 | 0.2 | 0.2 | 0.2 | 0.2 | 0.2 | 0.1 | 0.1 | 99.8 |
| Few-shot Qwen2.5-1.5B | 82.8 | 78.1 | 65.2 | 78.1 | 79.3 | 45.1 | 60.4 | 15.5 |
| Fine-tuned, no schema | 98.3 | 61.8 | 91.5 | 96.1 | 62.9 | 49.9 | 52.8 | 0.7 |
| Fine-tuned + schema | 98.3 | 84.8 | 96.9 | 96.6 | 96.9 | 70.0 | 72.5 | 0.7 |

**Test (1,116 questions, 14 unseen databases), %**, plus execution match on the 30 hand-written questions

| System | Chart | x | y | Agg | Valid | Exact | Exec | Hand-written exec | Raw invalid |
|---|---|---|---|---|---|---|---|---|---|
| Rule-based | 98.4 | 75.4 | 78.9 | 86.2 | 100.0 | 39.2 | 42.7 | 30.0 | 0.0 |
| Few-shot Flan-T5 | 0.1 | 0.0 | 0.1 | 0.1 | 0.1 | 0.0 | 0.0 | 0.0 | 99.9 |
| Few-shot Qwen2.5-1.5B | 79.5 | 76.8 | 70.2 | 74.0 | 72.0 | 46.0 | 49.6 | 40.0 | 19.3 |
| Fine-tuned, no schema | 97.8 | 62.3 | 88.2 | 97.0 | 55.8 | 48.7 | 49.9 | 10.0 | 0.9 |
| **Fine-tuned + schema** | 97.7 | **93.7** | **95.1** | 96.7 | 97.0 | **72.3** | **73.6** | 30.0 | 1.2 |

**Significance (exact McNemar, test execution match):** `ft_schema` beats each other system with p < 1e-56.

| ft_schema vs | Only ft_schema correct | Only the other correct |
|---|---|---|
| ft_no_schema | 269 | 5 |
| few_shot_llm | 295 | 28 |
| rule_based | 350 | 6 |

**Findings**
- **Schema awareness is the decisive factor:** +23.7 points execution match over the identical model without schema.
- **Where the gain comes from:** almost entirely validity and column choice. Without the schema the model invents column names: 493 of its 567 test failures are invalid outputs.
- **Few-shot Flan-T5-base cannot follow the output format** (99.9% raw invalid). A leaner prompt variant also failed on a 40-question probe (3 valid, 0 correct).
- **Qwen2.5-1.5B is the only usable few-shot baseline,** but 19.3% of its test outputs are invalid specs:
  - Its JSON *syntax* is always valid, so JSON repair recovered none of them.
  - Instead they break the spec vocabulary: `sort.by` set to a column name rather than "x"/"y" (41), an empty x (41), a column name as `bin` (28), or `y = "*"` without a count (15).
  - Another 8.7% parse but don't run on the table (unknown columns).
- **Hand-written questions:** the fine-tuned model drops to 30% (rule-based also 30%), and Qwen does best at 40%. The fine-tuned model's failures there:
  - it treats cell values as columns ("lunch");
  - it misses filters whose values are not among the 3 sampled values ("Australia", 2007);
  - it bins a text `day` column.
  These failures suggest fine-tuning on nvBench's templated phrasing transfers only partly to informal English. With 30 questions, one question is 3.3 points.

Files: `results/summary_{val,test,handwritten}.md|csv`, `results/results_table.md|csv`, `results/significance_test.csv`, `results/error_counts_test.csv`, `results/error_examples.md`, `results/confusion_test.csv`, `figures/*.png`.

---

## 2026-10-15 · Notebook simplified, extra analyses

**Code:** the notebook was rewritten more compactly (2,238 → 1,596 lines of code, including the new analyses), with the same logic. Processed splits, schemas and all summary tables are byte-identical to the previous run, so models and predictions were reused.

**New figures** (`figures/`): system overview, data funnel, split mix, test vs hand-written, by hardness / chart type / table width / column named verbatim, demo of the best model.

**Breakdowns (test execution match, %; rule-based / Qwen / no schema / schema-aware):**

| Slice | n | Rule | Qwen | No schema | Schema |
|---|---|---|---|---|---|
| x column named verbatim | 575 | 52.9 | 53.7 | 72.2 | 78.8 |
| x column not named | 541 | 32.0 | 45.3 | 26.2 | 68.0 |
| Easy | 408 | 53.7 | 54.9 | 64.0 | 84.3 |
| Medium | 560 | 44.6 | 56.8 | 51.4 | 81.2 |
| Hard | 148 | 5.4 | 8.1 | 5.4 | 14.9 |
| Line charts | 85 | 18.8 | 22.4 | 31.8 | 45.9 |

- **No-schema model:** mostly copies column names from the question (72.2% vs 26.2%).
- **Hard questions:** all have filters and half are binned. Of the schema-aware model's 126 Hard failures, 54 are filters and 44 aggregation or bin.
- **Wide tables (> 8 columns):** the drop is confounded by `bike_1.weather` (0%) and `store_1.invoices` (0%). The wide `driving_school.Customers` table scores 76%.

**Demo (schema-aware model, real CSVs):** 4 of 6 questions correct. It fails on "average apple price per month" (invented column `Apple Price`) and "how many parties came for lunch and dinner" (`lunch` used as a column).

---

## 2026-10-15 · Demo figure and report length

**Demo figure:** it now shows six questions the best model answers correctly, with a tighter layout and a date axis for time series:
- average tip per day;
- smokers vs non-smokers;
- the life-expectancy trend;
- GDP vs life expectancy;
- the AAPL trend;
- maximum life expectancy per continent, sorted.

The two failing informal questions are still covered by the hand-written analysis.

**Report:** shortened to 2,630 words of body text (3,127 with references), within the 2,500–3,500 limit, and saved as `NL2VIS_Report_v3.docx`. It contains 9 notebook figures.
