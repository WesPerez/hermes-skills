---
name: ml-experiment-lifecycle
description: "Full ML experiment lifecycle: benchmark models with lm-eval-harness, track experiments with Weights & Biases. Covers evaluation, hyperparameter sweeps, artifacts, model registry, and integrations."
version: 2.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [mlops, evaluation, benchmarking, lm-eval, wandb, experiment-tracking, sweeps, model-registry, artifacts]
---
# ML Experiment Lifecycle

Unified skill covering the full ML experiment lifecycle: **model benchmarking** with EleutherAI's lm-evaluation-harness and **experiment tracking** with Weights & Biases (W&B). These two tools together cover "benchmark → track → compare → iterate."

## When to Use

- User wants to benchmark an LLM on MMLU, GSM8K, HumanEval, etc.
- User wants to compare models across standard academic benchmarks
- User wants to track ML training runs, log metrics, visualize learning curves
- User needs hyperparameter optimization (sweeps/grid/bayes)
- User wants to version models, datasets, or artifacts
- User needs to integrate experiment tracking with HuggingFace, Lightning, Keras, etc.

## Reference Files

| File | Covers |
|------|--------|
| `references/benchmark-guide.md` | All 60+ lm-eval tasks (MMLU, GSM8K, HumanEval, BBH, IFEval, LongBench, etc.) |
| `references/custom-tasks.md` | Creating custom evaluation tasks with YAML + Python |
| `references/api-evaluation.md` | Evaluating OpenAI, Anthropic, and local API models |
| `references/distributed-eval.md` | Multi-GPU evaluation, tensor parallelism, vLLM |
| `references/sweeps.md` | Hyperparameter sweeps: grid/random/bayes, distributions, early termination |
| `references/artifacts.md` | Data/model versioning, model registry, lineage tracking |
| `references/integrations.md` | Framework integrations: HF, Lightning, Keras, Fast.ai, XGBoost |

---

## Part 1: Model Benchmarking with lm-evaluation-harness

### Quick Start

```bash
pip install lm-eval
# Benchmark a HuggingFace model:
lm_eval --model hf \
  --model_args pretrained=meta-llama/Llama-2-7b-hf \
  --tasks mmlu,gsm8k,hellaswag \
  --device cuda:0 \
  --batch_size auto
```

### Common Tasks

| Benchmark | What it measures | Command flag |
|-----------|-----------------|-------------|
| MMLU | Broad knowledge (57 subjects) | `--tasks mmlu` |
| GSM8K | Math word problems | `--tasks gsm8k` |
| HumanEval | Python code generation | `--tasks humaneval` |
| HellaSwag | Commonsense reasoning | `--tasks hellaswag` |
| ARC-Challenge | Science reasoning | `--tasks arc_challenge` |
| TruthfulQA | Factuality | `--tasks truthfulqa_mc2` |
| BBH | Advanced reasoning (23 tasks) | `--tasks bbh` |
| IFEval | Instruction following | `--tasks ifeval` |

### Typical Evaluation Suite

```bash
lm_eval --model hf \
  --model_args pretrained=meta-llama/Llama-2-7b-hf \
  --tasks mmlu,gsm8k,hellaswag,arc_challenge,truthfulqa_mc2 \
  --num_fewshot 5 \
  --output_path results/ \
  --log_samples
```

### Training Progress Tracking

Evaluate checkpoints at intervals during training:

```bash
# Small, fast benchmarks for frequent evaluation
lm_eval --model hf \
  --model_args pretrained=/path/to/checkpoint-step-$STEP \
  --tasks gsm8k,hellaswag \
  --num_fewshot 0 \
  --batch_size auto
```

### Faster Evaluation with vLLM

```bash
lm_eval --model vllm \
  --model_args pretrained=meta-llama/Llama-2-7b-hf,tensor_parallel_size=1 \
  --tasks mmlu \
  --batch_size auto
```

vLLM is 5-10× faster than HuggingFace for inference.

### Model Comparison

```bash
for model in meta-llama/Llama-2-7b-hf mistralai/Mistral-7B-v0.1; do
    lm_eval --model hf \
      --model_args pretrained=$model,dtype=bfloat16 \
      --tasks mmlu,gsm8k,hellaswag \
      --num_fewshot 5 \
      --output_path results/$(echo $model | tr '/' '-').json
done
```

### API Model Evaluation

```bash
# OpenAI
lm_eval --model openai-chat-completions \
  --model_args model=gpt-4-turbo,num_concurrent=5 \
  --tasks mmlu,gsm8k

# Anthropic
lm_eval --model anthropic-chat \
  --model_args model=claude-3-5-sonnet-20241022 \
  --tasks mmlu,gsm8k

# Local OpenAI-compatible (vLLM, TGI, Ollama, llama.cpp)
lm_eval --model local-completions \
  --model_args model=llama2,base_url=http://localhost:11434/v1 \
  --tasks mmlu
```

### Custom Evaluation Tasks

Create a YAML file for your own dataset:

```yaml
# my_tasks/simple_qa.yaml
task: simple_qa
dataset_path: data/questions.jsonl
output_type: generate_until
doc_to_text: "Question: {{question}}\nAnswer:"
doc_to_target: "{{answer}}"
metric_list:
  - metric: exact_match
    aggregation: mean
    higher_is_better: true
```

```bash
lm_eval --model hf \
  --model_args pretrained=model \
  --tasks simple_qa \
  --include_path my_tasks/
```

### Pitfalls — Benchmarking

1. **Evaluation too slow** — Use vLLM backend. Also reduces batch size or use 0-shot for faster iteration.
2. **Out of memory** — Use `load_in_8bit=True`, reduce batch size, or use tensor parallelism.
3. **Different results than reported** — Check fewshot count matches (`--num_fewshot 5`), exact task name (`mmlu` not `mmlu_direct`).
4. **HumanEval not executing code** — Install `pip install human-eval` and add `--allow_code_execution`.
5. **Always exclude .git, node_modules, venv** from pygount-style scans (if also using codebase inspection).

---

## Part 2: Experiment Tracking with Weights & Biases

### Quick Start

```bash
pip install wandb
wandb login
```

### Basic Training Loop

```python
import wandb

wandb.init(project="my-project", config={
    "learning_rate": 0.001, "epochs": 10, "batch_size": 32
})

for epoch in range(wandb.config.epochs):
    train_loss, val_loss = train_epoch(), validate()
    wandb.log({"epoch": epoch, "train/loss": train_loss, "val/loss": val_loss})

wandb.finish()
```

### Core Concepts

- **Project** — Collection of related runs
- **Run** — Single training execution
- **Config** — Logged hyperparameters
- **Metrics** — Scalar, image, histogram, table logging
- **Artifacts** — Versioned datasets, models, files with lineage
- **Sweeps** — Hyperparameter optimization (grid/random/bayes)

### Integration Examples

**HuggingFace Transformers:**

```python
from transformers import Trainer, TrainingArguments
training_args = TrainingArguments(
    output_dir="./results", report_to="wandb",
    run_name="bert-finetuning", logging_steps=100
)
trainer = Trainer(model=model, args=training_args, ...)
trainer.train()
```

**PyTorch Lightning:**

```python
from pytorch_lightning.loggers import WandbLogger
wandb_logger = WandbLogger(project="lightning-demo", log_model=True)
trainer = Trainer(logger=wandb_logger, max_epochs=10)
trainer.fit(model, datamodule=dm)
```

**Keras/TensorFlow:**

```python
from wandb.keras import WandbCallback
model.fit(x_train, y_train, callbacks=[WandbCallback()])
```

### Hyperparameter Sweeps

```python
sweep_config = {
    'method': 'bayes',
    'metric': {'name': 'val/accuracy', 'goal': 'maximize'},
    'parameters': {
        'learning_rate': {'distribution': 'log_uniform', 'min': 1e-5, 'max': 1e-1},
        'batch_size': {'values': [16, 32, 64]},
    }
}
sweep_id = wandb.sweep(sweep_config, project="my-project")
wandb.agent(sweep_id, function=train, count=50)
```

### Artifacts & Model Registry

```python
# Log a model
artifact = wandb.Artifact('resnet50-model', type='model', metadata={'accuracy': 0.95})
artifact.add_file('model.pth')
wandb.log_artifact(artifact, aliases=['best', 'production'])

# Use a dataset
dataset = run.use_artifact('training-data:latest')
data = dataset.download()
```

### Pitfalls — W&B

1. **Never log secrets or API keys** to W&B metadata.
2. **Use offline mode** for unstable connections: `os.environ["WANDB_MODE"] = "offline"` then sync with `wandb sync`.
3. **Sweep costs add up** — use `early_terminate` with Hyperband to kill underperforming runs.
4. **Rate limits** on W&B free tier (~1000 runs/hour). Batch logs when possible.
5. **Artifact path naming** — use `team/project/artifact_name:alias` for cross-project access.

## Unified Workflow: Benchmark → Track → Compare

```python
# 1. Run benchmark
import subprocess, json
result = subprocess.run([
    "lm_eval", "--model", "hf",
    "--model_args", "pretrained=my-model",
    "--tasks", "mmlu,gsm8k",
    "--output_path", "results.json"
], capture_output=True)

# 2. Log results to W&B
import wandb
wandb.init(project="model-benchmarks", name="my-model-v1")
with open("results.json") as f:
    data = json.load(f)
for task, metrics in data["results"].items():
    for metric, value in metrics.items():
        if metric != "alias":
            wandb.log({f"{task}/{metric}": value})
wandb.finish()
```

## Verification Checklist

- [ ] `lm_eval --tasks list` shows available benchmarks
- [ ] A single benchmark run completes and produces JSON output
- [ ] `wandb.login()` succeeds (API key configured)
- [ ] A W&B run appears in the dashboard with logged metrics
- [ ] Sweep configuration is tested on a small sample (use `--limit N` first)
- [ ] Artifact uploads and downloads work end-to-end
