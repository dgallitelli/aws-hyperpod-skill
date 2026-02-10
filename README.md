# Amazon SageMaker HyperPod Skill for Claude Code

A comprehensive Claude Code skill for provisioning and managing Amazon SageMaker HyperPod clusters for distributed ML training.

## Overview

Amazon SageMaker HyperPod is a managed service for provisioning resilient ML training clusters powered by AWS Trainium and NVIDIA GPUs (A100, H100). This skill provides expert guidance for:

- **Cluster creation** with EKS or Slurm orchestration
- **Job submission** for distributed training
- **Model compatibility verification** for Trainium/Inferentia
- **Troubleshooting** common issues
- **Best practices** for production deployments

## Installation

```bash
git clone https://github.com/dgallitelli/aws-hyperpod-skill.git ~/.claude/skills/sagemaker-hyperpod
```

Restart Claude Code to activate the skill.

## Requirements

### Common Requirements (Both Orchestrators)

| Requirement | Description |
|-------------|-------------|
| AWS CLI v2 | `aws --version` should show v2.x |
| AWS credentials | Configured via `aws configure` or environment variables |
| Service quotas | Approved for HyperPod instance types (e.g., `ml.trn1.32xlarge`) |
| VPC | With adequate IP space (see EFA requirements below) |
| S3 bucket | For lifecycle scripts and training artifacts |

### EKS Orchestrator Requirements

| Requirement | Installation | Purpose |
|-------------|--------------|---------|
| **HyperPod CLI** | `pip install sagemaker-hyperpod` | Primary tool for cluster/job management |
| kubectl | [Install guide](https://kubernetes.io/docs/tasks/tools/) | Kubernetes interaction |
| Helm v3 | [Install guide](https://helm.sh/docs/intro/install/) | Installing HyperPod dependencies |
| Python 3.8-3.11 | Required for HyperPod CLI | - |

### Slurm Orchestrator Requirements

| Requirement | Installation | Purpose |
|-------------|--------------|---------|
| Session Manager Plugin | [Install guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) | SSM access to nodes |
| Lifecycle scripts | Uploaded to S3 | Node configuration |

## Orchestrator Comparison

| Feature | EKS | Slurm |
|---------|-----|-------|
| **Primary CLI** | `hyp` (HyperPod CLI) | `aws sagemaker` |
| **Job submission** | PyTorchJob / `hyp create` | SBATCH scripts |
| **Best for** | Kubernetes teams, container workloads | HPC teams, traditional batch |
| **Learning curve** | Moderate (K8s concepts) | Lower (familiar HPC patterns) |
| **Ecosystem** | Kueue, Prometheus, Grafana | Slurm accounting, traditional HPC tools |

## Critical Requirements

### EFA Single-AZ Requirement

**CRITICAL**: For instances with EFA (Elastic Fabric Adapter) like `trn1.32xlarge`, all instances MUST be deployed in a **single Availability Zone**.

- Use `OverrideVpcConfig` with ONE subnet when creating clusters
- Multi-AZ deployments will cause EFA health check failures
- Nodes will cycle between Pending → ShuttingDown if misconfigured

### EKS Authentication Mode

HyperPod requires EKS clusters with `API` or `API_AND_CONFIG_MAP` authentication mode. The default `CONFIG_MAP` mode is **not supported**.

```bash
aws eks update-cluster-config \
  --name <cluster-name> \
  --access-config authenticationMode=API_AND_CONFIG_MAP
```

### HyperPod Helm Dependencies

Before creating a HyperPod cluster on EKS, install required dependencies:

```bash
git clone https://github.com/aws/sagemaker-hyperpod-cli.git
cd sagemaker-hyperpod-cli/helm_chart
helm dependencies update HyperPodHelmChart
helm install hyperpod-dependencies HyperPodHelmChart --namespace kube-system
```

## Quick Start

### EKS Orchestrator (using HyperPod CLI)

```bash
# 1. Install CLI
pip install sagemaker-hyperpod

# 2. Initialize and configure cluster
hyp init cluster-stack
hyp configure --resource-name-prefix my-hyperpod
hyp validate

# 3. Create cluster
hyp create --region us-east-1

# 4. Submit training job
hyp create hyp-pytorch-job \
  --job-name my-training \
  --image 763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training-neuronx:2.1.2 \
  --command '[python, train.py]' \
  --instance-type ml.trn1.32xlarge \
  --node-count 2

# 5. Monitor
hyp get jobs
hyp logs my-training
```

### Slurm Orchestrator (using AWS CLI)

```bash
# 1. Upload lifecycle scripts
aws s3 cp lifecycle-scripts/ s3://my-bucket/hyperpod/ --recursive

# 2. Create cluster
aws sagemaker create-cluster \
  --cluster-name my-slurm-cluster \
  --instance-groups file://cluster-config.json

# 3. Connect via SSM
aws ssm start-session --target <instance-id>

# 4. Submit job (on head node)
sbatch my-training-job.sh
```

## Usage

Trigger the skill by saying:
- "Create a HyperPod cluster"
- "Set up distributed training on AWS"
- "Help me configure Slurm for ML training"
- "Is Qwen3 supported on Trainium?"
- `/hyperpod` or `/sagemaker-hyperpod`

## Instance Types Supported

| Instance Type | Accelerator | GPUs/Chips | Memory | Use Case |
|---------------|-------------|------------|--------|----------|
| ml.p4d.24xlarge | NVIDIA A100 40GB | 8 | 320GB | General training |
| ml.p4de.24xlarge | NVIDIA A100 80GB | 8 | 640GB | Large models |
| ml.p5.48xlarge | NVIDIA H100 | 8 | 640GB | Latest gen training |
| ml.trn1.32xlarge | AWS Trainium | 16 | 512GB | Cost-effective training |
| ml.trn1n.32xlarge | AWS Trainium | 16 | 512GB | Enhanced networking |

## Model Compatibility (Trainium/Inferentia)

Before training on Trainium, verify model support via:
- [HuggingFace Optimum Neuron](https://huggingface.co/docs/optimum-neuron/en/supported_architectures)
- [AWS Neuron SDK Release Notes](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/)

**Currently supported for training**: Llama, Llama 2, Llama 3, Qwen3, Granite

## License

Apache-2.0

## Related Resources

- [SageMaker HyperPod Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-hyperpod.html)
- [HyperPod CLI GitHub](https://github.com/aws/sagemaker-hyperpod-cli)
- [AWS Neuron SDK](https://awsdocs-neuron.readthedocs-hosted.com/)
- [Awesome Distributed Training Examples](https://github.com/aws-samples/awsome-distributed-training)
