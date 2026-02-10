# Amazon SageMaker HyperPod Skill for Claude Code

A comprehensive Claude Code skill for provisioning and managing Amazon SageMaker HyperPod clusters for distributed ML training.

## Overview

Amazon SageMaker HyperPod is a managed service for provisioning resilient ML training clusters powered by AWS Trainium and NVIDIA GPUs (A100, H100). This skill provides expert guidance for:

- **Cluster creation** with EKS or Slurm orchestration
- **Prerequisites validation** before deployment
- **Job submission** for distributed training
- **Troubleshooting** common issues
- **Best practices** for production deployments

## Installation

```bash
git clone https://github.com/dgallitelli/aws-hyperpod-skill.git ~/.claude/skills/sagemaker-hyperpod
```

Restart Claude Code to activate the skill.

## Usage

Trigger the skill by saying:
- "Create a HyperPod cluster"
- "Set up distributed training on AWS"
- "Help me configure Slurm for ML training"
- `/sagemaker-hyperpod`

## Features

### Orchestrator Support

| Orchestrator | Description |
|--------------|-------------|
| **EKS** | Kubernetes-native orchestration with PyTorchJob support |
| **Slurm** | Traditional HPC workload management with SBATCH |

### Validation Scripts

- `validate-prerequisites.sh` - Pre-creation checks
- `check-quotas.sh` - Service quota validation
- `validate-vpc-config.sh` - Network configuration validation
- `diagnose-cluster.sh` - Cluster troubleshooting

### Instance Types Supported

- ml.p4d.24xlarge (NVIDIA A100 40GB)
- ml.p4de.24xlarge (NVIDIA A100 80GB)
- ml.p5.48xlarge (NVIDIA H100)
- ml.trn1.32xlarge (AWS Trainium)
- ml.trn1n.32xlarge (AWS Trainium with enhanced networking)

## Structure

```
sagemaker-hyperpod/
├── SKILL.md              # Skill definition
├── orchestrators/
│   ├── eks/              # EKS guides
│   └── slurm/            # Slurm guides
├── references/           # IAM, networking, prerequisites
├── scripts/              # Validation scripts
└── examples/             # Config examples
```

## Requirements

- AWS CLI v2
- AWS credentials configured
- Service quotas approved for HyperPod instances
- VPC with adequate IP space
- S3 bucket for lifecycle scripts

### For EKS Orchestration
- HyperPod CLI (`pip install hyperpod`)
- kubectl
- Helm v3

### For Slurm Orchestration
- Session Manager Plugin (for SSM access)

## License

Apache-2.0

## Related Resources

- [SageMaker HyperPod Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-hyperpod.html)
- [HyperPod CLI Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-hyperpod-cli.html)
