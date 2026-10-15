# NL2VIS: natural language to data visualisation

UTS Advanced NLP, Assessment 3 (group project). The system turns a plain-English question about a table into a JSON chart specification and a chart. It asks whether **schema-aware fine-tuning** of Flan-T5 beats a rule-based baseline and few-shot prompting on [nvBench](https://github.com/TsinghuaDatabaseGroup/nvBench).

## Layout

Everything lives in one notebook, `nl2vis.ipynb`: data, models, training, evaluation, figures and the demo.

- **Part A** (sections 1-18) only defines functions.
- **Part B** (sections 19-30) runs the pipeline:
  - download and convert nvBench, with self-checks;
  - build the hand-written test set;
  - run the baselines;
  - fine-tune both models;
  - evaluate on validation, test and hand-written questions;
  - breakdowns, error analysis, figures, and the demo of the best model.

| System | What it is |
|---|---|
| `rule_based` | keyword rules + fuzzy column matching |
| `few_shot_t5` | Flan-T5-base, prompted with the schema and 4 examples |
| `few_shot_llm` | Qwen2.5-1.5B-Instruct, prompted with the schema and 4 examples, with JSON repair |
| `ft_no_schema` | Flan-T5-base fine-tuned on the question only (ablation) |
| `ft_schema` | Flan-T5-base fine-tuned on question + serialised schema |

The notebook writes all of its outputs next to itself:

```
nl2vis.ipynb
RESULTS.md              experiment log (config, date, metrics)
data/raw/               NVBench.json and database/<db_id>/<db_id>.sqlite (downloaded)
data/processed/         train/val/test/handwritten.jsonl and schemas.json
data/handwritten/       tips.csv, gapminder.csv, stocks.csv (real data bundled with Plotly)
outputs/<system>/       fine-tuned models, run_config.json, log_history.json
results/                summaries, results_table.md/.csv, predictions, scores, error examples
figures/                13 PNG figures for the report (all report figures come from here)
```

## Setup and reproduction

You need Python 3.10 or newer and an NVIDIA GPU with about 16 GB of memory: a local card, or a Colab T4/L4.

```bash
pip install torch==2.7.1 --index-url https://download.pytorch.org/whl/cu128   # skip on Colab
pip install pandas==2.3.1 numpy==2.2.6 matplotlib==3.10.3 rapidfuzz==3.13.0 transformers==4.53.2 \
            accelerate==1.8.1 "sentencepiece>=0.2.1" plotly==6.2.0 scipy==1.18.1 jupyter
jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=-1 nl2vis.ipynb
```

You can also open the notebook and choose **Run All**.

- **First run:** downloads nvBench (about 110 MB) and two models (Flan-T5-base and Qwen2.5-1.5B-Instruct).
- **Seeds:** every random step uses `CONFIG["seed"]`.
- **Reruns:** trained models and predictions are cached, so a rerun skips finished work. Set `OVERWRITE = True` in section 24 to redo everything.

To try your own data, go to section 30 (Demo). Load any CSV into a DataFrame and call `visualise(question, table, predict)`, where `predict = build_system("ft_schema", project)`.
