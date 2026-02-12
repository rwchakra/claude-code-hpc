'''
name: resolve-deps
description: Given a repo map and README.md of a github repo, this automatically installs and tests the dependencies.
'''
## Goal
Given a cloned repository, create a working environment where all dependencies are installed and verified. Works for any ML/research repo regardless of framework or how well-maintained it is. Adapts to the active cluster defined in CLAUDE.md.

## Prerequisites
- Repo has already been cloned and navigated (repo map exists)
- CLAUDE.md has been read -- you know which cluster is active (LUMI or Olivia)
- You understand the cluster's constraints (see CLAUDE.md for full details)

*WE ARE ON OLIVIA - SO IGNORE LUMI BASED INSTRUCTIONS*
You can find the relevant .sif files on `<cluster_support_path>/container/`. Look for the pytorch
based ones and choose one.

Remember, you will be working in a containerized environment.
## Step 0: Determine Cluster Context

Read CLAUDE.md and identify `ACTIVE_CLUSTER`. This determines everything downstream:

| Decision | LUMI | Olivia |
|---|---|---|
| GPU vendor | AMD (ROCm) | NVIDIA (CUDA) |
| CPU arch | x86_64 | ARM (aarch64) |
| CUDA code works? | NO -- needs ROCm/HIP port | YES |
| Container runtime | Singularity | Apptainer |
| Direct conda/pip? | NO -- must containerize | NO -- must containerize |
| Base containers | LUMI pre-built PyTorch SIF images | NVIDIA NGC images (ARM builds) |
| Container wrapper | lumi-container-wrapper (Tykky) | HPC-container-wrapper |


**CRITICAL CHECK**: If on LUMI, immediately scan the repo for CUDA-specific code:
- Custom CUDA kernels (`.cu` files, `CUDAExtension` in setup.py)
- `nvcc` compilation steps
- NCCL-specific code (needs RCCL on LUMI)
- cuDNN calls (needs MIOpen on LUMI)
- flash-attn (needs AMD fork on LUMI)

If the repo has deep CUDA dependencies, flag this early. Some can be resolved (flash-attn has an AMD port, many PyTorch ops work via ROCm's HIP layer), others may be blockers (custom CUDA kernels with no HIP equivalent).

**If on Olivia**, scan for x86-specific assumptions:
- Pre-compiled binaries or wheels that are x86-only
- Inline assembly or architecture-specific code
- Docker images that are amd64-only

## Step 1: Discover Dependency Sources

Check for these files in priority order. Multiple may exist -- use the most specific one:

1. `environment.yml` or `environment.yaml` (conda)
2. `pyproject.toml` (modern Python)
3. `setup.cfg` + `setup.py` (setuptools)
4. `setup.py` alone (legacy setuptools)
5. `requirements.txt` (pip)
6. `requirements/*.txt` (split requirements)
7. `Pipfile` / `Pipfile.lock` (pipenv)
8. `Dockerfile` or `docker/` (extract pip/conda install lines -- do NOT try to run Docker)
9. `Makefile` or `install.sh` (extract install commands)

If none exist, fall back to Step 1b.

### Step 1b: Infer Dependencies from Source

Only if no dependency file exists:

- Scan all `.py` files for import statements
- Map imports to PyPI package names (note: import name != package name in many cases, e.g. `import cv2` -> `opencv-python`, `import sklearn` -> `scikit-learn`, `import yaml` -> `pyyaml`)
- Check for C extensions or custom CUDA/HIP kernels in `setup.py`, `csrc/`, or `kernels/`
- Produce a `requirements.txt` and note that it was inferred, not author-provided

## Step 2: Assess Version Constraints

Before installing anything, analyze what we're working with:

- **Python version**: Check `python_requires` in setup files, `.python-version`, `runtime.txt`, or README. If unspecified, check the paper's publication date and repo's last active commit date to estimate. Default to Python 3.9 for 2021-2022 era repos, 3.10 for 2023+, 3.8 for older.
- **Framework version + GPU compatibility**: This is where cluster matters most. See Step 2b.
- **Pinned vs unpinned**: If versions are unpinned, this is a risk. Note it but proceed -- we'll catch breakage in verification.
- **Conflicting pins**: Flag any obvious conflicts (e.g., two requirements files pinning different versions of the same package).

### Step 2b: Map Framework to Cluster GPU Stack

**On LUMI (ROCm)**:

The repo almost certainly specifies CUDA versions of PyTorch/TF/JAX. You need to find the ROCm equivalent:

1. Identify the PyTorch version the repo needs (e.g., `torch==2.1.0`)
2. Check which LUMI pre-built containers provide this version or a compatible one: `module spider PyTorch`
3. If an exact match exists, use it. If not, use the closest newer version and note potential API differences.
4. For packages that ship CUDA binaries (flash-attn, triton, xformers, bitsandbytes, etc.), check if:
   - The LUMI container already includes a ROCm-compatible version
   - An AMD fork exists (flash-attn has one, bitsandbytes has ROCm support in newer versions)
   - The package can work without it (maybe it's optional)

**On Olivia (CUDA, ARM)**:

1. Identify PyTorch/TF version needed
2. Find a matching NGC container (`nvcr.io/nvidia/pytorch:xx.xx-py3`) -- ensure it's ARM-compatible
3. For pip packages, check ARM wheel availability. Many scientific packages now have `aarch64` wheels but some don't.

### Common framework version patterns

- PyTorch: Check for deprecated APIs (`torch.no_grad()` vs `@torch.no_grad`, `torch.cuda.amp` vs `torch.amp`). These hint at the expected version.
- TensorFlow: Check for `tf.compat.v1` usage (TF2 with TF1 compat) vs pure TF2 vs pure TF1. This drastically changes the install.
- JAX: Very sensitive to jaxlib version alignment. On LUMI, needs ROCm-compatible jaxlib.
- Hugging Face: Check `transformers` version carefully -- API changes frequently between minor versions.

## Step 3: Choose Environment Strategy

**Decision tree** (follow in order):

### 3a: Can we use a pre-built container as-is?

Check if the LUMI PyTorch container (or NGC container on Olivia) already includes everything the repo needs. The LUMI containers are quite comprehensive -- they often include transformers, DeepSpeed, flash-attn, xformers.

If the repo only needs PyTorch + a few pip packages on top, this is the best path. You'll add extras via a virtual environment overlay.

### 3b: Do we need a custom container?

If the repo needs a very different framework version or has unusual system-level dependencies, build a custom container:

**LUMI**: Use `cotainr` with a LUMI ROCm base image:
```bash
module load LUMI/24.03 cotainr
cotainr build my_container.sif --base-image=<cluster_container_path>/sif-images/<rocm-base> --conda-env=environment.yml
```

**Olivia**: Build from NGC base:
```bash
apptainer build my_container.sif docker://nvcr.io/nvidia/pytorch:xx.xx-py3
```

### 3c: Lightweight Python-only install?

If the repo doesn't need GPU frameworks (e.g., data preprocessing, evaluation scripts), use the container wrapper:

**LUMI**:
```bash
module load LUMI lumi-container-wrapper
pip-containerize new --prefix ./env requirements.txt
# Or: conda-containerize new --prefix ./env environment.yml
```

**Olivia**:
```bash
hpc-container-wrapper pip new --prefix ./env requirements.txt
```

## Step 4: Install Dependencies

### If using pre-built container + virtual environment overlay (most common path):

**LUMI**:
```bash
# Load the container module
module load LUMI/24.03 partition/G
module load PyTorch/<best-matching-version>

# Create a virtual environment for extra packages
# The container's module documentation will tell you how
# Typically: python -m venv --system-site-packages ./venv
# Then: source ./venv/bin/activate

# Install additional requirements, filtering out what the container already has
pip install --no-deps -r requirements_extra.txt
```

Before pip installing inside the container, **filter the requirements**:
- Remove PyTorch, torchvision, torchaudio (already in container)
- Remove flash-attn, xformers, DeepSpeed if already in container
- Remove any CUDA-specific packages that won't work on LUMI
- Keep the rest

**Olivia**:
```bash
apptainer exec --nv <container.sif> bash -c "
    python -m venv --system-site-packages ./venv
    source ./venv/bin/activate
    pip install -r requirements_extra.txt
"
```

### Order of operations for any install path:

1. **Framework first** (via container -- already handled)
2. **Filter requirements** -- remove what the container already provides
3. **Install remaining Python packages** via pip in virtual environment
4. **Handle compilable extensions**: Set `CC`, `CXX` to an appropriate compiler version. On LUMI, `gcc-12`/`g++-12` are usually available in containers.
5. **Install the repo itself** if it has a `setup.py` / `pyproject.toml`: `pip install -e .` (inside the container/environment)

### During installation, watch for:

- Version resolution conflicts from pip -- read the conflict message carefully
- Packages that fail to build from source (system-level deps, compiler issues)
- **LUMI-specific**: any package trying to use `nvcc` or link against CUDA libraries
- **Olivia-specific**: any package failing because no `aarch64` wheel exists (may need to compile from source or find ARM-compatible alternative)

## Step 5: Patch Hardcoded Paths and Assumptions

Many research repos have hardcoded assumptions. Scan for and fix:

- Hardcoded absolute paths (`/home/author/`, `/data/`, `/checkpoint/`)
  - Replace with paths relative to the project or using environment variables
  - On LUMI: point to `<cluster_scratch_path>/` or `<cluster_fast_scratch_path>/`
  - On Olivia: point to cluster scratch equivalent
- Hardcoded GPU device IDs (`cuda:0`, `cuda:3`)
  - On LUMI, device references may need adjustment for GCD layout
- Hardcoded `torch.cuda` calls
  - On LUMI, these actually work (ROCm maps to the CUDA API via HIP) but occasionally `torch.cuda.get_device_name()` returns AMD names which can break string-matching logic in repos
- Hardcoded batch sizes that assume specific GPU memory
- Missing `os.makedirs` for output directories
- Assumptions about working directory
- **LUMI-specific**: Any code that shells out to `nvidia-smi` (use `rocm-smi` instead)
- **LUMI-specific**: `MIOPEN_USER_DB_PATH` must be set to `/tmp/...` (Lustre breaks MIOpen caching)
- **LUMI-specific**: NCCL environment variables need mapping to RCCL equivalents

Log every patch in `PATCHES.md` at the repo root with the original line, the replacement, and why.

## Step 6: Verify Installation

This is the most important step. Do NOT skip any of these.

All verification must run INSIDE the container environment (via `singularity exec` on LUMI or `apptainer exec` on Olivia).

### 6a: Import check
```bash
# LUMI example:
singularity exec $SIF python -c "import <module>"  # for each dependency

# Olivia example:
apptainer exec --nv <container.sif> python -c "import <module>"
```

### 6b: GPU check

**LUMI**:
```bash
singularity exec $SIF python -c "
import torch
print(f'CUDA available: {torch.cuda.is_available()}')
print(f'Device count: {torch.cuda.device_count()}')
print(f'Device name: {torch.cuda.get_device_name(0)}')
# Should show AMD Instinct MI250X or similar
"
```
Note: `torch.cuda.is_available()` returns True on ROCm -- this is expected. ROCm uses the HIP-CUDA compatibility layer.

Also run: `rocm-smi` to verify GPU visibility.

**Olivia**:
```bash
apptainer exec --nv <container.sif> python -c "
import torch
print(f'CUDA available: {torch.cuda.is_available()}')
print(f'Device: {torch.cuda.get_device_name(0)}')
# Should show NVIDIA GH200
"
```
Also run: `nvidia-smi`

### 6c: Repo module import check
```python
# Import the repo's own top-level Python modules
# e.g., if the repo is called "my_model", try: import my_model
# This catches broken __init__.py files, relative import issues,
# and missing intra-repo dependencies
# Do NOT instantiate any models or load any data -- that belongs in verify-assets
```

That's it. If all three checks pass, the environment is ready. Model instantiation, forward passes, and data loading are verified later in the verify-assets skill.

## Step 7: Diagnosis Loop

If any verification step fails, enter this loop (max 5 attempts per issue):

```
while verification fails:
    1. Read the full error traceback
    2. Identify the root cause category:
       a. Missing package -> install it (in the venv overlay, not system)
       b. Version incompatibility -> find compatible version
       c. Missing system library -> check if available in container; if not, may need container rebuild
       d. GPU stack mismatch -> see cluster-specific section below
       e. Custom op build failure -> see cluster-specific section below
       f. Import error in repo code -> likely a bug or missing __init__.py
       g. Shape/dtype mismatch -> likely a config issue
    3. Apply the fix
    4. Re-run the failing verification step
    5. If fix didn't work, try the next most likely cause
    6. If 5 attempts fail, report what was tried and what the blocking error is
```

### LUMI-Specific Failure Modes

- **"CUDA kernel not found" / custom CUDA extension fails**: The repo has CUDA code that needs porting to HIP. Try `hipify-perl` on the `.cu` files. If it's a well-known package, check if an ROCm fork exists.
- **MIOpen errors**: Almost always caused by MIOpen trying to write cache to Lustre. Set `MIOPEN_USER_DB_PATH=/tmp/...`.
- **RCCL failures / multi-GPU hangs**: Need the aws-ofi-rccl plugin. Pre-built LUMI containers include this. If using custom container, you need to bind-mount the host libfabric.
- **"No HIP GPU available"**: Container bindings are wrong. Ensure `--rocm` flag or correct `SINGULARITY_BIND` is set (the EasyBuild module does this for you).
- **Slow startup / hanging on import**: Python environment is on Lustre with too many small files. This is why you must containerize.

### Olivia-Specific Failure Modes

- **"No matching distribution found" for a pip package**: No ARM wheel available. Try `pip install --no-binary :all: <package>` to compile from source. May need build dependencies.
- **Segfault or illegal instruction**: x86 binary running on ARM. Need to find or build an ARM-native version.
- **Container pull fails**: Image is amd64-only. Check NGC for ARM variants or build from ARM base.

### General Gotchas (Both Clusters)

- `numpy` version conflicts: numpy 2.x breaks many older packages. Pin `numpy<2` for repos from before 2024.
- `protobuf` version: older TF/HF code often needs `protobuf<4`
- `setuptools` version: very new setuptools removed `pkg_resources` features some old packages need
- `PIL` vs `Pillow`: never install both
- `cv2` compiled against wrong numpy: reinstall `opencv-python` after pinning numpy
- `triton` version must match PyTorch version exactly in recent releases

## Output

When complete, produce:

1. **Status**: VERIFIED or BLOCKED (with blocking issue described)
2. **Cluster**: Which cluster this was verified on
3. **Environment spec**: Container image used + list of additional packages in venv overlay. Or exported conda env if using cotainr/container-wrapper.
4. **Activation commands**: The exact sequence to reproduce this environment:
   ```bash
   # Example for LUMI:
   module load LUMI/24.03 partition/G
   module load PyTorch/2.7.0-rocm-6.2.4-python-3.12-singularity-20250527
   source ./venv/bin/activate
   # Ready to run
   ```
5. **Patches applied**: Summary of `PATCHES.md`
6. **Portability notes**: If verified on LUMI, what would need to change for Olivia (and vice versa). Key differences: CUDA vs ROCm, x86 vs ARM, container images, SLURM partition names.
7. **Warnings**: Anything that worked but looks fragile (unpinned deps, deprecated APIs, ROCm compatibility layer workarounds, etc.)
