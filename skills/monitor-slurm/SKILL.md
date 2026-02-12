# Monitor Slurm Jobs

Monitor currently running Slurm jobs and inspect their logs for errors or issues.

## Usage

```
/monitor-slurm
```

## Features

- Lists all running Slurm jobs for your account
- If no running jobs, inspects the k most recently added job IDs from the logs directory and returns a big picture summary
- Automatically finds matching `.out` and `.err` files in `./<project_name>/logs/`
- Displays last N lines of each log file for quick diagnostics
- Shows job status summary

## Options

- `--tail N`: Show last N lines of log files (default: 20)
- `--full`: Show complete log file contents instead of tail
- `--verbose`: Show detailed job information

## Examples

```bash
# Basic monitoring
/monitor-slurm

# Show more log lines
/monitor-slurm --tail 50

# Show full log contents
/monitor-slurm --full

# Verbose output with all job details
/monitor-slurm --verbose
```

## How it works

1. Queries active Slurm jobs using `squeue`
2. If running jobs found:
   - For each job, searches `./<project_name>/logs/` for matching log files
   - Matches files by job ID pattern: `<job_id>_*.out` and `<job_id>_*.err`
   - Displays log contents to help identify issues
   - Provides job status summary
3. If no running jobs:
   - Takes the k most recently added job IDs from the logs directory
   - Inspects their log files
   - Returns a big picture summary of recent job history

## Log File Matching

The skill expects log files named by job ID as:
- `<job_id>_<rest_of_name>.out`
- `<job_id>_<rest_of_name>.err`

For example, if your job ID is `12345`, it will find:
- `12345_*.out`
- `12345_*.err`
