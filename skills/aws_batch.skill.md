# AWS Batch Executor SKILL

## Prerequisites
- `aws` CLI installed and authenticated
- Config values from bioagent.yaml: region, bucket, batch_queue
- AWS Batch compute environment and job queue must be pre-created

## Job Submission

### 1. Register Job Definition (once per tool)
```bash
aws batch register-job-definition \
  --job-definition-name bioagent-{tool_name} \
  --type container \
  --container-properties '{
    "image": "{docker_image}",
    "vcpus": {cpu},
    "memory": {memory_mb},
    "command": ["/bin/bash", "-c", "{cmd}"],
    "volumes": [
      {
        "name": "work",
        "host": {"sourcePath": "/tmp"}
      }
    ],
    "mountPoints": [
      {
        "sourceVolume": "work",
        "containerPath": "/work"
      }
    ]
  }' \
  --region {region}
```

### 2. Submit Job
```bash
aws batch submit-job \
  --job-name bioagent-{run_id}-{step}-{sample_id} \
  --job-queue {batch_queue} \
  --job-definition bioagent-{tool_name} \
  --container-overrides '{
    "command": ["/bin/bash", "-c", "{actual_cmd_with_params}"],
    "vcpus": {cpu},
    "memory": {memory_mb},
    "environment": [
      {"name": "AWS_DEFAULT_REGION", "value": "{region}"}
    ]
  }' \
  --region {region}
```

Returns: `{ "jobId": "xxx-xxx-xxx" }`

### 3. Check Status
```bash
aws batch describe-jobs \
  --jobs {job_id} \
  --region {region} \
  --query "jobs[0].status"
```

States: SUBMITTED, PENDING, RUNNABLE, STARTING, RUNNING, SUCCEEDED, FAILED

### 4. Get Logs
```bash
# First get log stream name
aws batch describe-jobs \
  --jobs {job_id} \
  --region {region} \
  --query "jobs[0].container.logStreamName" \
  --output text

# Then read logs
aws logs get-log-events \
  --log-group-name /aws/batch/job \
  --log-stream-name {log_stream_name} \
  --limit 100 \
  --region {region}
```

### 5. Cancel Job
```bash
aws batch cancel-job \
  --job-id {job_id} \
  --reason "Cancelled by BioAgent" \
  --region {region}
```

### 6. Terminate Job (force)
```bash
aws batch terminate-job \
  --job-id {job_id} \
  --reason "Terminated by BioAgent" \
  --region {region}
```

## Data Transfer

### Input: Copy to S3 (if not already there)
```bash
aws s3 cp --recursive {local_files} s3://{bucket}/work/{run_id}/input/
```

### Between Steps: Use S3 as shared storage
All jobs read/write to: `s3://{bucket}/work/{run_id}/{step_name}/{sample_id}/`

Commands inside containers use aws cli:
```bash
aws s3 cp s3://{input_path} /work/input/
# ... run tool ...
aws s3 cp /work/output/ s3://{output_path} --recursive
```

### Output: Final results
`s3://{bucket}/output/{run_id}/`

### Download results locally
```bash
aws s3 cp --recursive s3://{bucket}/output/{run_id}/ {local_output_dir}/
```

## Parallel Execution
- Submit all independent jobs at once
- AWS Batch handles scheduling based on compute environment capacity
- Poll batch of jobs:
```bash
aws batch list-jobs \
  --job-queue {batch_queue} \
  --filters name=JOB_NAME,values=bioagent-{run_id}-{step} \
  --region {region}
```

## Array Jobs (for many samples with same command)
When running the same tool on many samples:
```bash
aws batch submit-job \
  --job-name bioagent-{run_id}-{step} \
  --job-queue {batch_queue} \
  --job-definition bioagent-{tool_name} \
  --array-properties '{"size": {num_samples}}' \
  --region {region}
```
Use `AWS_BATCH_JOB_ARRAY_INDEX` inside container to pick sample.

## Cost Awareness
- Use Spot instances in compute environment for cost savings
- Use Fargate for small jobs (no EC2 management)
- Right-size memory/cpu to avoid waste

## Error Patterns
- RUNNABLE stuck → compute environment has no capacity, check instance types
- CannotPullContainerError → ECR auth issue or image not found
- OutOfMemoryError → increase memory in container overrides
- Essential container exited → check logs for tool-specific error
