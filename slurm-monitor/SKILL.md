---
name: slurm-monitor
description: Monitor and report on Slurm cluster status, job queues, job history, and node resource allocation. Use when the user asks about cluster health, node status, job queues, running jobs, pending jobs, completed jobs, failed jobs, CPU/memory usage on nodes, partition utilization, or any Slurm workload manager monitoring tasks. Also use when the user mentions checking swedev or any other cluster by name, or wants to SSH into a cluster to run Slurm commands.
---

# Slurm Cluster Monitoring

This skill helps you monitor Slurm clusters by running diagnostic commands and generating reports. Commands can run locally (if already on the cluster) or via SSH to a remote cluster.

## Command Prefix

Determine the command prefix based on the user's request:

- **Local execution**: If the user doesn't specify a cluster name, or you're already on the cluster, run commands directly
- **Remote execution**: If the user mentions a cluster name (e.g., "swedev", "check the production cluster"), prefix all commands with `ssh <cluster-name>`

Example: For cluster "swedev", use `ssh swedev "sinfo -N"` instead of just `sinfo -N`.

## Core Monitoring Tasks

### 1. Cluster Status Report

Generate a summary of cluster health by partition and node state.

**Command:**
```bash
sinfo -N -o "%N|%P|%T|%C|%m|%e|%O"
```

**What to report:**
- Count nodes by state (idle/allocated/mixed/down/drain) per partition
- Identify any down or drained nodes and highlight them
- Show partition-level utilization (allocated vs total CPUs)
- Flag any anomalies (e.g., nodes stuck in drain, unexpected down nodes)

**Example output structure:**
```
Cluster Status Summary
======================

Partition: debug
- 4 nodes total
- 4 idle, 0 allocated, 0 mixed
- 0 down/drain (healthy)

Partition: core32
- 10 nodes total
- 3 idle, 0 allocated, 7 mixed
- 0 down/drain (healthy)
- Utilization: ~70% (7/10 nodes active)

Issues Found:
- None
```

### 2. Job Queue Analysis

Analyze pending and running jobs across partitions and users.

**Command:**
```bash
squeue -o "%i|%p|%j|%u|%t|%M|%D|%R|%C|%m|%l" --all
```

**What to report:**
- Total running vs pending jobs
- Breakdown by partition
- Breakdown by user (top users by job count)
- Identify long-running jobs (TIME > threshold like 24h)
- Identify jobs stuck pending with reasons (e.g., JobArrayTaskLimit, Resources, Priority)
- For job arrays, note the array size and concurrency limit

**Example output structure:**
```
Job Queue Summary
=================

Overall: 32 running, 2670 pending

By Partition:
- core32: 28 running, 0 pending
- mini: 4 running, 2670 pending

By User:
- root: 32 running, 2670 pending

Long-running jobs (>24h):
- None

Pending job reasons:
- JobArrayTaskLimit: 2670 jobs (array 36419, limited to 32 concurrent)
```

### 3. Job History Analysis

Analyze completed jobs for success rates, runtime patterns, and failures.

**Command:**
```bash
sacct --parsable --format=JobID,JobName,Partition,User,State,ExitCode,Elapsed,Start,End,AllocCPUS,NodeList -a --starttime=$(date -d '24 hours ago' +%Y-%m-%dT%H:%M:%S)
```

Adjust `--starttime` based on the user's request (default: last 24 hours).

**What to report:**
- Total completed jobs in time window
- Success rate (COMPLETED with ExitCode 0:0 vs FAILED/CANCELLED/TIMEOUT)
- Average runtime for completed jobs
- Identify failed jobs with non-zero exit codes
- Breakdown by user and partition
- Note any patterns (e.g., specific job names failing repeatedly)

**Example output structure:**
```
Job History (last 24 hours)
============================

Total jobs: 1,245
- Completed successfully: 1,240 (99.6%)
- Failed: 5 (0.4%)

Failed jobs:
- Job 36420_15: ExitCode 1:0, User: alice, Node: zj021-swe-11
- Job 36421_8: ExitCode 2:0, User: bob, Node: zj021-swe-14

Average runtime: 42 minutes

By Partition:
- core32: 800 jobs, 99.8% success
- mini: 445 jobs, 99.2% success
```

### 4. Node Resource Inspection

Check detailed CPU and memory allocation for specific nodes.

**Command:**
```bash
scontrol show node <node-name>
```

**What to report:**
- CPUAlloc vs CPUTot (allocated vs total CPUs)
- AllocMem vs RealMemory (allocated vs total memory)
- Current state (IDLE/MIXED/ALLOCATED/DOWN/DRAIN)
- CPULoad (actual load average)
- Any issues: high load with low allocation, memory pressure, drain reason

**Example output structure:**
```
Node: zj021-swe-11
==================

State: MIXED
CPUs: 20/32 allocated (62.5%)
Memory: 120GB/128GB allocated (93.8%)
Load: 0.36 (very low despite allocation)

Partition: core32
Last busy: 2026-03-26T20:22:41

Status: Healthy (low load suggests jobs may be idle/waiting)
```

## Multi-Node Inspection

When the user asks about multiple nodes or a partition, run `scontrol show node` for each and summarize:

```bash
for node in $(sinfo -N -h -p <partition> -o "%N"); do
  scontrol show node $node
done
```

Aggregate the results into a partition-level summary showing total allocated vs available resources.

## Handling Job Arrays

Job arrays appear in `squeue` with notation like `36419_[3328-5999%32]` where:
- `36419` is the array job ID
- `[3328-5999%32]` means tasks 3328-5999 with max 32 concurrent

**Getting accurate job array counts:**

Use `scontrol show job <array_job_id>` to get the full task range and concurrency limit:

```bash
scontrol show job 36419
```

This shows `ArrayTaskId=3383-5999%32` which means:
- Tasks 3383-5999 (2,617 total tasks)
- Max 32 concurrent (`ArrayTaskThrottle=32`)

When reporting, explain this clearly: "Job array 36419 has 2,617 pending tasks, limited to 32 running at once."

## Error Handling

If a command fails (e.g., SSH connection refused, sinfo not found), report the error clearly and suggest:
- Check if you're on the right cluster
- Verify SSH access: `ssh <cluster> hostname`
- Check if Slurm is installed: `which sinfo`

## Output Format

Present reports in clear, structured markdown with:
- Section headers for each category
- Bullet points for key findings
- Tables for detailed breakdowns (if many items)
- **Highlight issues** in bold or with a dedicated "Issues Found" section

Keep reports concise but actionable. If the user asks for a quick status, give a 3-4 line summary. If they ask for details, provide the full breakdown.
