# EKS Job Submission Guide

This guide covers submitting distributed training jobs on HyperPod with EKS orchestration.

## Overview

EKS orchestration supports multiple job submission methods:
- **HyperPod CLI**: Simplified job submission with `hyp`
- **PyTorchJob**: Kubernetes-native distributed training
- **Raw Kubernetes**: Pod manifests for custom workloads

## Method 1: HyperPod CLI (Recommended)

### Basic Job Submission

```bash
hyp create hyp-pytorch-job \
  --job-name my-training-job \
  --image 763104351884.dkr.ecr.us-west-2.amazonaws.com/pytorch-training:2.1.0-gpu-py310-cu118-ubuntu20.04-sagemaker \
  --command '[python, train.py]' \
  --instance-type ml.p5.48xlarge \
  --node-count 4
```

### With Additional Options

```bash
hyp create hyp-pytorch-job \
  --job-name llm-training \
  --image your-ecr-repo/training:latest \
  --command '[torchrun, --nproc_per_node=8, train.py]' \
  --instance-type ml.p5.48xlarge \
  --node-count 4 \
  --env "NCCL_DEBUG=INFO,HF_HOME=/data/cache" \
  --volume-mounts "/fsx:/data" \
  --namespace training
```

### Monitor Job

```bash
# List jobs
hyp get jobs

# Get job details
hyp describe job my-training-job

# View logs
hyp logs my-training-job

# Stream logs
hyp logs my-training-job --follow

# Delete job
hyp delete job my-training-job
```

## Method 2: PyTorchJob (Kubernetes Native)

### Basic PyTorchJob

```yaml
# pytorch-job.yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: multi-node-training
  namespace: default
spec:
  pytorchReplicaSpecs:
    Worker:
      replicas: 4
      restartPolicy: OnFailure
      template:
        metadata:
          labels:
            app: pytorch-training
        spec:
          containers:
          - name: pytorch
            image: 763104351884.dkr.ecr.us-west-2.amazonaws.com/pytorch-training:2.1.0-gpu-py310-cu118-ubuntu20.04-sagemaker
            imagePullPolicy: Always
            command:
              - torchrun
              - --nproc_per_node=8
              - --nnodes=4
              - --rdzv_backend=c10d
              - --rdzv_endpoint=$(MASTER_ADDR):29500
              - train.py
            resources:
              limits:
                nvidia.com/gpu: 8
                vpc.amazonaws.com/efa: 4
                memory: 1000Gi
                cpu: "96"
              requests:
                nvidia.com/gpu: 8
                vpc.amazonaws.com/efa: 4
                memory: 500Gi
                cpu: "48"
            env:
              - name: NCCL_DEBUG
                value: "INFO"
              - name: NCCL_SOCKET_IFNAME
                value: "eth0"
              - name: FI_EFA_USE_DEVICE_RDMA
                value: "1"
              - name: FI_PROVIDER
                value: "efa"
            volumeMounts:
              - name: data
                mountPath: /data
              - name: dshm
                mountPath: /dev/shm
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: fsx-pvc
            - name: dshm
              emptyDir:
                medium: Memory
                sizeLimit: 100Gi
          nodeSelector:
            node.kubernetes.io/instance-type: ml.p5.48xlarge
          tolerations:
            - key: "nvidia.com/gpu"
              operator: "Exists"
              effect: "NoSchedule"
```

### Submit and Monitor

```bash
# Submit job
kubectl apply -f pytorch-job.yaml

# Check status
kubectl get pytorchjobs

# Describe job
kubectl describe pytorchjob multi-node-training

# Get worker pods
kubectl get pods -l pytorch-job-name=multi-node-training

# View logs
kubectl logs -f multi-node-training-worker-0

# Delete job
kubectl delete pytorchjob multi-node-training
```

## Method 3: Custom Pod Manifest

For non-PyTorch workloads or custom scheduling:

```yaml
# training-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-training
  labels:
    app: training
spec:
  restartPolicy: Never
  containers:
  - name: training
    image: your-image:latest
    command: ["python", "train.py"]
    resources:
      limits:
        nvidia.com/gpu: 8
    env:
      - name: CUDA_VISIBLE_DEVICES
        value: "0,1,2,3,4,5,6,7"
    volumeMounts:
      - name: dshm
        mountPath: /dev/shm
  volumes:
    - name: dshm
      emptyDir:
        medium: Memory
  nodeSelector:
    node.kubernetes.io/instance-type: ml.p5.48xlarge
```

## Training Script Integration

### Environment Variables

HyperPod sets these environment variables automatically:

| Variable | Description |
|----------|-------------|
| `MASTER_ADDR` | IP of rank 0 worker |
| `MASTER_PORT` | Port for distributed communication |
| `WORLD_SIZE` | Total number of workers |
| `RANK` | Global rank of this worker |
| `LOCAL_RANK` | Local rank on this node |

### PyTorch Training Script Example

```python
# train.py
import os
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

def setup_distributed():
    """Initialize distributed training."""
    # HyperPod sets these automatically
    rank = int(os.environ.get("RANK", 0))
    world_size = int(os.environ.get("WORLD_SIZE", 1))
    local_rank = int(os.environ.get("LOCAL_RANK", 0))

    # Initialize process group
    dist.init_process_group(
        backend="nccl",
        init_method="env://",
        world_size=world_size,
        rank=rank
    )

    # Set device
    torch.cuda.set_device(local_rank)

    return rank, world_size, local_rank

def main():
    rank, world_size, local_rank = setup_distributed()

    # Create model
    model = YourModel().cuda(local_rank)
    model = DDP(model, device_ids=[local_rank])

    # Training loop
    for epoch in range(num_epochs):
        train_one_epoch(model, dataloader)

        # Save checkpoint (only rank 0)
        if rank == 0:
            torch.save(model.state_dict(), f"checkpoint_{epoch}.pt")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
```

## Using Shared Storage

### FSx for Lustre PVC

```yaml
# fsx-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fsx-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: fsx-lustre
  resources:
    requests:
      storage: 1200Gi
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fsx-lustre
provisioner: fsx.csi.aws.com
parameters:
  subnetId: subnet-xxxxx
  securityGroupIds: sg-xxxxx
  deploymentType: PERSISTENT_2
  perUnitStorageThroughput: "250"
```

### Mount in Job

```yaml
spec:
  containers:
  - name: training
    volumeMounts:
      - name: data
        mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: fsx-pvc
```

## Job Queuing with Kueue

### Create Workload

```yaml
# queued-job.yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: queued-training
  labels:
    kueue.x-k8s.io/queue-name: training-queue
spec:
  suspend: true  # Required for Kueue
  pytorchReplicaSpecs:
    Worker:
      replicas: 4
      template:
        spec:
          containers:
          - name: pytorch
            image: your-image:latest
            resources:
              limits:
                nvidia.com/gpu: 8
```

### Monitor Queue

```bash
# Check queue status
kubectl get clusterqueues
kubectl get localqueues

# Check workload status
kubectl get workloads

# Check pending jobs
kubectl get pytorchjobs -l kueue.x-k8s.io/queue-name=training-queue
```

## Job Monitoring

### View GPU Utilization

```bash
# Run on worker pod
kubectl exec -it multi-node-training-worker-0 -- nvidia-smi

# Continuous monitoring
kubectl exec -it multi-node-training-worker-0 -- watch -n1 nvidia-smi
```

### Check NCCL Communication

```bash
# View NCCL debug output
kubectl logs multi-node-training-worker-0 | grep NCCL
```

### Prometheus Metrics (if installed)

```yaml
# ServiceMonitor for training metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: training-monitor
spec:
  selector:
    matchLabels:
      app: pytorch-training
  endpoints:
  - port: metrics
    interval: 30s
```

## Common Patterns

### Checkpoint to S3

```python
import boto3

def save_checkpoint_s3(model, epoch, bucket, prefix):
    """Save checkpoint to S3."""
    s3 = boto3.client("s3")
    checkpoint_path = f"/tmp/checkpoint_{epoch}.pt"
    torch.save(model.state_dict(), checkpoint_path)
    s3.upload_file(
        checkpoint_path,
        bucket,
        f"{prefix}/checkpoint_{epoch}.pt"
    )
```

### Resume from Checkpoint

```yaml
spec:
  containers:
  - name: pytorch
    command:
      - python
      - train.py
      - --resume=/data/checkpoints/latest.pt
```

### Multi-Stage Training

```yaml
# Stage 1: Pretraining
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pretrain
spec:
  pytorchReplicaSpecs:
    Worker:
      replicas: 8
      # ... pretrain config
---
# Stage 2: Fine-tuning (depends on pretrain)
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: finetune
  annotations:
    depends-on: pretrain
spec:
  pytorchReplicaSpecs:
    Worker:
      replicas: 2
      # ... finetune config
```

## Troubleshooting Jobs

### Pod Not Scheduling

```bash
# Check pod events
kubectl describe pod multi-node-training-worker-0

# Check node resources
kubectl describe nodes | grep -A20 "Allocated resources"
```

### NCCL Timeout

```bash
# Increase timeout
env:
  - name: NCCL_TIMEOUT
    value: "1800"
```

### OOM Errors

```bash
# Increase shared memory
volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 200Gi
```

See [troubleshooting.md](troubleshooting.md) for more solutions.
