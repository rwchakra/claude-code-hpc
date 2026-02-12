kill: Sample Run

## Goal
Run a short training or inference job on the cluster to confirm the full pipeline works end-to-end. Not a reproduction -- just proof that everything is wired up and compute can proceed.

## Prerequisites

Before doing anything, verify all gates:

1. `REPO_MAP.md` exists in repo root -> navigate-repo completed
2. Environment activates and imports work -> resolve-deps completed
3. Asset manifest shows all required assets DOWNLOADED -> fetch-assets completed
4. Verification report shows VERIFIED status -> verify-assets completed

If any gate fails, stop and report which stage needs to be (re-)run. Do not attempt a sample run with incomplete prerequisites.

## Step 1: Read Inputs

Read these files:

- `REPO_MAP.md` -- Entry Points section (training command), Config System section (config files, arguments)
- `CLAUDE.md` -- Active cluster, Slurm template, project account, partition names
- Verification report -- GPU memory estimate (to set batch size), number of GPUs needed

## Step 2: Determine Run Command

From the repo map's entry point and config system, construct the training command. Modify it for a short sample run:

### Reduce scope:
- **Epochs/steps**: Set to a small number (50-200 steps, or 1 epoch on a subset)
- **Batch size**: Use the verification report's memory estimate. If unsure, start conservative.
- **Data subset**: If the config supports it, limit to a fraction of the dataset. If not, the short step count handles this.
- **Logging**: Keep whatever the repo uses (wandb, tensorboard, print). Don't disable it -- we need the output.
- **Checkpointing**: Disable or set to save only at end. Don't waste time on frequent checkpoints for a sample run.
- **Validation**: Disable or run once at the end. Not needed for a sample run.

### Identify how to set these:
- CLI args (e.g., `--max_steps 100 --batch_size 8`)
- Config file override (e.g., hydra: `training.max_steps=100`)
- Editing a config file directly (last resort -- log in PATCHES.md)

Do NOT change anything that affects model architecture, optimizer choice, or data preprocessing. Only reduce scale.

## Step 3: Create Slurm Script

Build a job script from the cluster template in CLAUDE.md. Fill in:

```bash
#!/bin/bash
#SBATCH --job-name=sample-<repo_name>
#SBATCH --account=<from CLAUDE.md>
#SBATCH --partition=<from CLAUDE.md>
#SBATCH --nodes=<1 unless multi-GPU is required>
#SBATCH --gpus-per-node=<from verification report>
#SBATCH --time=00:15:00
#SBATCH --output=logs/%j.out
#SBATCH --error=logs/%j.err
```

Then add cluster-specific setup:

**LUMI**:
```bash
module load LUMI/24.03 partition/G
module load PyTorch/<version from resolve-deps>

export MIOPEN_USER_DB_PATH="/tmp/$(whoami)-miopen-cache-$SLURM_NODEID"
export MIOPEN_CUSTOM_CACHE_DIR=$MIOPEN_USER_DB_PATH
export ROCM_PATH=/opt/rocm

# Multi-node RCCL settings (if multi-node)
export NCCL_SOCKET_IFNAME=hsn
export NCCL_NET_GDR_LEVEL=PHB

# Activate venv overlay if used
source <path_to_venv>/bin/activate

srun <training command with sample-run overrides>
```

**Olivia**:
```bash
module load NRIS/GPU

apptainer exec --nv <container.sif> bash -c "
    source <path_to_venv>/bin/activate
    <training command with sample-run overrides>
"
```

Ensure `logs/` directory exists before submission:
```bash
mkdir -p logs
```

Save the script as `sample_run.sh` in the repo root.

## Step 4: Submit and Monitor

```bash
JOB_ID=$(sbatch sample_run.sh | awk '{print $4}')
echo "Submitted job: $JOB_ID"
```

Monitor for early failures (first 2-3 minutes):

```bash
# Wait for job to start
while ! squeue -j $JOB_ID | grep -q "R"; do
    sleep 5
done

# Tail output for early errors
sleep 30
tail -50 logs/${JOB_ID}.out
tail -20 logs/${JOB_ID}.err
```

If the job fails within the first minute, it's almost always an environment or path issue -- not a training issue. Check stderr first.

## Step 5: Check Output

After the job completes, verify:

### 5a: Job completed successfully
```bash
sacct -j $JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS
```
State should be COMPLETED, ExitCode should be 0:0.

### 5b: Loss is decreasing
Parse the training output for loss values. Check:
- First reported loss is finite (not NaN, not Inf)
- Loss at end is lower than loss at start
- No NaN appears at any point during training

A flat loss is a warning (learning rate too low, frozen weights) but not a blocker. NaN is a blocker.

### 5c: GPU was utilized
Check for GPU utilization in logs if available (some frameworks report this). Alternatively, if the job ran on LUMI:
```bash
# Check from job output if rocm-smi was called at start
# Or check elapsed time -- if 100 steps of a real model took <10 seconds, GPU probably wasn't used
```

A rough check: the job should not finish suspiciously fast (implying CPU-only execution) or suspiciously slow (implying a bottleneck).

### 5d: No warnings or errors in stderr
Read `logs/${JOB_ID}.err`. Common acceptable warnings:
- Deprecation warnings (note them but not a blocker)
- "Setting OMP_NUM_THREADS" warnings
- cuDNN/MIOpen autotuning messages

Unacceptable:
- OOM errors (reduce batch size)
- RCCL/NCCL timeout (communication issue)
- Segfaults
- Python tracebacks

## Output

Produce a sample run summary:

```markdown
# Sample Run: <repo_name>

- Cluster: <LUMI / Olivia>
- Job ID: <id>
- Status: SUCCESS / FAILED
- Wall time: <elapsed>
- GPUs: <count x type>

## Training
- Steps completed: <n>
- Initial loss: <value>
- Final loss: <value>
- Loss trend: DECREASING / FLAT / DIVERGING / NAN

## Resources
- GPU memory used: <if available>
- GPU utilization: <if available>

## Files
- Job script: `sample_run.sh`
- Stdout: `logs/<job_id>.out`
- Stderr: `logs/<job_id>.err`

## Verdict
<READY or NOT READY for a full run, with reason>

## Full Run Recommendation
- Estimated command: `<full training command without sample-run overrides>`
- Estimated GPUs: <n>
- Estimated wall time: <extrapolation from sample if possible>
- Config changes needed: <any adjustments for full scale>
```
