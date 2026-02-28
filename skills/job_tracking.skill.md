# Job Tracking SKILL

## Role
Track all pipeline runs and job statuses using JSON files. This is the source of truth for what has been run, what succeeded, what failed, and why.

## Directory Structure
```
bioagent/
  runs/
    {run_id}.json          ← one file per pipeline run
  memory.md                ← latest run IDs only (quick reference)
```

## Run ID Format
`{YYYY-MM-DD}_{pipeline}_{3-digit-counter}`
Example: `2026-02-27_rnaseq_001`

## Run JSON Schema

```json
{
  "run_id": "2026-02-27_rnaseq_001",
  "pipeline": "rnaseq",
  "executor": "gcp_batch",
  "status": "running",
  "created_at": "2026-02-27T10:30:00Z",
  "completed_at": null,
  "config": {
    "input": "gs://my-bucket/fastq/",
    "output": "gs://my-bucket/output/2026-02-27_rnaseq_001/",
    "reference": "GRCh38",
    "executor": "gcp_batch",
    "region": "us-central1"
  },
  "samples": ["sample01", "sample02", "sample03"],
  "steps": ["fastqc", "trimgalore", "star_align", "salmon_quant", "multiqc"],
  "current_step": "star_align",
  "jobs": {
    "fastqc-sample01": {
      "job_id": "bioagent-001-fastqc-sample01",
      "step": "fastqc",
      "sample": "sample01",
      "status": "succeeded",
      "submitted_at": "2026-02-27T10:31:00Z",
      "completed_at": "2026-02-27T10:35:00Z",
      "exit_code": 0,
      "retry_count": 0
    },
    "fastqc-sample02": {
      "job_id": "bioagent-001-fastqc-sample02",
      "step": "fastqc",
      "sample": "sample02",
      "status": "failed",
      "submitted_at": "2026-02-27T10:31:00Z",
      "completed_at": "2026-02-27T10:34:00Z",
      "exit_code": 137,
      "retry_count": 1,
      "error": "OOM killed",
      "decision": {
        "reasoning": "Exit code 137 indicates OOM. Original memory was 8GB. Retried with 16GB but failed again. Sample file is 2x larger than others.",
        "action": "skip",
        "timestamp": "2026-02-27T10:40:00Z"
      }
    }
  },
  "summary": {
    "total_jobs": 20,
    "succeeded": 18,
    "failed": 1,
    "skipped": 1,
    "notes": "sample02 skipped due to OOM - file may be corrupted or unusually large"
  }
}
```

## Status Values

### Run Status
- `planning` → building execution plan
- `running` → jobs are active
- `completed` → all jobs finished (some may have failed)
- `failed` → >50% jobs failed, run aborted
- `cancelled` → user cancelled

### Job Status
- `pending` → waiting to submit
- `submitted` → submitted to executor
- `running` → executing
- `succeeded` → exit code 0 + outputs verified
- `failed` → non-zero exit code or missing outputs
- `skipped` → skipped after failed retries
- `cancelled` → cancelled by user or system

## How to Use

### Creating a New Run
```bash
mkdir -p bioagent/runs
# Write initial run JSON with status "planning"
```

### Updating Job Status
After each job completes, update the corresponding entry in the jobs object.
Always update `current_step` when moving to the next pipeline step.

### Completing a Run
1. Set `status` to "completed" or "failed"
2. Set `completed_at` timestamp
3. Fill in `summary` with counts and notes
4. Update memory.md

### Querying Past Runs
```bash
# List all runs
ls bioagent/runs/

# Find failed runs
grep -l '"status": "failed"' bioagent/runs/*.json

# Find runs for a specific pipeline
ls bioagent/runs/*_rnaseq_*.json
```

## memory.md Format

Keep this minimal — just a quick reference:

```markdown
# BioAgent Recent Runs

| Run ID | Pipeline | Status | Date |
|--------|----------|--------|------|
| 2026-02-27_rnaseq_001 | rnaseq | completed | 2026-02-27 |
| 2026-02-27_sarek_002 | sarek | running | 2026-02-27 |
| 2026-02-26_atacseq_001 | atacseq | failed | 2026-02-26 |
```

Only keep the last 20 runs here. Older runs still exist in runs/ directory.

## Notification Integration

When a run completes or fails, the monitoring system (cron/cloud function) should:
1. Update the run JSON with final status
2. Send notification via configured channel:
   - iMessage: use iMessage automation
   - Slack: webhook POST
   - Email: SES/SendGrid

Notification message format:
```
🧬 BioAgent Run Complete
Run: 2026-02-27_rnaseq_001
Pipeline: rnaseq
Status: ✅ 9/10 succeeded, ❌ 1 failed
Output: gs://my-bucket/output/2026-02-27_rnaseq_001/
```
