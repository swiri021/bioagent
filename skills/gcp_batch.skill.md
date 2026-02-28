# GCP Batch Executor SKILL

## Prerequisites
- `gcloud` CLI installed and authenticated
- Config values from bioagent.yaml: project, region, bucket, service_account

## Job Submission

### 1. Prepare Job Config
For each job, create a JSON config file:

```json
{
  "taskGroups": [
    {
      "taskSpec": {
        "runnables": [
          {
            "container": {
              "imageUri": "{docker_image}",
              "commands": ["/bin/bash", "-c", "{cmd}"],
              "volumes": ["/mnt/disks/output:/output"]
            }
          }
        ],
        "volumes": [
          {
            "gcs": {
              "remotePath": "{gcs_bucket_path}"
            },
            "mountPath": "/mnt/gcs"
          },
          {
            "deviceName": "output-disk",
            "mountPath": "/mnt/disks/output"
          }
        ],
        "computeResource": {
          "cpuMilli": "{cpu * 1000}",
          "memoryMib": "{memory_mb}",
          "bootDiskMib": "{disk_mb}"
        },
        "maxRetryCount": 1,
        "maxRunDuration": "{timeout}s"
      },
      "taskCount": 1
    }
  ],
  "logsPolicy": {
    "destination": "CLOUD_LOGGING"
  }
}
```

### 2. Submit Job
```bash
gcloud batch jobs submit {job_id} \
  --project={project} \
  --location={region} \
  --config={job_config.json}
```

Job ID naming convention: `bioagent-{run_id}-{step}-{sample_id}`

### 3. Check Status
```bash
gcloud batch jobs describe {job_id} \
  --project={project} \
  --location={region} \
  --format="value(status.state)"
```

States: QUEUED, SCHEDULED, RUNNING, SUCCEEDED, FAILED

### 4. Get Logs
```bash
gcloud logging read \
  "resource.type=cloud_batch_job AND labels.job_uid={job_uid}" \
  --project={project} \
  --limit=100 \
  --format="value(textPayload)"
```

Or:
```bash
gcloud batch jobs describe {job_id} \
  --project={project} \
  --location={region} \
  --format="json(status)"
```

### 5. Cancel Job
```bash
gcloud batch jobs cancel {job_id} \
  --project={project} \
  --location={region}
```

### 6. Delete Job (cleanup)
```bash
gcloud batch jobs delete {job_id} \
  --project={project} \
  --location={region} \
  --quiet
```

## Data Transfer

### Input: Copy to GCS (if not already there)
```bash
gsutil -m cp {local_files} gs://{bucket}/work/{run_id}/input/
```

### Between Steps: Use GCS as shared storage
All jobs read/write to: `gs://{bucket}/work/{run_id}/{step_name}/{sample_id}/`

### Output: Final results location
`gs://{bucket}/output/{run_id}/`

### Download results locally
```bash
gsutil -m cp -r gs://{bucket}/output/{run_id}/ {local_output_dir}/
```

## Parallel Execution
- Submit all independent jobs at once (GCP Batch handles scheduling)
- Poll all running jobs in a single loop:
```bash
gcloud batch jobs list \
  --project={project} \
  --location={region} \
  --filter="name~bioagent-{run_id}-{step}" \
  --format="table(name, status.state)"
```

## Cost Awareness
- Use `e2-standard` machine types for CPU tasks
- Use `n1-highmem` for memory-intensive tasks (STAR, BWA)
- Consider spot/preemptible VMs for non-urgent jobs:
  Add to job config:
  ```json
  "allocationPolicy": {
    "instances": [{
      "policy": {
        "provisioningModel": "SPOT"
      }
    }]
  }
  ```

## Error Patterns
- RESOURCE_EXHAUSTED → region quota exceeded, try different region or wait
- PREEMPTED → spot VM was reclaimed, auto-retry
- IMAGE_NOT_FOUND → check docker image URI
- BOOT_DISK_TOO_SMALL → increase bootDiskMib
