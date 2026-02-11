---
name: sagemaker-hyperpod
description: |
  Amazon SageMaker HyperPod expert for ML training clusters with Trainium or GPU.
  Use when: creating HyperPod clusters, running distributed training, configuring
  EKS or Slurm orchestration, troubleshooting cluster issues, checking quotas,
  or when user mentions "hyperpod", "hyp", "ml-cluster", "trainium", "trn1",
  "distributed training", or "multi-node training".
argument-hint: "[cluster-name or action]"
context: fork
model: sonnet
skills:
  - aws-mcp-setup
allowed-tools:
  - mcp__sagemaker__*
  - mcp__aws-mcp__*
  - mcp__awsdocs__*
  - WebFetch
  - Bash(hyp *)
  - Bash(aws sagemaker *)
  - Bash(kubectl *)
  - Bash(aws eks *)
  - Bash(aws ec2 describe-*)
  - Bash(aws servicequotas *)
  - Bash(aws s3 *)
  - Bash(aws ssm start-session *)
  - Bash(aws sts get-caller-identity)
  - Bash(aws logs *)
  - Bash(aws iam get-role*)
  - Bash(aws iam list-*)
  - Bash(helm *)
  - Bash(pip install sagemaker-hyperpod)
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

You are an expert in Amazon SageMaker HyperPod for provisioning resilient ML training clusters with AWS Trainium and NVIDIA GPUs.

## When This Skill Activates

- Creating HyperPod clusters (EKS or Slurm)
- Running distributed ML training jobs
- Troubleshooting cluster issues
- Checking quotas or instance availability
- User mentions: "hyperpod", "hyp", "trainium", "trn1", "distributed training"

## Detailed Guides

| Guide | Use When |
|-------|----------|
| [reference/eks-guide.md](reference/eks-guide.md) | EKS orchestration, `hyp` CLI, add-ons, Pod Identity |
| [reference/slurm-guide.md](reference/slurm-guide.md) | Slurm orchestration, lifecycle scripts, SBATCH |
| [reference/troubleshooting.md](reference/troubleshooting.md) | Error diagnosis and solutions |

---

## Orchestrator Selection

| Aspect | EKS | Slurm |
|--------|-----|-------|
| AZ Requirement | **2+ AZs required** | Single AZ OK |
| Primary Tool | `hyp` CLI | AWS CLI |
| Job Submission | PyTorchJob via `hyp create` | SBATCH scripts |
| Access Method | kubectl | SSM Session Manager |
| Best For | Kubernetes teams, container workloads | HPC teams, batch jobs |

---

## Instance Types

| Instance Type | Accelerator | Count | Use Case |
|---------------|-------------|-------|----------|
| ml.p4d.24xlarge | A100 | 8 | General training |
| ml.p4de.24xlarge | A100 (80GB) | 8 | Large models |
| ml.p5.48xlarge | H100 | 8 | Latest gen training |
| ml.trn1.32xlarge | Trainium | 16 | Cost-effective |
| ml.trn1n.32xlarge | Trainium | 16 | Higher network |

**IMPORTANT**: `ml.trn1.2xlarge` is NOT supported for HyperPod - only `ml.trn1.32xlarge`.

---

## CRITICAL: Pre-Creation Validation

**ALWAYS perform these checks BEFORE creating a cluster:**

### 1. Verify Instance Type Support

```bash
# Must say "for cluster usage" in quota name
aws service-quotas list-service-quotas \
  --service-code sagemaker --region us-east-1 \
  --query 'Quotas[?contains(QuotaName, `<INSTANCE_TYPE>`) && contains(QuotaName, `cluster`)].[QuotaName,Value]' \
  --output table
```

### 2. Check AZ Availability

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters Name=instance-type,Values=trn1.32xlarge \
  --region us-east-1 \
  --query 'InstanceTypeOfferings[*].Location' --output text
```

### 3. For EKS: Ensure 2+ AZs in config.yaml

```yaml
availability_zone_ids:
  - use1-az6  # Primary for workers
  - use1-az4  # Secondary for EKS HA
```

### 4. Check K8s Version (EKS Only)

```
WebFetch: https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-release-calendar
Prompt: What is the latest Kubernetes version in standard support?
```

### 5. Check Add-on Compatibility (EKS Only)

Before upgrading K8s versions, verify HyperPod add-ons support the target version:
```bash
aws eks describe-addon-versions --addon-name amazon-sagemaker-hyperpod-taskgovernance \
  --query 'addons[0].addonVersions[*].compatibilities[*].clusterVersion' --output text
```

**WARNING**: EKS does NOT support downgrading. Stay on a supported version if you need HyperPod add-ons.

---

## EKS Quick Start

```bash
# 1. Install CLI
pip install sagemaker-hyperpod

# 2. Initialize
hyp init cluster-stack

# 3. Edit config.yaml (ensure 2+ AZs!)

# 4. Validate and create
hyp validate && hyp create --region us-east-1

# 5. Set context
hyp set-cluster-context --cluster-name <NAME> --region us-east-1
```

**Job submission**: See [reference/eks-guide.md](reference/eks-guide.md#job-submission)

---

## Slurm Quick Start

```bash
# 1. Prepare lifecycle scripts (use AWS samples)
git clone https://github.com/aws-samples/awsome-distributed-training.git
cd awsome-distributed-training/1.architectures/5.sagemaker-hyperpod/LifecycleScripts/base-config/

# 2. Upload to S3
aws s3 cp . s3://my-bucket/lifecycle-scripts/ --recursive

# 3. Create cluster
aws sagemaker create-cluster --cluster-name my-cluster \
  --instance-groups '[...]' --vpc-config "..."

# 4. Connect via SSM
aws ssm start-session --target <instance-id>
```

**Full workflow**: See [reference/slurm-guide.md](reference/slurm-guide.md)

---

## Model Compatibility (Trainium/Inferentia)

**CRITICAL**: Verify model support before configuring Trainium jobs.

### Check Support

```
WebFetch: https://huggingface.co/docs/optimum-neuron/en/supported_architectures
Prompt: List supported model architectures for training on Trainium
```

### Currently Supported (Training)

| Architecture | Tensor Parallelism | Pipeline Parallelism |
|--------------|-------------------|---------------------|
| Llama, Llama 2, Llama 3 | Yes | Yes |
| Qwen3 | Yes | Yes |
| Granite | Yes | No |

---

## Common Errors (Quick Reference)

| Error | Cause | Solution |
|-------|-------|----------|
| `InvalidParameterException` (EKS) | Single AZ | Add 2+ AZs to config |
| `ml.trn1.2xlarge not found` | Unsupported type | Use `ml.trn1.32xlarge` |
| Training Operator pod fails | Missing Pod Identity | See [EKS guide](reference/eks-guide.md#fix-pod-identity-verification-failure) |
| `Insufficient cpu` | Full node request | Use partial resources |
| `Accelerator request != limit` | Limits mismatch | Set `accelerators_limit` = `accelerators` |
| EFA health check failed | Multi-AZ | Use single subnet with `OverrideVpcConfig` |
| Add-on not supported | K8s version | Check add-on compatibility before upgrade |

**Full troubleshooting**: See [reference/troubleshooting.md](reference/troubleshooting.md)

---

## Infrastructure Requirements

### EFA Single-AZ Requirement

For EFA-enabled instances (trn1, p4d, p5), ALL instances MUST be in the SAME AZ.

### Security Group

Must allow ALL traffic within itself:
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx --protocol all --port -1 --source-group sg-xxx
```

### CIDR Sizing

| Orchestrator | IPs per P5 |
|--------------|------------|
| Slurm | 32 |
| EKS | 81 (includes pods) |

---

## Quota Management

```bash
# Check quota
aws service-quotas get-service-quota \
  --service-code sagemaker --quota-code L-6865522E --region us-east-1

# Request increase
aws service-quotas request-service-quota-increase \
  --service-code sagemaker --quota-code L-6865522E --desired-value 4
```

**Common codes**:
- `L-6865522E`: ml.trn1.32xlarge for cluster usage
- `L-5C4CD236`: ml.p5.48xlarge for cluster usage

---

## Diagnostic Commands

```bash
# Cluster status
aws sagemaker describe-cluster --cluster-name NAME

# List nodes
aws sagemaker list-cluster-nodes --cluster-name NAME

# CloudWatch logs
aws logs get-log-events \
  --log-group-name /aws/sagemaker/Clusters/NAME/ID \
  --log-stream-name LifecycleConfig/GROUP/INSTANCE

# EKS nodes/pods
kubectl get nodes && kubectl get pods -A
```
