# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Continual Pre-Training (CPT) / Domain Adaptation Pre-Training (DAPT) pipeline for the Qwen3.5-4B model, specializing it on telecom billing domain knowledge (Kenan BP billing system). Uses Unsloth for accelerated LoRA training with 4-bit quantization.

## Development Workflow

This codebase is a local copy of the actual project running on a Linux VM with an NVIDIA L40S GPU. The current Windows machine cannot run any of the training/inference code (no GPU, no CUDA). The workflow is:

1. Edit and debug code here in this local copy.
2. Copy the changed files to the Linux VM.
3. Run the code on the VM.
4. Copy errors/logs/output back here for debugging.

**Do not attempt to run `train.py`, `infer.py`, `perplexity_test.py`, or any GPU-dependent code locally.** When the user pastes error output, it originates from the VM, not this machine.

## Commands

```bash
# Training (requires GPU with CUDA 12.8, e.g. L40S)
python train.py

# Inference / domain knowledge QA test (runs offline, uses cached weights)
python infer.py

# Perplexity evaluation (compares base vs fine-tuned model)
python perplexity_test.py

# Tokenizer expansion validation (run after expand_vocab.py)
python test_expanded_tokenizer.py
```

There is no formal test framework (pytest/unittest). Tests are standalone scripts with exit code 0/1.

## Architecture

### Training Pipeline (train.py)

1. Loads `.env` (HF_TOKEN, MODEL_NAME) and `config.yaml` (hyperparams, paths)
2. Loads Qwen3.5-4B via `FastVisionModel.from_pretrained()` -- **not** `FastLanguageModel`
3. Applies LoRA to language layers only (`finetune_vision_layers=False`)
4. Loads JSONL dataset from `data/dataset.jsonl`, tokenizes with base tokenizer
5. Trains with HuggingFace `Trainer`, saves LoRA adapters + tokenizer to `outputs/model/`

### Critical Unsloth/Qwen3.5 Quirk

Qwen3.5-4B is classified as a Vision-Language model in Unsloth, even for text-only work. This means:

- **Always use `FastVisionModel`**, never `FastLanguageModel`
- The tokenizer returned is a VL processor wrapper. Extract the real tokenizer via: `base_tok = getattr(tokenizer, "tokenizer", tokenizer)`
- Pass this base tokenizer (not the VL processor) to `DataCollatorForLanguageModeling` and for all text encoding/decoding

### Configuration

- `config.yaml` -- All training hyperparameters, LoRA config, and file paths
- `.env` -- `HF_TOKEN` (required for model download), `MODEL_NAME` (defaults to `unsloth/Qwen3.5-4B`)

### Key Paths

| Path | Purpose |
|------|---------|
| `data/dataset.jsonl` | Training corpus (~55K JSONL lines, `"text"` field) |
| `outputs/model/` | Saved LoRA adapters + tokenizer after training |
| `outputs/checkpoints/` | Mid-training checkpoints |
| `outputs/expanded_tokenizer/` | Vocabulary-expanded tokenizer + `expansion_report.json` |
| `logs/` | Training logs, loss curve JSON, test logs |

### Evaluation

- **perplexity_test.py** -- Compares base vs fine-tuned perplexity on in-domain and out-of-domain text. Success: in-domain PPL drops >10%, out-of-domain PPL change <10%.
- **infer.py** -- Runs 10 domain-specific QA prompts (Kenan BP abbreviations/modules). Uses `enable_thinking=False` in Qwen3.5 chat template for direct answers.
- **test_expanded_tokenizer.py** -- Validates vocab expansion against `expansion_report.json`. Checks per-term fertility reduction and global vocab size consistency.

## Dependencies

Python with pip. Key packages: `torch` (CUDA 12.8), `transformers`, `unsloth`, `peft`, `bitsandbytes`, `datasets`. Full list in `requirements.txt`.
