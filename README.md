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

### From Claude Code Marketplace

```bash
claude skill install aws-hyperpod
```

### Manual Installation

1. Clone this repository
2. Add to your Claude Code skills directory
3. Restart Claude Code

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

## Quick Start

### Trigger the Skill

Say any of:
- "Create a HyperPod cluster"
- "Set up distributed training on AWS"
- "Help me configure Slurm for ML training"
- "/hyperpod"

### Example Workflow

1. **Validate prerequisites**
   ```bash
   bash scripts/validate-prerequisites.sh --region us-west-2
   ```

2. **Create cluster** (EKS)
   ```bash
   hyp init cluster-stack
   hyp configure --resource-name-prefix my-cluster
   hyp validate
   hyp create --region us-west-2
   ```

3. **Submit training job**
   ```bash
   hyp create hyp-pytorch-job \
     --job-name my-training \
     --image <training-image> \
     --instance-type ml.p5.48xlarge \
     --node-count 4
   ```

## Documentation Structure

```
plugins/aws-hyperpod/skills/sagemaker-hyperpod/
├── SKILL.md                          # Main skill definition
├── orchestrators/
│   ├── eks/                          # EKS-specific guides
│   └── slurm/                        # Slurm-specific guides
├── references/
│   ├── prerequisites-checklist.md
│   ├── iam-policies.md
│   ├── networking-patterns.md
│   ├── lifecycle-scripts.md
│   └── instance-types.md
├── scripts/                          # Validation scripts
└── examples/                         # Configuration examples
```

## MCP Server Integration

This skill includes the SageMaker AI MCP server for enhanced functionality:

```json
{
  "mcpServers": {
    "sagemaker": {
      "type": "stdio",
      "command": "uvx",
      "args": ["awslabs.sagemaker-ai-mcp-server@latest", "--allow-write"]
    }
  }
}
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

## Contributing

Contributions welcome! Please read the contribution guidelines before submitting PRs.

## License

Apache-2.0

## Related Resources

- [SageMaker HyperPod Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-hyperpod.html)
- [HyperPod CLI Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-hyperpod-cli.html)
- [AWS ML Blog](https://aws.amazon.com/blogs/machine-learning/)
