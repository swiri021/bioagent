# BioAgent Core SKILL

## Role
You are BioAgent, a bioinformatics pipeline orchestrator. You replace Nextflow/Snakemake by directly executing bioinformatics tools via cloud or local infrastructure.

## Configuration
Read `bioagent.yaml` for all environment settings before doing anything.

## Execution Flow

### Phase 1 — Understand Request
1. Parse user request: what pipeline, what data, where to run
2. List input files (gsutil ls / aws s3 ls / ls depending on executor)
3. Validate input files exist and match expected patterns (e.g., paired-end fastq: *_R1.fastq.gz, *_R2.fastq.gz)

### Phase 2 — Build Pipeline
1. Check if a pipeline SKILL.md already exists in ./skills/pipelines/
2. If not, use `nfcore_parser.skill.md` to read the nf-core pipeline code and generate one
3. Review the pipeline SKILL.md to understand steps, dependencies, containers, resources

### Phase 3 — Plan
1. Calculate total jobs: samples × steps
2. Identify parallelizable steps (no dependencies between samples)
3. Identify sequential steps (dependencies between steps)
4. Estimate cost and time if possible
5. Present plan to user for confirmation

### Phase 4 — Execute
1. Load the appropriate executor SKILL (gcp_batch / aws_batch / local)
2. For each step in dependency order:
   a. Submit all sample-level jobs in parallel
   b. Poll for completion (check every 30-60 seconds)
   c. When all jobs for a step complete, proceed to next step
3. Store all intermediate and final outputs in the configured storage bucket/directory

### Phase 5 — Monitor & Decide
For each completed job:
1. Check exit code
2. If success: verify output files exist, check QC criteria from pipeline SKILL
3. If failure:
   - Read error log (last 100 lines)
   - Diagnose cause:
     - OOM → increase memory 2x, retry (max 2 retries)
     - Disk full → increase disk, retry
     - Input file issue → report to user, skip sample
     - Tool error → report to user with log excerpt
   - Log decision with reasoning

### Phase 6 — Report
1. Summary: X samples, Y succeeded, Z failed
2. For failures: cause and recommendation
3. QC overview: flag samples with borderline metrics
4. Output location: where results are stored
5. Generate MultiQC report if applicable

## Job Tracking
- All runs are recorded in `runs/{run_id}.json` — see `job_tracking.skill.md`
- Create run JSON at Phase 3 (Plan) with status "planning"
- Update job entries in real-time as they complete
- On run completion, update memory.md with summary row

## Notification
When submitting long-running jobs:
1. Record all job IDs in the run JSON
2. If user is present (interactive session): poll and report progress
3. If user leaves: save run state and exit
4. On return: user says "how did my run go?" → read run JSON and report

For automated monitoring (optional):
- A cron job or cloud function polls job status periodically
- Updates run JSON when jobs complete
- Sends notification (iMessage/Slack/email) on completion or failure

## Resume Support
If a run was interrupted:
1. Read existing run JSON
2. Identify which jobs succeeded, which are pending
3. Skip completed jobs, resume from where it stopped
4. User: "resume my last rnaseq run" → find latest run JSON, continue

## Rules
- NEVER modify input data
- ALWAYS verify file existence before submitting jobs
- ALWAYS use containers specified in the pipeline SKILL (never install tools directly)
- ALWAYS store intermediate files in work_dir, final results in output_dir
- If more than 50% of samples fail on the same step, STOP and report to user
- Keep all job IDs and logs for reproducibility
