# Skill: Navigate Repo
'''
name: navigate-repo
description: Given a path to a repo, build a map of the repo to reference later.

'''
## Goal
Build a structured understanding of a cloned repo by reading its README first, then surgically inspecting only the files needed to answer what downstream skills require. Produce a repo map document that resolve-deps, fetch-assets, verify-assets, and sample-run all reference

## Prerequisites
- Repo has been cloned
- No environment setup needed -- this is read-only reconnaissance

## Principle
The README is the primary source of truth. Most well-maintained research repos document their setup, data, models, and usage in the README. Read it thoroughly before touching any other file. Only explore the codebase to fill specific gaps the README left.

## Step 1: Read the README

Read `README.md` (or `README.rst`, `README.txt`, `README`) in its entirety. Extract the following, noting what is present and what is missing:

### For resolve-deps:
- [ ] Python version requirement
- [ ] Installation instructions (pip, conda, docker, or manual)
- [ ] Dependency file referenced (requirements.txt, environment.yml, etc.)
- [ ] Framework and version (PyTorch X.Y, TensorFlow X.Y, JAX, etc.)
- [ ] Any special build steps (compiling extensions, custom ops)
- [ ] CUDA version requirement or GPU assumptions

### For fetch-assets:
- [ ] Dataset names and download links/instructions
- [ ] Pretrained model/checkpoint download links
- [ ] HuggingFace model or dataset IDs
- [ ] Expected directory structure for data
- [ ] Any gated/licensed datasets requiring manual access
- [ ] Preprocessing scripts to run after download

### For verify-assets and sample-run:
- [ ] Training command (exact CLI invocation)
- [ ] Evaluation/inference command
- [ ] Config file(s) referenced in those commands
- [ ] Expected results / metrics (for later comparison)
- [ ] Multi-GPU / distributed training instructions
- [ ] Any noted hardware requirements (GPU memory, number of GPUs)

Mark each item as FOUND (with the relevant text/link) or MISSING.

## Step 2: Map Structure

List the directory tree at depth 2. This is just for orientation:

```bash
find . -maxdepth 2 -type f -o -type d | head -100
```

From this, identify:
- Where does source code live? (`src/`, `lib/`, root-level `.py` files, package name directory)
- Is there a `configs/` or `conf/` directory?
- Is there a `scripts/` or `tools/` directory?
- Is there a `data/` or `datasets/` directory (even if empty)?
- Are there multiple README files (subdirectories with their own docs)?

Do NOT read these files yet. Just note their existence.

## Step 3: Fill Gaps

For each item marked MISSING in Step 1, make ONE targeted check. Do not browse broadly.

### If entry point is missing:
Check these files in order, stop at the first match:
1. `train.py`, `main.py`, `run.py`, `run.sh` at repo root
2. `scripts/train.py`, `tools/train.py`
3. Any `.sh` file at root (read it -- it probably calls a Python script)
4. `Makefile` targets

Read only the top ~30 lines (imports, argparse setup) to understand what it takes as arguments.

### If config system is missing:
Check in order:
1. What does the entry point import? (hydra, argparse, omegaconf, absl, fire)
2. If a `configs/` directory exists, what format are the files? (.yaml, .json, .py)
3. What arguments does the entry point accept?

### If dependency info is missing:
Check for files: `requirements.txt`, `environment.yml`, `setup.py`, `pyproject.toml`, `Pipfile`. Note which exist. Do not parse them here -- resolve-deps handles that.

### If data/model info is missing:
1. Check for `scripts/download_*.sh`, `tools/prepare_*.py`, `data/README.md`
2. Check the entry point or config files for default data paths or model paths
3. Check for HuggingFace dataset/model references in the code: grep for `from_pretrained`, `load_dataset`, `hf_hub_download` (targeted grep, not repo-wide)

### If training command is missing:
1. Check `scripts/` for shell scripts that invoke training
2. Check the entry point's argparse for required vs optional arguments
3. Check for a `Makefile` with a train target

## Step 4: Flag Red Flags

Quick checks only -- do not do exhaustive scans:

- **Hardcoded paths**: Grep the entry point and config files only for absolute paths (`/home/`, `/data/`, `/mnt/`)
- **CUDA-only code**: Check if `csrc/`, `kernels/`, or `*.cu` files exist (relevant for LUMI, not relevant for Olivia, where we are.)
- **Dead links**: Spot-check 1-2 download URLs from the README (a HEAD request is enough)
- **Missing pieces**: No README install section, no requirements file, no config files -- note these as risks.

## Output: Repo Map

Produce a file called `REPO_MAP.md` in the repo root with this exact structure:

```markdown
# Repo Map: <repo_name>

## Overview
<One sentence: what this repo does, from the README>

Paper: <title + link if mentioned>
Last commit: <date>
Framework: <PyTorch X.Y / TF / JAX / other>

## Entry Points
- Train: `<path>` -- `<example command if known>`
- Eval: `<path>` -- `<example command if known>`
- Inference: `<path>` -- `<example command if known>`

## Config System
- Type: <argparse / hydra / yaml files / hardcoded / other>
- Location: `<path to config dir or main config file>`
- Key config for training: `<specific config file if identifiable>`

## Dependencies
- File: `<requirements.txt / environment.yml / etc.>`
- Python: `<version>`
- Framework: `<torch==X.Y / etc.>`
- Special builds: `<any custom ops, CUDA extensions, etc.>`

## Assets
### Datasets
| Name | Source | Auth required | Path expected |
|---|---|---|---|
| ... | ... | ... | ... |

### Pretrained Models
| Name | Source | Auth required | Path expected |
|---|---|---|---|

## Red Flags
- <list any issues found>

## Gaps
- <list anything that could not be determined -- these need manual investigation>
```

This document is the input for all downstream skills. Keep it factual and terse. No commentary, no suggestions -- just what was found and where.
