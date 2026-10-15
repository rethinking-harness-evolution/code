# Rethinking the Evaluation of Harness Evolution for Agents

This repo supports four methodologies:

- Parallel Sampling
- Sequential Refinement
- Harness Evolution
- Harness Scaling

## Setup

Requirements:

- Python 3.13+
- `uv`
- `tmux` (Harness Evolution long runs via `scripts/evolve.sh`)
- access to the benchmark dataset path used in the config files

Install dependencies:

```bash
uv sync
cp .env.example .env   # fill in API keys — see .env.example for the full list
```

Set the required environment variables before running experiments:

```bash
export CLAUDE_API_KEY="your_api_key"      # Scaling / Sequential / Parallel
export E2B_API_KEY="your_e2b_key"
export GITHUB_TOKEN="your_github_token"
```

For Harness Evolution (`configs/experiments/`), also set `GPT54_LLM_*`, `ANTHROPIC_*`,
`ADB_LLM_*`, and `SERPER_API_KEY` as needed — see `.env.example`.

Update the `path` field in the config files under `configs/our-experiments/`
so it points to your local benchmark task directory.

Before the first Terminal-Bench 2.1 rollout, build E2B templates once:

```bash
uv run python scripts/build_templates.py --dataset-dir /path/to/terminal-bench-2-1/tasks -j 16
```

## Run

### 1. Parallel Sampling

First run the pass@k baseline:

```bash
uv run python run_code_agent_baseline.py \
  --config configs/our-experiments/parallel-exp-claude-opus46-k5.yaml \
  --experiment parallel-exp-claude-opus46-k5
```

Then run blind rollout selection:

```bash
uv run python run_blind_rollout_selector.py \
  --experiment-dir experiments/parallel-exp-claude-opus46-k5 \
  --provider claude \
  --name claude-reasoning-high-select-full
```

Finally audit the selected rollouts:

```bash
uv run python audit_blind_rollout_selection.py \
  experiments/parallel-exp-claude-opus46-k5/blind_rollout_selector/claude-reasoning-high-select-full
```

### 2. Sequential Refinement

```bash
uv run python evolve_seq.py \
  --config configs/our-experiments/exp-seq-claude-opus46-local-iter5-c10.yaml \
  --experiment exp-seq-claude-opus46-local-iter5-c10
```


### 3. Harness Evolution

Use `evolve_ahe.py` for configs under `configs/experiments/`:

```bash
uv run python evolve_ahe.py --config configs/experiments/exp-simple-code-gpt54-w.yaml
```

Or launch in tmux via the helper script:

```bash
./scripts/evolve.sh --attach configs/experiments/exp-simple-code-gpt54-w.yaml
```

After a train-split run finishes, evaluate held-out test/val splits:

```bash
uv run python scripts/eval_held_out_iters.py \
  --experiment <train-exp-dir> \
  --test-config configs/experiments/exp-simple-code-gpt54-w-test.yaml \
  --val-config configs/experiments/exp-simple-code-gpt54-w-val.yaml \
  --splits test val
```

Resume or skip evaluation:

```bash
./scripts/evolve.sh \
  --experiment <existing-exp-dir> \
  --start-iteration 3 \
  configs/experiments/exp-simple-code-gpt54-w.yaml
```

### 4. Harness Scaling

```bash
uv run python evolve.py \
  --config configs/our-experiments/exp-per-task-claude-opus46-local-iter5-c10.yaml \
  --experiment exp-per-task-claude-opus46-local-iter5-c10
```

## Outputs

Experiment artifacts are written under `experiments/`. Important summary files
include:

- `baseline_summary.json` and `baseline_summary.md` for Parallel Sampling
- `blind_rollout_selection_summary.json` for blind rollout selection
- `iteration_scores.md` / `iteration_scores.yaml` for evolution runs
- `held_out_scores.md` / `held_out_scores.yaml` after held-out evaluation
- per-task workspaces and trajectory histories under `experiments/<name>/tasks/`

## Notes

- Harness Scaling configs live under `configs/our-experiments/`; Harness Evolution
  configs live under `configs/experiments/`.
- Two evolution entry points:
  - `evolve.py` — Harness Scaling (`configs/our-experiments/`)
  - `evolve_ahe.py` — Harness Evolution (`configs/experiments/`)
- The provided configs use Claude-style or OpenAI-style API settings. To use
  another model, edit the corresponding config file.
- Large experiments can be resumed by reusing the same `--experiment` name and
  passing the appropriate resume options supported by each script.



## Contact Us

Feel free to find more details in the paper. If you have any questions or comments, or notice any issues, contact us at yikewang@cs.washington.edu, tengxiao01@gmail.com, and zhuhs1998@gmail.com. If you found this work helpful, please consider starring the repository and citing us:
```bibtex
@misc{wang2026rethinking,
  title  = {Rethinking the Evaluation of Harness Evolution for Agents},
  author = {Wang, Yike and Zhu, Huaisheng and Hu, Zhengyu and Yuan, Yige and Chen, Zhengyu and Senthil, Shakti and Hajishirzi, Hannaneh and Tsvetkov, Yulia and Smith, Noah A. and Dasigi, Pradeep and Xiao, Teng},
  year          = {2026},
  eprint        = {2607.12227},
  archivePrefix = {arXiv},
  url           = {https://arxiv.org/abs/2607.12227}
}
```
