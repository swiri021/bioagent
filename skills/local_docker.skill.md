# Local Docker Executor SKILL

## Prerequisites
- Docker installed and running
- Config values from bioagent.yaml: output_dir, max_cpus, max_memory

## Job Execution

### 1. Run Container
```bash
docker run --rm \
  --name bioagent-{run_id}-{step}-{sample_id} \
  --cpus={cpu} \
  --memory={memory} \
  -v {input_dir}:/input:ro \
  -v {work_dir}/{step}/{sample_id}:/output \
  -v {reference_dir}:/reference:ro \
  {docker_image} \
  /bin/bash -c "{cmd}"
```

### 2. Run Detached (for parallel jobs)
```bash
docker run -d --rm \
  --name bioagent-{run_id}-{step}-{sample_id} \
  --cpus={cpu} \
  --memory={memory} \
  -v {input_dir}:/input:ro \
  -v {work_dir}/{step}/{sample_id}:/output \
  -v {reference_dir}:/reference:ro \
  {docker_image} \
  /bin/bash -c "{cmd}"
```

Returns container ID.

### 3. Check Status
```bash
# Check if running
docker inspect --format='{{.State.Status}}' bioagent-{run_id}-{step}-{sample_id}
# Returns: running, exited

# Check exit code
docker inspect --format='{{.State.ExitCode}}' bioagent-{run_id}-{step}-{sample_id}
# 0 = success, non-zero = failure
```

Or list all BioAgent containers:
```bash
docker ps -a --filter "name=bioagent-{run_id}" --format "table {{.Names}}\t{{.Status}}"
```

### 4. Get Logs
```bash
docker logs bioagent-{run_id}-{step}-{sample_id} 2>&1 | tail -100
```

### 5. Stop Container
```bash
docker stop bioagent-{run_id}-{step}-{sample_id}
```

### 6. Cleanup
```bash
# Remove stopped containers for this run
docker rm $(docker ps -a --filter "name=bioagent-{run_id}" -q) 2>/dev/null

# Prune unused images (optional, ask user first)
docker image prune -f
```

## Data Flow

### Directory Structure
```
{work_dir}/
  {run_id}/
    input/           ← symlink or copy of input files
    {step_name}/
      {sample_id}/   ← output of each step per sample
    output/          ← final results copied here
```

### Input: Symlink to avoid copying
```bash
mkdir -p {work_dir}/{run_id}/input
ln -s {original_fastq_dir}/* {work_dir}/{run_id}/input/
```

### Output: Copy final results
```bash
cp -r {work_dir}/{run_id}/{final_step}/ {output_dir}/{run_id}/
```

## Parallel Execution

### Resource Management
- Track total allocated CPUs and memory
- Before submitting: check if enough resources available
  ```
  allocated_cpus + new_job_cpus <= max_cpus
  allocated_memory + new_job_memory <= max_memory
  ```
- If not enough resources: wait for running jobs to finish, then submit

### Parallel Launch Script
For a batch of jobs on the same step:
```bash
# Submit all that fit within resource limits
for sample in {sample_list}; do
  docker run -d --rm \
    --name bioagent-{run_id}-{step}-${sample} \
    ...
done

# Wait for all to complete
while docker ps --filter "name=bioagent-{run_id}-{step}" -q | grep -q .; do
  sleep 10
done
```

## Resource Limits
- Respect max_cpus and max_memory from config
- If a job requests more than available, warn user
- Default timeout: 24 hours per job
  ```bash
  timeout 86400 docker run ...
  ```

## Error Patterns
- "no space left on device" → clean work_dir or increase disk
- "OOM killed" (exit code 137) → increase --memory
- "image not found" → docker pull {image} first
- "permission denied" → check volume mount permissions
- Exit code 139 (segfault) → likely tool bug, report to user
