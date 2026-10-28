# JANUS: Evaluating Goal-Conditioned Distortion in Factually Grounded LLM Communication

This directory contains the code and data for the paper *JANUS: Evaluating
Goal-Conditioned Distortion in Factually Grounded LLM Communication*. It
contains the 160-scenario benchmark, the primary generation runs,
sentence--fact matches, framing judgments, derived metrics, and the code used
for the paper's main and supplemental analyses.


## Contents

- `data/janus.csv`: 160 scenarios across eight domains, with three favorable
  and three adverse facts per scenario.
- `results/`: the twelve primary runs. For each model, `<model>.jsonl` contains
  the paired generations, `<model>_matched.jsonl` contains sentence--fact
  matches, and `<model>_framing.jsonl` contains framing judgments.
- `metrics/`: per-model fact-, output-, scenario-, and category-level metrics,
  plus the aggregate tables and figures in `metrics/radar_charts/`.
- `janus_payloads.py` and `llm_clients.py`: paired prompt construction and
  generation through local Hugging Face models or supported API providers.
- `judge_fact_units.py`, `judge_framing.py`, and `judge_prompts.py`: the two
  model-judge stages and their exact prompts.
- `metrics_pipeline.py`: deterministic computation of selection, emphasis,
  ordering, specificity, and framing metrics.
- `scripts/`: one-command reproduction, significance testing, plotting, and
  release-integrity checks.
- `analysis/`: paper supplements for cross-family judge robustness, prompt
  mitigation, and fact valence sensitivity.

See `MANIFEST.md` for a paper-to-artifact map and `DATA_CARD.md` for the dataset
schema and intended use.

## Setup

Python 3.10 or newer is recommended.

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Local generation and judge runs download model checkpoints unless they are
already cached. API runs read credentials from environment variables:
`OPENAI_API_KEY`, `DEEPSEEK_API_KEY`, or `ANTHROPIC_API_KEY`. Never commit a
`.env` file.

## Reproduce the released metrics and figures

The following command starts from the released raw generations and judge
outputs. It does not call an external model or API.

```bash
python scripts/reproduce_main_results.py
```

This writes a fresh copy to `reproduced_metrics/`, leaving the released
artifacts unchanged. To reproduce the paper's paired sign-flip tests:

```bash
python scripts/analyze_significance.py
```

The paper uses 200,000 Monte Carlo draws. For a quick smoke test, pass
`--draws 1000`.

## Run a new benchmark

This example runs both conditions for Qwen3-8B without thinking:

```bash
python llm_clients.py \
  --provider qwen_local \
  --model Qwen/Qwen3-8B \
  --qwen-thinking false \
  --dtype bfloat16 \
  --batch-size 4 \
  --max-tokens 768 \
  --output results/my_model.jsonl
```

Then evaluate and aggregate the run:

```bash
python judge_fact_units.py results/my_model.jsonl \
  --output results/my_model_matched.jsonl

python judge_framing.py \
  results/my_model.jsonl \
  results/my_model_matched.jsonl \
  --output results/my_model_framing.jsonl

python metrics_pipeline.py \
  results/my_model.jsonl \
  results/my_model_matched.jsonl \
  --framing-jsonl results/my_model_framing.jsonl \
  --output-dir metrics/my_model
```

All stages are resumable by default. The same fact-shuffle seed is keyed by
scenario, so neutral and goal-conditioned prompts receive the same fact order.
