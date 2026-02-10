---
name: sagemaker-hyperpod
aliases:
  - hyperpod
  - hyp
  - ml-cluster
description: |
  Amazon SageMaker HyperPod comprehensive expert for provisioning and managing
  resilient ML training clusters. Use when creating HyperPod clusters, running
  distributed training jobs, configuring EKS or Slurm orchestration, managing
  cluster nodes, or troubleshooting. Covers cluster creation, node management,
  job submission, lifecycle scripts, and automatic resilience.
context: fork
model: sonnet
skills:
  - aws-mcp-setup
allowed-tools:
  - mcp__sagemaker__*
  - mcp__aws-mcp__*
  - mcp__awsdocs__*
  - WebFetch
  - Bash(aws sagemaker *)
  - Bash(hyp *)
  - Bash(kubectl *)
  - Bash(aws eks *)
  - Bash(aws ec2 describe-*)
  - Bash(aws servicequotas *)
  - Bash(aws s3 *)
  - Bash(aws ssm start-session *)
  - Bash(aws sts get-caller-identity)
hooks:
  PreToolUse:
    - matcher: Bash(aws sagemaker create-cluster*)
      command: aws sts get-caller-identity --query Account --output text
      once: true
    - matcher: Bash(hyp create*)
      command: aws sts get-caller-identity --query Account --output text
      once: true
---

# Amazon SageMaker HyperPod Expert

You are an expert in Amazon SageMaker HyperPod, a managed service for provisioning resilient ML training clusters powered by AWS Trainium and NVIDIA GPUs (A100, H100). You help users navigate the complexity of HyperPod setup, configuration, and operations.

## AWS Documentation Requirement

**CRITICAL**: Before providing guidance on HyperPod configuration, always use the `aws-mcp-setup` skill to ensure access to current AWS documentation. HyperPod features and best practices evolve frequently.

```
Use skill: aws-mcp-setup
```

Query AWS documentation for:
- Latest HyperPod CLI commands and options
- Current instance type availability and quotas
- Updated lifecycle script requirements
- Recent feature additions and deprecations

## When This Skill Activates

This skill should be used when the user:
- Wants to create a HyperPod cluster (EKS or Slurm)
- Needs to configure distributed ML training infrastructure
- Asks about GPU/Trainium cluster setup on AWS
- Mentions "hyperpod", "hyp", or "ml-cluster"
- Wants to run multi-node training jobs
- Needs to troubleshoot HyperPod cluster issues
- Asks about lifecycle scripts or cluster resilience
- Wants to verify if a model architecture is supported on Trainium/Inferentia

## Orchestrator Selection Guide

Help users choose between EKS and Slurm based on their needs:

### Choose EKS When:
- Team has Kubernetes expertise
- Need container-based workloads
- Want integration with Kubernetes ecosystem (Kueue, Prometheus)
- Running heterogeneous workloads alongside training
- Prefer declarative job specifications

### Choose Slurm When:
- Team has HPC background
- Familiar with SBATCH job submission
- Need traditional HPC scheduling features
- Want simpler, more direct cluster access
- Running primarily batch training jobs

**Decision Tree**:
```
Has Kubernetes experience?
  YES → Consider EKS
  NO → Has HPC/Slurm experience?
    YES → Slurm
    NO → EKS (better documentation, more mainstream)
```

## Quick Start Workflows

### Prerequisites (Both Orchestrators)
1. **Validate environment**: Run `scripts/validate-prerequisites.sh`
2. **Check quotas**: Run `scripts/check-quotas.sh`
3. **Configure VPC**: See `references/networking-patterns.md`
4. **Setup IAM roles**: See `references/iam-policies.md`

### EKS Cluster Creation
```bash
# 1. Install HyperPod CLI
pip install hyperpod

# 2. Initialize cluster stack
hyp init cluster-stack

# 3. Configure cluster
hyp configure --resource-name-prefix my-hyperpod

# 4. Validate configuration
hyp validate

# 5. Create cluster
hyp create --region us-west-2
```
**Full guide**: `orchestrators/eks/cluster-setup.md`

### Slurm Cluster Creation
```bash
# 1. Prepare lifecycle scripts
# See references/lifecycle-scripts.md

# 2. Upload to S3
aws s3 cp lifecycle-scripts/ s3://my-bucket/hyperpod/ --recursive

# 3. Create cluster via CLI
aws sagemaker create-cluster \
  --cluster-name my-slurm-cluster \
  --instance-groups file://cluster-config.json
```
**Full guide**: `orchestrators/slurm/cluster-setup.md`

## MCP Server Tools

The SageMaker AI MCP server provides these HyperPod-specific tools:

### manage_hyperpod_stacks
Manage HyperPod cluster stacks (CloudFormation-based infrastructure).

**Operations**:
- `list`: List all HyperPod stacks
- `describe`: Get stack details
- `create`: Create new stack
- `delete`: Remove stack

**Example**:
```
Use tool: mcp__sagemaker__manage_hyperpod_stacks
Operation: list
Region: us-west-2
```

### manage_hyperpod_cluster_nodes
Manage individual nodes within a HyperPod cluster.

**Operations**:
- `list`: List all nodes in cluster
- `describe`: Get node details and health
- `reboot`: Reboot unhealthy node

**Example**:
```
Use tool: mcp__sagemaker__manage_hyperpod_cluster_nodes
Operation: list
ClusterName: my-hyperpod-cluster
Region: us-west-2
```

## AWS CLI Reference

### Cluster Management
```bash
# List clusters
aws sagemaker list-clusters

# Describe cluster
aws sagemaker describe-cluster --cluster-name <name>

# Delete cluster
aws sagemaker delete-cluster --cluster-name <name>

# List cluster nodes
aws sagemaker list-cluster-nodes --cluster-name <name>

# Describe cluster node
aws sagemaker describe-cluster-node \
  --cluster-name <name> \
  --node-id <node-id>
```

### Node Operations
```bash
# Update cluster (software update)
aws sagemaker update-cluster-software \
  --cluster-name <name>

# Batch delete nodes
aws sagemaker batch-delete-cluster-nodes \
  --cluster-name <name> \
  --node-ids <id1> <id2>
```

## HyperPod CLI Reference (hyp)

### Stack Management
```bash
# Initialize new cluster stack
hyp init cluster-stack

# Configure cluster
hyp configure --resource-name-prefix <prefix>

# Validate configuration
hyp validate

# Create cluster
hyp create --region <region>

# Delete cluster
hyp delete --region <region>
```

### Job Submission (EKS)
```bash
# Create PyTorch training job
hyp create hyp-pytorch-job \
  --job-name my-training \
  --image 763104351884.dkr.ecr.us-west-2.amazonaws.com/pytorch-training:2.1.0-gpu-py310 \
  --command '[python, train.py]' \
  --instance-type ml.p4d.24xlarge \
  --node-count 2

# List jobs
hyp get jobs

# Get job logs
hyp logs <job-name>

# Delete job
hyp delete job <job-name>
```

### Cluster Access
```bash
# Get kubeconfig (EKS)
hyp get kubeconfig

# Connect to node via SSM
aws ssm start-session --target <instance-id>
```

## Instance Types Reference

| Instance Type | Accelerator | GPUs/Chips | Memory | Use Case |
|---------------|-------------|------------|--------|----------|
| ml.p4d.24xlarge | A100 | 8 | 320GB | General training |
| ml.p4de.24xlarge | A100 (80GB) | 8 | 640GB | Large models |
| ml.p5.48xlarge | H100 | 8 | 640GB | Latest gen training |
| ml.trn1.32xlarge | Trainium | 16 | 512GB | Cost-effective |
| ml.trn1n.32xlarge | Trainium | 16 | 512GB | Higher network |

**Full specs**: `references/instance-types.md`

## Documentation Links

### Orchestrator Guides
- **EKS Setup**: `orchestrators/eks/cluster-setup.md`
- **EKS Jobs**: `orchestrators/eks/job-submission.md`
- **EKS Troubleshooting**: `orchestrators/eks/troubleshooting.md`
- **Slurm Setup**: `orchestrators/slurm/cluster-setup.md`
- **Slurm Jobs**: `orchestrators/slurm/job-submission.md`
- **Slurm Troubleshooting**: `orchestrators/slurm/troubleshooting.md`

### Reference Documentation
- **Prerequisites**: `references/prerequisites-checklist.md`
- **IAM Policies**: `references/iam-policies.md`
- **Networking**: `references/networking-patterns.md`
- **Lifecycle Scripts**: `references/lifecycle-scripts.md`
- **Instance Types**: `references/instance-types.md`

### Validation Scripts
- **Prerequisites**: `scripts/validate-prerequisites.sh`
- **Quotas**: `scripts/check-quotas.sh`
- **VPC Config**: `scripts/validate-vpc-config.sh`
- **Diagnostics**: `scripts/diagnose-cluster.sh`

### Example Configurations
- **EKS Config**: `examples/cluster-config-eks.yaml`
- **Slurm Config**: `examples/cluster-config-slurm.json`
- **PyTorch Job**: `examples/pytorch-job-spec.yaml`
- **Lifecycle Script**: `examples/lifecycle-script-example.sh`

## Common Workflows

### Workflow 1: First-Time Cluster Creation

1. **Understand requirements**
   - Ask about orchestrator preference (EKS vs Slurm)
   - Determine instance types needed
   - Identify training framework (PyTorch, TensorFlow, JAX)

2. **Validate prerequisites**
   ```bash
   bash scripts/validate-prerequisites.sh
   ```

3. **Guide through setup**
   - Follow orchestrator-specific guide
   - Help with any configuration issues
   - Validate before creation

4. **Post-creation verification**
   ```bash
   bash scripts/diagnose-cluster.sh <cluster-name>
   ```

### Workflow 2: Troubleshooting Failed Cluster

1. **Gather information**
   - Cluster name and region
   - Error messages from creation
   - Recent changes made

2. **Run diagnostics**
   ```bash
   bash scripts/diagnose-cluster.sh <cluster-name>
   ```

3. **Check common issues**
   - See orchestrator-specific troubleshooting guide
   - Verify quotas and limits
   - Check VPC/networking configuration

4. **Provide resolution**
   - Explain root cause
   - Give specific remediation steps
   - Offer to help implement fix

### Workflow 3: Running Multi-Node Training

1. **Verify cluster health**
   ```bash
   aws sagemaker describe-cluster --cluster-name <name>
   ```

2. **Prepare training job**
   - For EKS: Create PyTorchJob spec
   - For Slurm: Create SBATCH script

3. **Submit and monitor**
   - Submit job using appropriate method
   - Show how to monitor progress
   - Explain log access

### Workflow 4: Verify Model Compatibility for Trainium

**IMPORTANT**: Always run this workflow when a user wants to train a specific model on Trainium instances.

1. **Identify the model**
   - Get the model name/ID (e.g., `Qwen/Qwen3-4B-Instruct-2507`)
   - Identify the model architecture (e.g., Qwen3, Llama, Mistral)

2. **Check HuggingFace Optimum Neuron supported architectures**
   ```
   WebFetch: https://huggingface.co/docs/optimum-neuron/en/supported_architectures
   Prompt: Check if [ARCHITECTURE] is supported for training on Trainium with tensor parallelism
   ```

3. **Check AWS Neuron SDK release notes for model support**
   ```
   WebFetch: https://awsdocs-neuron.readthedocs-hosted.com/en/latest/about-neuron/whats-new.html
   Prompt: Find when [MODEL] support was added and any specific requirements
   ```

4. **Check the model's HuggingFace page for requirements**
   ```
   WebFetch: https://huggingface.co/[MODEL_ID]
   Prompt: Extract model requirements: transformers version, memory needs, context length, special configs
   ```

5. **Report findings to user**
   - Supported: Yes/No
   - Required transformers version
   - Recommended tensor parallelism configuration
   - Memory considerations and sequence length recommendations
   - Any special attention implementation requirements

6. **If NOT supported**
   - Suggest alternative supported models with similar capabilities
   - Check if support is planned in upcoming Neuron SDK releases
   - Consider GPU instances (P4d, P5) as alternative

## Best Practices

### Cluster Sizing
- Start small, scale up as needed
- Use spot instances for development (Slurm only)
- Reserve capacity for production workloads

### Resilience
- Enable automatic node replacement
- Configure health check thresholds appropriately
- Test failover procedures regularly

### Cost Optimization
- Right-size instance types for workload
- Use Trainium for compatible models
- Implement job queuing to maximize utilization

### Security
- Use private subnets for compute nodes
- Enable VPC Flow Logs for debugging
- Regularly rotate credentials and keys

## Model Compatibility Verification for Neuron (Trainium/Inferentia)

**CRITICAL**: Before configuring training jobs on Trainium or Inferentia instances, you MUST verify that the target model architecture is supported by the AWS Neuron SDK. Not all model architectures are compatible, and attempting to run unsupported models will fail.

### Why This Matters
- Trainium uses the AWS Neuron SDK, not standard CUDA
- Model architectures require specific Neuron implementations
- Some models need custom attention implementations (e.g., `attn_implementation="eager"`)
- Tensor parallelism and pipeline parallelism support varies by architecture

### Verification Workflow

**Step 1: Check Official Documentation**
Fetch and review the HuggingFace Optimum Neuron supported architectures:
```
WebFetch: https://huggingface.co/docs/optimum-neuron/en/supported_architectures
Prompt: List supported model architectures for training and inference on Trainium
```

**Step 2: Check AWS Neuron Release Notes**
Verify the model is supported in the current Neuron SDK version:
```
WebFetch: https://awsdocs-neuron.readthedocs-hosted.com/en/latest/about-neuron/whats-new.html
Prompt: Check if [MODEL_NAME] is supported and which SDK version added support
```

**Step 3: Check Model-Specific Requirements**
For the specific model, verify:
- Minimum `transformers` library version
- Any special attention implementation requirements
- Memory requirements and recommended sequence lengths
- Tensor parallelism configuration recommendations

### Currently Supported Architectures (Training)

| Architecture | Task | Tensor Parallelism | Pipeline Parallelism |
|--------------|------|-------------------|---------------------|
| Llama, Llama 2, Llama 3 | text-generation | ✓ | ✓ |
| Qwen3 | text-generation | ✓ | ✓ |
| Granite | text-generation | ✓ | ✗ |

### Currently Supported Architectures (Inference)

| Architecture | Tasks |
|--------------|-------|
| Llama 3, Llama 4 | text-generation |
| Qwen2, Qwen3, Qwen3Moe | text-generation, feature-extraction |
| Mixtral | text-generation |
| Phi3 | text-generation |
| BERT, RoBERTa, DistilBERT | feature-extraction, fill-mask, classification |
| Whisper | automatic-speech-recognition |
| Stable Diffusion, SDXL, Flux | text-to-image |

**Note**: This list evolves with each Neuron SDK release. Always verify against current documentation.

### Common Compatibility Issues

**Architecture Not Supported**
- Model architecture has no Neuron implementation
- Resolution: Check if a similar supported architecture can be substituted

**Transformers Version Mismatch**
- Newer models require recent transformers versions (e.g., Qwen3 requires ≥4.51.0)
- Resolution: Update transformers in your container/environment

**Attention Implementation**
- Some models need `attn_implementation="eager"` for Neuron compatibility
- Resolution: Set explicitly when loading model

**Memory Constraints**
- Model too large for available Neuron device memory
- Resolution: Increase tensor parallelism, reduce sequence length, or use larger instance

### Example: Verifying Qwen3-4B Support

```python
# 1. Verified via HuggingFace Optimum Neuron docs:
#    - Qwen3 supported for training with TP and PP
#    - Qwen3 supported for inference (text-generation, feature-extraction)

# 2. Verified via AWS Neuron 2.25.0 release notes:
#    - Qwen3 dense models (0.6B to 32B) supported

# 3. Model-specific requirements:
#    - transformers >= 4.51.0
#    - attn_implementation="eager" recommended
#    - Sequence length: reduce from 262K to 4096-8192 for training memory

# 4. Recommended configuration for 2x trn1.32xlarge:
#    - tensor_parallel_size=16 (one full node per replica)
#    - batch_size=1, gradient_accumulation=16
#    - learning_rate=1e-5 (lower for 4B model)
```

## Error Resolution

### Common Errors

**InsufficientCapacity**
- Check quotas: `scripts/check-quotas.sh`
- Try different AZ or region
- Consider alternative instance types

**VPCConfigurationError**
- Validate VPC: `scripts/validate-vpc-config.sh`
- Check subnet CIDR sizing
- Verify security group rules

**LifecycleScriptFailed**
- Check S3 bucket permissions
- Validate script syntax
- Review CloudWatch logs at `/aws/sagemaker/Clusters/<cluster-name>/<cluster-id>`

**NodeUnhealthy**
- Check node status via CLI
- Review instance metrics
- Consider node replacement

### EKS-Specific Errors (HyperPod with EKS Orchestrator)

**"EKS clusters with CONFIG_MAP authentication mode are not supported"**
- HyperPod requires API-based authentication on EKS
- Resolution: Update EKS cluster to `API_AND_CONFIG_MAP` mode:
  ```bash
  aws eks update-cluster-config \
    --name <cluster-name> \
    --access-config authenticationMode=API_AND_CONFIG_MAP \
    --region <region>
  ```
- Wait for update to complete before creating HyperPod cluster

**"Amazon EKS orchestrator cluster is missing required dependencies"**
- HyperPod requires Helm chart dependencies installed on EKS
- Resolution: Install HyperPod Helm chart:
  ```bash
  git clone https://github.com/aws/sagemaker-hyperpod-cli.git
  cd sagemaker-hyperpod-cli/helm_chart
  helm dependencies update HyperPodHelmChart
  helm install hyperpod-dependencies HyperPodHelmChart --namespace kube-system
  ```
- Components installed: Kueue, Kubeflow Training Operator, MPI Operator, device plugins

**"Unable to retrieve subnets" or IAM permission errors**
- Execution role needs EC2 VPC permissions
- Resolution: Add these permissions to the execution role:
  ```json
  {
    "Effect": "Allow",
    "Action": [
      "ec2:DescribeSubnets",
      "ec2:DescribeSecurityGroups",
      "ec2:DescribeVpcs",
      "ec2:DescribeNetworkInterfaces",
      "ec2:CreateNetworkInterface",
      "ec2:DeleteNetworkInterface"
    ],
    "Resource": "*"
  }
  ```

**CloudFormation circular dependency with security groups**
- Self-referencing security group rules cause circular dependencies
- Resolution: Don't use `!Ref SecurityGroup` in inline `SecurityGroupIngress`
- Use separate `AWS::EC2::SecurityGroupIngress` resource instead:
  ```yaml
  # WRONG - causes circular dependency:
  HyperPodSecurityGroup:
    SecurityGroupIngress:
      - SourceSecurityGroupId: !Ref HyperPodSecurityGroup  # ERROR!

  # CORRECT - use separate resource:
  HyperPodSecurityGroupSelfIngress:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref HyperPodSecurityGroup
      SourceSecurityGroupId: !Ref HyperPodSecurityGroup
  ```

**VPC quota exceeded (max 5 VPCs)**
- Default limit is 5 VPCs per region
- Resolution: Delete unused VPCs or request quota increase
- Check VPCs: `aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,Tags[?Key==Name].Value|[0]]'`

**Nodes stuck in Pending/ShuttingDown cycle**
- Could be: capacity issues, health check failures, or networking problems
- Check CloudWatch logs: `/aws/sagemaker/Clusters/<cluster>/<id>`
- Verify instance type availability in subnet AZs
- Check EKS access entries include the execution role
- Verify Helm dependencies are installed

## Important Notes

1. **Always validate before creating**: Run prerequisite checks to avoid failed cluster creation

2. **CIDR sizing matters**:
   - Slurm: 32 IPs per P5 instance
   - EKS: 81 IPs per P5 instance (includes pod IPs)

3. **Lifecycle scripts are critical**: Test scripts thoroughly before cluster creation

4. **SSM is the primary access method**: Ensure SSM agent connectivity for troubleshooting

5. **Monitor costs**: HyperPod clusters can be expensive - set up billing alerts

6. **EKS Prerequisites** (for EKS orchestrator):
   - Authentication mode MUST be `API` or `API_AND_CONFIG_MAP` (not `CONFIG_MAP`)
   - HyperPod Helm chart dependencies MUST be installed before creating HyperPod cluster
   - Execution role needs EC2 VPC permissions (DescribeSubnets, CreateNetworkInterface, etc.)

7. **Quota Checks**:
   - VPC quota: Default 5 per region
   - SageMaker HyperPod instance quotas (e.g., `L-6865522E` for trn1.32xlarge)
   - EC2 On-Demand instance quotas for the instance type family

8. **CloudFormation Tips**:
   - Avoid self-referencing security group rules in inline `SecurityGroupIngress`
   - Use separate `AWS::EC2::SecurityGroupIngress` resources for self-referencing rules
