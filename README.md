 
Claude Code skills and configuration for running research code on HPC clusters. For the complete text, see Workflow.md. If you know your way around claude code already, the README should suffice.

The README.md has been written with the help (obviously), of Claude. 

## Overview

This workflow automates the end-to-end process of taking a research repository and getting it running on HPC infrastructure. Designed for clusters with containerized environments (Apptainer/Singularity) and GPU scheduling (Slurm).

## Configuration

**`CLAUDE.md`** - Main configuration file defining:
- Active cluster settings (GPU type, CPU architecture, container runtime)
- Storage paths and resource allocations
- Slurm templates and environment setup
- Cluster-specific gotchas and workarounds

Edit this and other files to match your HPC environment and paths.

## Skills

### `/navigate-repo`
Builds a structured map of a cloned repository by analyzing README, directory structure, and key files. Identifies entry points, dependencies, configs, and data requirements.

**Output:** `REPO_MAP.md`

### `/resolve-deps`
Creates a working environment with all dependencies installed and verified. Handles cluster-specific constraints (ROCm vs CUDA, x86 vs ARM, containerization requirements).

**Prerequisites:** Repo map exists
**Output:** Environment spec, activation commands, `PATCHES.md`

### `/sample-run`
Executes a short verification run to confirm the full pipeline works. Reduces scope (steps, batch size) to target ~5-10 min wall time.

**Prerequisites:** Dependencies resolved, assets verified
**Output:** Sample run summary with loss trends, resource usage, readiness verdict

### `/monitor-slurm`
Monitors running Slurm jobs and inspects their logs for errors. If no jobs are running, summarizes recent job history.

**Options:** `--tail N`, `--full`, `--verbose`

## Workflow

```bash
# 1. Clone the repo
git clone <repo-url> && cd <repo-name>

# 2. Navigate and map the repo
/navigate-repo

# 3. Resolve dependencies
/resolve-deps

# 4. Run a sample job
/sample-run

# 5. Monitor job progress
/monitor-slurm
```

## Cluster Support

Currently configured for:
- **LUMI**: AMD MI250X GPUs (ROCm), x86_64, Singularity
- **Olivia**: NVIDIA GH200 GPUs (CUDA), ARM Grace CPUs, Apptainer

The workflow adapts automatically based on the active cluster in `CLAUDE.md`.

## Notes

- All modifications to repo code are logged in `PATCHES.md`
- Container-first approach - direct conda/pip installs on filesystem are avoided
- Skills are designed to work with poorly-maintained or undocumented repos
- Verification is thorough: imports, GPU visibility, repo module loading

## License

These skills are designed for academic research and HPC environments.
