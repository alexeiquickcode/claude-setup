---
name: infra-patterns
description: ML infrastructure patterns for high-throughput, zero-downtime systems. Apply when writing IaC, provisioning compute, or architecting ML systems.
---

# ML Infrastructure Patterns

## Philosophy

**Reliability and performance first.** Systems cannot fail. Design for 10-50k r/s from day one. Cost optimization comes after the system is bulletproof.

Reference implementations: Google (Borg/Kubernetes patterns), Netflix (chaos engineering, redundancy), Meta (distributed training at scale).

## Compute Patterns

### GPU Provisioning

**Avoid SageMaker for latency-sensitive GPU provisioning.** Cold start times are unacceptable for production workloads.

**Preferred approach: Pre-warmed Kubernetes GPU pools**

```yaml
# GKE Autopilot or EKS with Karpenter - pre-provisioned GPU nodes
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: gpu-training
spec:
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["p4d.24xlarge", "p5.48xlarge"]  # A100, H100
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]  # No spot for training
  limits:
    resources:
      nvidia.com/gpu: 64
  ttlSecondsAfterEmpty: 300  # Keep warm for 5min after idle
```

### Training Jobs (<3 days)

**Never use spot/preemptible for training under 3 days.** The checkpoint overhead and interruption risk exceeds any cost savings.

```
Training <3 days:    On-demand, dedicated GPU pools
Training >3 days:    On-demand with aggressive checkpointing (consider spot only if you have mature checkpoint/resume)
Inference:           On-demand, autoscaled
```

### Keep GPU Pools Warm

Pre-provision capacity. Cold GPU provisioning kills SLOs.

```terraform
# Reserve capacity for critical workloads
resource "aws_ec2_capacity_reservation" "gpu_training" {
  instance_type           = "p4d.24xlarge"
  instance_platform       = "Linux/UNIX"
  availability_zone       = "us-east-1a"
  instance_count          = 4
  instance_match_criteria = "targeted"
}
```

## High Availability Patterns

### Zero Downtime Deployments

Reference: Netflix's red/black deployment, Google's gradual rollouts.

```yaml
# Kubernetes rolling update - never drop requests
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 6  # Minimum for HA at scale
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # Never reduce capacity during deploy
      maxSurge: 2            # Spin up new before killing old
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: model-server
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]  # Drain connections
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
```

### Connection Draining & Graceful Shutdown

At 10-50k r/s, dropped connections are unacceptable.

```python
# Application-level graceful shutdown
import signal
import asyncio

class GracefulShutdown:
    def __init__(self):
        self.shutdown_event = asyncio.Event()
        self.active_requests = 0

    def start_request(self):
        if self.shutdown_event.is_set():
            raise ServiceUnavailable("Shutting down")
        self.active_requests += 1

    def end_request(self):
        self.active_requests -= 1

    async def shutdown(self, timeout=30):
        self.shutdown_event.set()
        # Wait for in-flight requests
        for _ in range(timeout * 10):
            if self.active_requests == 0:
                break
            await asyncio.sleep(0.1)
```

## High Throughput Data Patterns

### Storage for 10-50k r/s

**S3 is not a database.** For hot path data at this scale:

```
Hot path (real-time):    Redis Cluster / DynamoDB / Bigtable
Warm path (features):    Feature store (Feast + Redis, Vertex Feature Store)
Cold path (training):    S3/GCS with optimized formats
```

### Data Format Selection

At scale, format matters:

```
Parquet:       Columnar analytics, feature engineering
Delta/Iceberg: ACID transactions, time travel, concurrent writes
TFRecord:      TensorFlow training (but Parquet + petastorm often faster)
Lance:         Vector data, embeddings (newer, very fast)
```

### Streaming Architecture

For real-time at 50k r/s, batch is not an option.

```
Ingestion:     Kafka (self-managed) or Kinesis (managed)
Processing:    Flink (true streaming) > Spark Streaming (micro-batch)
Feature Store: Online store with <10ms p99 reads
```

```yaml
# Kafka cluster sizing for 50k r/s
# Rule of thumb: 3x headroom minimum
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
spec:
  kafka:
    replicas: 6  # Minimum for HA
    config:
      num.partitions: 64
      default.replication.factor: 3
      min.insync.replicas: 2
    storage:
      type: persistent-claim
      size: 1Ti
      class: io2  # High IOPS
```

## CI/CD Patterns

### Runner Selection

```
VPC access required:     CodeBuild runners (via GitHub Actions)
No VPC access needed:    ubuntu-latest GitHub-hosted runners
```

**CodeBuild for VPC workloads** - deploy, integration tests, anything touching private resources.

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: codebuild-${{ vars.CODEBUILD_PROJECT }}-${{ github.run_id }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name ${{ vars.CLUSTER_NAME }}
          kubectl apply -k overlays/production
```

**GitHub-hosted for non-VPC tasks** - linting, unit tests, building public artifacts.

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: |
          pip install ruff mypy
          ruff check .
          mypy src/
```

### CodeBuild Runner Setup

```terraform
resource "aws_codebuild_project" "github_runner" {
  name          = "github-actions-runner"
  service_role  = aws_iam_role.codebuild.arn

  environment {
    compute_type    = "BUILD_GENERAL1_LARGE"
    image           = "aws/codebuild/amazonlinux2-x86_64-standard:5.0"
    type            = "LINUX_CONTAINER"
    privileged_mode = true  # Required for Docker builds
  }

  vpc_config {
    vpc_id             = aws_vpc.main.id
    subnets            = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.codebuild.id]
  }

  source {
    type = "GITHUB"
    location = "https://github.com/org/repo.git"
  }

  artifacts {
    type = "NO_ARTIFACTS"
  }
}
```

## Container Patterns

### Base Images - Use Latest Stable

Always pin to specific versions, but stay current with security patches.

```dockerfile
# NVIDIA NGC containers - optimized and tested
FROM nvcr.io/nvidia/pytorch:24.01-py3

# Or AWS DLC for EKS/ECS
FROM 763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-inference:2.2.0-gpu-py310-cu121-ubuntu22.04-ec2
```

### Image Build Pipeline

Pre-build and cache. Never build during deployment.

```yaml
# Build on merge via CodeBuild
name: Build ML Images
on:
  push:
    branches: [main]
    paths: ['docker/**', 'requirements*.txt']

jobs:
  build:
    runs-on: codebuild-${{ vars.CODEBUILD_PROJECT }}-${{ github.run_id }}
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/amazon-ecr-login@v2
      - name: Build and push
        run: |
          docker build -t $ECR_REPO:${{ github.sha }} .
          docker push $ECR_REPO:${{ github.sha }}
```

## Observability (Non-Negotiable)

### Metrics at Scale

Reference: Google SRE's Four Golden Signals.

```python
# Prometheus metrics - must have for any service
from prometheus_client import Histogram, Counter, Gauge

REQUEST_LATENCY = Histogram(
    'inference_request_duration_seconds',
    'Inference request latency',
    ['model', 'version'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

REQUEST_COUNT = Counter(
    'inference_requests_total',
    'Total inference requests',
    ['model', 'version', 'status']
)

IN_FLIGHT = Gauge(
    'inference_requests_in_flight',
    'Currently processing requests',
    ['model']
)

MODEL_LOAD_TIME = Gauge(
    'model_load_seconds',
    'Time to load model',
    ['model', 'version']
)
```

### Alerting Thresholds

Set before production, not after incidents.

```yaml
# Prometheus alerting rules
groups:
  - name: inference-slos
    rules:
      - alert: HighLatencyP99
        expr: histogram_quantile(0.99, rate(inference_request_duration_seconds_bucket[5m])) > 0.5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "P99 latency exceeds 500ms"

      - alert: HighErrorRate
        expr: rate(inference_requests_total{status="error"}[5m]) / rate(inference_requests_total[5m]) > 0.001
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Error rate exceeds 0.1%"

      - alert: LowThroughput
        expr: rate(inference_requests_total[5m]) < 5000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Throughput dropped below 5k r/s"
```

## Disaster Recovery

### RTO/RPO Targets

Define these explicitly. "As fast as possible" is not a target.

```
Inference API:    RTO <30s (automatic failover), RPO N/A (stateless)
Feature Store:    RTO <5min, RPO <1min (sync replication)
Training Data:    RTO <1hr, RPO <24hr (daily backups acceptable)
Model Registry:   RTO <15min, RPO 0 (replicated storage)
```

### Chaos Engineering

Reference: Netflix Chaos Monkey. Test failures before they happen.

```yaml
# Litmus Chaos experiment - kill random inference pods
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: inference-pod-kill
spec:
  appinfo:
    appns: ml-inference
    applabel: app=model-server
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '30'
            - name: CHAOS_INTERVAL
              value: '10'
            - name: FORCE
              value: 'true'
```

## Anti-Patterns

| Don't | Do Instead |
|-------|------------|
| Spot for training <3 days | On-demand with reserved capacity |
| SageMaker for latency-critical GPU | Pre-warmed Kubernetes GPU pools |
| S3 for hot path reads | Redis/DynamoDB with <10ms p99 |
| Build images during deploy | Pre-build, cache, and promote |
| Alert on symptoms | Alert on SLOs (latency, error rate, throughput) |
| Manual failover | Automatic health-check based failover |
| Hope it works | Chaos test regularly |
| GitHub-hosted runners for VPC tasks | CodeBuild runners |
