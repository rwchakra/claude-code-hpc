# ML Research Reproduction Workflow

## About

This project helps reproduce ML research papers by automating repo navigation, dependency resolution, asset fetching, and sample runs. The workflow is designed to work across different HPC clusters.

## Active Cluster

Set this to whichever cluster you are currently working on. Skills will read this to determine the correct environment strategy.

### Olivia (Sigma2/NRIS, Norway)

- **GPU**: NVIDIA Grace-Hopper GH200 (Hopper H200 GPUs + ARM Grace CPUs)
- **CPU**: ARM-based Grace CPUs (NOT x86!)
- **GPU Software**: CUDA (standard NVIDIA stack)
- **Container Runtime**: Apptainer/Singularity
- **Scheduler**: Slurm
- **Software Stacks**:
  - `NRIS/CPU` -- software for CPU compute nodes
  - `NRIS/GPU` -- libraries, compilers, tools for GH200 GPUs
  - `NRIS/Login` -- login node tools only (do NOT use for compute)
  - `EESSI/2023.06` -- European software stack
  - `CrayEnv` -- Cray Programming Environment

#### CRITICAL: Environment Setup on Olivia

**DO NOT install conda/pip directly on filesystem.** Use:

1. **HPC-container-wrapper (preferred for Python/conda)**:
   - Equivalent to LUMI's lumi-container-wrapper / Tykky
   - Wraps conda/pip installations into containers with wrapper scripts
   - `hpc-container-wrapper conda new --prefix <dir> env.yml`
   - `hpc-container-wrapper pip new --prefix <dir> requirements.txt`

2. **Pre-built AI containers**:
   - Download from NVIDIA NGC or other registries
   - Run with: `apptainer exec --nv <container.sif> <command>`
   - `--nv` flag enables GPU passthrough

3. **Custom Apptainer containers**:
   - Build from NVIDIA base images (e.g., `nvcr.io/nvidia/pytorch:xx.xx-py3`)
   - Must use ARM-compatible images (Grace CPU is ARM, not x86!)

#### Olivia-Specific Gotchas

- **ARM architecture**: Many pip packages don't have pre-built ARM wheels. Expect more compilation from source.
- Grace-Hopper has unified memory between CPU and GPU -- some repos may benefit from this but most won't use it by default.
- Check that any container images you pull are `linux/arm64` compatible.
- EESSI stack may have GPU-enabled software: check `module load EESSI/2023.06` then search.

#### Olivia Slurm Template

```bash
#!/bin/bash
#SBATCH --account=<project_code>
#SBATCH --partition=accel          # GPU partition (verify name)
#SBATCH --nodes=1
#SBATCH --gpus-per-node=1          # Adjust as needed
#SBATCH --time=00:30:00
#SBATCH --output=logs/%j.out
#SBATCH --error=logs/%j.err

module load NRIS/GPU

apptainer exec --nv <container.sif> python train.py
```

---

## General Preferences

- **Preferred framework**: PyTorch (most repos I work with are PyTorch-based)
- **Logging**: Weights & Biases (wandb) when available, otherwise tensorboard
- **Storage for datasets**: `<cluster_storage_path>/datasets/<project_code>/<username>/` on Olivia
- **Storage for checkpoints**: `<cluster_storage_path>/checkpoints/<project_code>/<username>`
- **HuggingFace token and cache location**: `<cluster_cache_path>/hf_cache`
- **Always log patches**: Any modification to repo source code goes in `PATCHES.md`
- **Sample run duration**: Target ~5-10 minutes wall time for verification runs

---

## Skill Permissions

The following skills are authorized to run without requesting confirmation:

### `/monitor-slurm`

**Purpose**: Monitor currently running Slurm jobs and inspect their logs for diagnostics

**Allowed operations** (no confirmation needed):
- Execute `squeue` to query running jobs
- Run `find` to search for log files in `./Talk2DINO/logs/`
- Read `.out` and `.err` log files from the logs directory
- Display log contents to user for analysis
- Check file modifications and sizes

**Scope**: Read-only inspection of job status and logs; no modifications to jobs or files
