---
name: llm-model-operations
description: "Use for operational LLM/model workflows: Hugging Face model discovery/download/upload, local llama.cpp/GGUF inference, vLLM serving, and deployment-oriented verification."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [llm, huggingface, llama-cpp, gguf, vllm, inference, serving, deployment]
    related_skills: [evaluating-llms-harness, weights-and-biases]
---

# LLM Model Operations

## Overview

Use this umbrella for operational model work: finding models on Hugging Face, downloading/uploading artifacts, running local GGUF inference with llama.cpp, or serving models at scale with vLLM. Keep training/evaluation separate unless the task explicitly crosses into those areas.

## Routing Table

| Need | Section |
|---|---|
| Search/download/upload models or datasets | Hugging Face Hub |
| CPU/consumer GPU/Apple Silicon local inference | llama.cpp / GGUF |
| OpenAI-compatible high-throughput serving | vLLM |
| Quantization/server troubleshooting | Relevant references under each section |

## Hugging Face Hub

- Use `hf` CLI for login, search, download, upload, and repo metadata.
- Verify license, model card, file sizes, and revision before downloading large artifacts.
- Prefer snapshot downloads with explicit revision for reproducibility.

## llama.cpp / GGUF

- Use for local inference, especially quantized GGUF models and consumer hardware.
- Match model architecture, context length, quantization, and GPU offload to hardware.
- Detailed references migrated under `references/llama-cpp/`.

## vLLM Serving

- Use for high-throughput GPU serving, OpenAI-compatible APIs, batching, and production-ish deployments.
- Verify startup logs, served model name, health endpoint, and a test completion before reporting success.
- Detailed references migrated under `references/serving-llms-vllm/`.

## Verification Checklist

- [ ] Model/revision/license identified
- [ ] Hardware and memory constraints checked
- [ ] Download/server command uses explicit paths and revisions where possible
- [ ] Smoke inference or API call succeeds
- [ ] Final answer includes endpoint/path/model id and caveats
