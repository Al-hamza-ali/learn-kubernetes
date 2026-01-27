# Day 9: Jobs and CronJobs

## Overview

Not all workloads run forever. Some tasks need to run once (batch processing, database migrations) or on a schedule (backups, reports). Kubernetes provides **Jobs** and **CronJobs** for these use cases.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKLOAD TYPES                                │
│                                                                  │
│   Long-running (Deployments):                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Web Server ──────────────────────────────────────►      │   │
│   │  (runs forever until stopped)                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   One-time tasks (Jobs):                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Database Migration ────► Done ✓                         │   │
│   │  (runs once, then completes)                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Scheduled tasks (CronJobs):                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Backup ───► Done    Backup ───► Done    Backup ───►    │   │
│   │  (runs on schedule)                                      │   │
│   │         ▲                  ▲                  ▲          │   │
│   │       12:00              12:00              12:00        │   │
│   │       Day 1              Day 2              Day 3        │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Jobs

### What is a Job?

A **Job** creates one or more Pods and ensures that a specified number of them successfully terminate. The Job tracks successful completions and is considered complete when the required number of successes is reached.

### Basic Job Example

```yaml
# simple-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ['sh', '-c', 'echo "Hello from Kubernetes Job!" && sleep 5']
      restartPolicy: Never    # Required for Jobs
  backoffLimit: 4             # Retry 4 times on failure
```

### Job Completion Types

#### 1. Single Completion (Default)

```yaml
# One pod runs to completion
apiVersion: batch/v1
kind: Job
metadata:
  name: single-job
spec:
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'echo "Processing..." && sleep 10']
      restartPolicy: Never
```

#### 2. Multiple Completions

```yaml
# Run the task 5 times (sequentially or in parallel)
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-job
spec:
  completions: 5        # Run 5 times total
  parallelism: 2        # Run 2 at a time
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'echo "Task $JOB_COMPLETION_INDEX" && sleep 5']
      restartPolicy: Never
```

#### 3. Parallel Processing with Work Queue

```yaml
# Workers process items from a queue
apiVersion: batch/v1
kind: Job
metadata:
  name: queue-job
spec:
  parallelism: 3           # 3 workers
  completions: null        # Workers run until queue is empty
  template:
    spec:
      containers:
      - name: worker
        image: my-queue-worker
        env:
        - name: QUEUE_URL
          value: "redis://redis:6379"
      restartPolicy: Never
```

### Job with Indexed Completions

```yaml
# Each pod gets a unique index
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed    # Each pod gets JOB_COMPLETION_INDEX
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "I am worker index: $JOB_COMPLETION_INDEX"
          # Process partition based on index
          sleep $((JOB_COMPLETION_INDEX * 2))
      restartPolicy: Never
```

---

## Job Configuration Options

| Field | Description | Default |
|-------|-------------|---------|
| `completions` | Number of successful completions needed | 1 |
| `parallelism` | Max pods running at once | 1 |
| `backoffLimit` | Retries before marking failed | 6 |
| `activeDeadlineSeconds` | Max time for job to run | No limit |
| `ttlSecondsAfterFinished` | Auto-delete after completion | No auto-delete |

### Practical Job Example: Database Migration

```yaml
# db-migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    description: "Migrate database schema to v2"
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp-migrations:v2
        command: ['./migrate', 'up']
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      restartPolicy: Never
  backoffLimit: 3
  activeDeadlineSeconds: 600    # Timeout after 10 minutes
  ttlSecondsAfterFinished: 3600 # Delete 1 hour after completion
```

### Job with Init Container

```yaml
# job-with-init.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  template:
    spec:
      initContainers:
      - name: download-data
        image: curlimages/curl
        command: ['curl', '-o', '/data/input.csv', 'https://example.com/data.csv']
        volumeMounts:
        - name: data
          mountPath: /data
      containers:
      - name: process
        image: python:3.11
        command: ['python', '/scripts/process.py', '/data/input.csv']
        volumeMounts:
        - name: data
          mountPath: /data
        - name: scripts
          mountPath: /scripts
      volumes:
      - name: data
        emptyDir: {}
      - name: scripts
        configMap:
          name: processing-scripts
      restartPolicy: Never
```

---

## CronJobs

### What is a CronJob?

A **CronJob** creates Jobs on a schedule, using the standard Cron format.

### Cron Schedule Format

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday = 0)
│ │ │ │ │
│ │ │ │ │
* * * * *
```

### Common Cron Patterns

| Schedule | Cron Expression |
|----------|-----------------|
| Every minute | `* * * * *` |
| Every hour | `0 * * * *` |
| Every day at midnight | `0 0 * * *` |
| Every day at 6 AM | `0 6 * * *` |
| Every Monday at 9 AM | `0 9 * * 1` |
| Every 15 minutes | `*/15 * * * *` |
| First day of month | `0 0 1 * *` |
| Weekdays at 8 AM | `0 8 * * 1-5` |

### Basic CronJob Example

```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"    # Every day at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backup/db-$(date +%Y%m%d).sql
            env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_NAME
              value: "mydb"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### CronJob Configuration Options

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scheduled-task
spec:
  schedule: "*/30 * * * *"
  
  # Concurrency policy
  concurrencyPolicy: Forbid    # Allow, Forbid, or Replace
  
  # Deadline for starting job
  startingDeadlineSeconds: 100
  
  # History limits
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  
  # Suspend scheduling
  suspend: false
  
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: task
            image: busybox
            command: ['echo', 'Hello']
          restartPolicy: OnFailure
```

### Concurrency Policies

| Policy | Behavior |
|--------|----------|
| `Allow` | Allow concurrent jobs (default) |
| `Forbid` | Skip new job if previous still running |
| `Replace` | Stop current job, start new one |

---

## Practical Examples

### Example 1: Log Cleanup CronJob

```yaml
# log-cleanup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 3 * * *"    # Daily at 3 AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              echo "Cleaning up logs older than 7 days..."
              find /logs -type f -mtime +7 -delete
              echo "Cleanup complete"
            volumeMounts:
            - name: logs
              mountPath: /logs
          volumes:
          - name: logs
            hostPath:
              path: /var/log/app
          restartPolicy: OnFailure
      backoffLimit: 2
```

### Example 2: Report Generation

```yaml
# weekly-report.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-report
spec:
  schedule: "0 9 * * 1"    # Every Monday at 9 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report
            image: report-generator:latest
            env:
            - name: REPORT_TYPE
              value: "weekly"
            - name: EMAIL_TO
              value: "team@example.com"
            - name: SMTP_SERVER
              valueFrom:
                configMapKeyRef:
                  name: email-config
                  key: smtp-server
          restartPolicy: OnFailure
      backoffLimit: 3
      activeDeadlineSeconds: 3600
```

### Example 3: SSL Certificate Renewal Check

```yaml
# cert-check.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cert-expiry-check
spec:
  schedule: "0 8 * * *"    # Daily at 8 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cert-checker
            image: curlimages/curl
            command:
            - /bin/sh
            - -c
            - |
              EXPIRY=$(echo | openssl s_client -servername example.com \
                -connect example.com:443 2>/dev/null | \
                openssl x509 -noout -enddate)
              echo "Certificate expiry: $EXPIRY"
              # Alert if expiring within 30 days
          restartPolicy: OnFailure
```

### Example 4: Data Sync Job

```yaml
# data-sync.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-sync
spec:
  parallelism: 4
  completions: 4
  completionMode: Indexed
  template:
    spec:
      containers:
      - name: sync
        image: data-syncer:latest
        env:
        - name: PARTITION
          value: "$(JOB_COMPLETION_INDEX)"
        - name: TOTAL_PARTITIONS
          value: "4"
        command:
        - /bin/sh
        - -c
        - |
          echo "Syncing partition $PARTITION of $TOTAL_PARTITIONS"
          ./sync --partition=$PARTITION --total=$TOTAL_PARTITIONS
      restartPolicy: Never
  backoffLimit: 2
```

---

## Commands Reference

```bash
# Jobs
kubectl get jobs
kubectl describe job my-job
kubectl logs job/my-job
kubectl delete job my-job

# Watch job progress
kubectl get jobs -w

# Get pods created by job
kubectl get pods -l job-name=my-job

# View job logs
kubectl logs -l job-name=my-job

# CronJobs
kubectl get cronjobs
kubectl get cj                    # Short form
kubectl describe cronjob my-cron

# Manually trigger a CronJob
kubectl create job manual-run --from=cronjob/my-cronjob

# Suspend a CronJob
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":true}}'

# Resume a CronJob
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":false}}'

# View CronJob history
kubectl get jobs -l parent-cronjob=my-cronjob
```

---

## Best Practices

### 1. Always Set Deadlines and Retries
```yaml
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 600
```

### 2. Use TTL for Auto-Cleanup
```yaml
spec:
  ttlSecondsAfterFinished: 3600
```

### 3. Choose Appropriate Concurrency Policy
```yaml
# For data-sensitive jobs
concurrencyPolicy: Forbid
```

### 4. Set History Limits for CronJobs
```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 1
```

### 5. Use Indexed Jobs for Parallel Processing
```yaml
completionMode: Indexed
```

### 6. Handle Failures Gracefully
```yaml
restartPolicy: OnFailure    # Retry in same pod
# or
restartPolicy: Never        # Create new pod on failure
```

---

## Restart Policies for Jobs

| Policy | Behavior |
|--------|----------|
| `Never` | Create new pod on failure |
| `OnFailure` | Restart container in same pod |

Note: `Always` is not allowed for Jobs.

---

## Summary

| Resource | Purpose | Use Case |
|----------|---------|----------|
| **Job** | Run task to completion | Migrations, batch processing |
| **CronJob** | Run Jobs on schedule | Backups, reports, cleanup |

### Key Concepts

| Concept | Description |
|---------|-------------|
| `completions` | How many times to run |
| `parallelism` | How many pods at once |
| `backoffLimit` | Retry count |
| `activeDeadlineSeconds` | Timeout |
| `ttlSecondsAfterFinished` | Auto-cleanup |
| `concurrencyPolicy` | Handle overlapping jobs |

### Key Takeaways

1. **Use Jobs** for one-time tasks
2. **Use CronJobs** for scheduled tasks
3. **Set appropriate limits** (deadline, retries)
4. **Use indexed jobs** for parallel processing
5. **Configure history limits** to manage resources
6. **Choose the right restart policy** for your use case

---

## Next Steps

Learn about stateful applications and node-level workloads in **[Day 10: StatefulSets and DaemonSets](../day-10/README.md)**!
