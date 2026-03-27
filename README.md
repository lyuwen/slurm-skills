# Slurm Skills

Claude Code skills for monitoring and managing Slurm workload manager clusters.

## Skills

### slurm-monitor

Monitor Slurm cluster status, job queues, job history, and node resources.

**Features:**
- Cluster status inspection (node states, partition utilization)
- Job queue analysis (running/pending jobs, job arrays)
- Job history analysis (completion stats, failures)
- Node resource inspection (CPU/memory allocation)
- SSH remote execution support

**Usage:**
```bash
# Install the skill
ln -s $(pwd)/slurm-monitor ~/.claude/skills/slurm-monitor

# Example prompts
"Check the job queue on swedev cluster"
"Show me cluster status with job arrays grouped"
"Inspect node resource allocation on node-01"
```

## Installation

1. Clone this repository
2. Symlink the skill to your Claude Code skills directory:
   ```bash
   ln -s $(pwd)/slurm-monitor ~/.claude/skills/slurm-monitor
   ```

## Testing

Each skill includes evaluation tests in `evals/evals.json`.
