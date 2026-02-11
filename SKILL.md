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

You are an expert in Amazon SageMaker HyperPod, a managed service for provisioning resilient ML training clusters powered by AWS Trainium and NVIDIA GPUs (A100, H100). You help users navigate the complexity of HyperPod setup, configuration, and operations.

## Skill Organization

This skill covers **two orchestrators** with different tooling:

| Orchestrator | Primary Tool | Job Submission |
|--------------|--------------|----------------|
| **EKS** | HyperPod CLI (`hyp`) | PyTorchJob via `hyp create` |
| **Slurm** | AWS CLI | SBATCH scripts |

**Shared capabilities** (both orchestrators):
- Model compatibility verification for Trainium/Inferentia
- Service quota management
- VPC/networking configuration
- IAM role setup
- Troubleshooting and diagnostics

---

# PART 1: SHARED (Both Orchestrators)

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
- Want simplified management via HyperPod CLI

### Choose Slurm When:
- Team has HPC background
- Familiar with SBATCH job submission
- Need traditional HPC scheduling features
- Want simpler, more direct cluster access
- Running primarily batch training jobs

## Instance Types Reference

| Instance Type | Accelerator | GPUs/Chips | Memory | Use Case |
|---------------|-------------|------------|--------|----------|
| ml.p4d.24xlarge | A100 | 8 | 320GB | General training |
| ml.p4de.24xlarge | A100 (80GB) | 8 | 640GB | Large models |
| ml.p5.48xlarge | H100 | 8 | 640GB | Latest gen training |
| ml.trn1.32xlarge | Trainium | 16 | 512GB | Cost-effective |
| ml.trn1n.32xlarge | Trainium | 16 | 512GB | Higher network |

## Model Compatibility Verification (Trainium/Inferentia)

**CRITICAL**: Before configuring training jobs on Trainium or Inferentia instances, you MUST verify that the target model architecture is supported by the AWS Neuron SDK.

### Verification Workflow

**Step 1: Check HuggingFace Optimum Neuron**
```
WebFetch: https://huggingface.co/docs/optimum-neuron/en/supported_architectures
Prompt: List supported model architectures for training on Trainium with tensor parallelism
```

**Step 2: Check AWS Neuron Release Notes**
```
WebFetch: https://awsdocs-neuron.readthedocs-hosted.com/en/latest/about-neuron/whats-new.html
Prompt: Check if [MODEL_NAME] is supported and which SDK version added support
```

**Step 3: Check Model-Specific Requirements**
```
WebFetch: https://huggingface.co/[MODEL_ID]
Prompt: Extract model requirements: transformers version, memory needs, context length
```

### Currently Supported Architectures

**Training:**
| Architecture | Tensor Parallelism | Pipeline Parallelism |
|--------------|-------------------|---------------------|
| Llama, Llama 2, Llama 3 | ✓ | ✓ |
| Qwen3 | ✓ | ✓ |
| Granite | ✓ | ✗ |

**Inference:**
| Architecture | Tasks |
|--------------|-------|
| Llama 3, Llama 4 | text-generation |
| Qwen2, Qwen3, Qwen3Moe | text-generation, feature-extraction |
| Mixtral | text-generation |
| BERT, RoBERTa | feature-extraction, classification |

### Common Compatibility Issues

- **Architecture Not Supported**: Check for similar supported architecture
- **Transformers Version Mismatch**: Qwen3 requires ≥4.51.0
- **Attention Implementation**: Some models need `attn_implementation="eager"`
- **Memory Constraints**: Increase tensor parallelism or reduce sequence length

## CRITICAL: Pre-Creation Validation Checklist

**ALWAYS perform these validation steps BEFORE creating a HyperPod cluster:**

### 1. Validate Instance Type is Supported for Cluster Usage

**CRITICAL**: Not all SageMaker instance types are available for HyperPod clusters. The quota must specifically say "**for cluster usage**".

```bash
# List all cluster usage quotas
aws service-quotas list-service-quotas \
  --service-code sagemaker \
  --region us-east-1 \
  --query 'Quotas[?contains(QuotaName, `for cluster usage`)].[QuotaName,Value]' \
  --output table

# Check specific instance type (MUST include "for cluster usage" in name)
aws service-quotas list-service-quotas \
  --service-code sagemaker \
  --region us-east-1 \
  --query 'Quotas[?contains(QuotaName, `ml.trn1.32xlarge`) && contains(QuotaName, `cluster`)].[QuotaName,Value]' \
  --output table
```

**IMPORTANT - Trainium Instance Limitations:**
| Instance Type | HyperPod Support | Notes |
|---------------|------------------|-------|
| ml.trn1.32xlarge | YES | Default quota: 2 |
| ml.trn1.2xlarge | **NO** | NOT supported for HyperPod clusters |
| ml.trn1n.32xlarge | YES | Requires quota increase |
| ml.trn2.48xlarge | YES | Requires quota increase |

### 2. Validate Availability Zone Requirements

**EKS (CRITICAL)**: Requires subnets in at least **2 different Availability Zones**. Single-AZ will fail with `InvalidParameterException`.

**Slurm**: Single AZ is acceptable. Can use `--vpc-config` with one subnet.

```bash
# Check AZ availability for your instance type
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters Name=instance-type,Values=trn1.32xlarge \
  --region us-east-1 \
  --query 'InstanceTypeOfferings[*].Location' \
  --output text

# EKS config.yaml - MUST have 2+ AZs:
# availability_zone_ids:
#   - use1-az6  # Primary for Trainium workers
#   - use1-az4  # Secondary for EKS HA

# Slurm create_cluster.json - Single AZ OK:
# "VpcConfig": { "Subnets": ["subnet-xxx"], "SecurityGroupIds": ["sg-xxx"] }
```

### 3. Validate Instance Count Against Quota

```bash
# Check total cluster instance limit
aws service-quotas list-service-quotas \
  --service-code sagemaker \
  --region us-east-1 \
  --query 'Quotas[?contains(QuotaName, `Total number of instances allowed across SageMaker HyperPod clusters`)].[QuotaName,Value]'
```

### 4. Validate AZ Capacity

```bash
# Check if the AZ has capacity for your instance type
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters Name=instance-type,Values=<INSTANCE_TYPE> \
  --region <REGION> \
  --query 'InstanceTypeOfferings[*].[InstanceType,Location]' \
  --output table
```

## Quota Management

Check and request quotas before cluster creation:

```bash
# Check current quota
aws service-quotas get-service-quota \
  --service-code sagemaker \
  --quota-code L-6865522E \
  --region us-east-1

# Request increase
aws service-quotas request-service-quota-increase \
  --service-code sagemaker \
  --quota-code L-6865522E \
  --desired-value 4 \
  --region us-east-1
```

**Common Quota Codes:**
- `L-6865522E`: ml.trn1.32xlarge for cluster usage
- `L-5C4CD236`: ml.p5.48xlarge for cluster usage

## Critical Infrastructure Requirements

### EFA Single-AZ Requirement

**CRITICAL**: For instances with EFA (trn1, p4d, p5), ALL instances MUST be in the SAME Availability Zone.

- Multi-AZ deployments cause EFA health check failures
- Nodes will cycle Pending → ShuttingDown
- Use `OverrideVpcConfig` with ONE subnet

### Security Group Configuration

Security group must allow ALL traffic within itself for EFA:

```yaml
# Correct: Separate SecurityGroupIngress resource
HyperPodSecurityGroupSelfIngress:
  Type: AWS::EC2::SecurityGroupIngress
  Properties:
    GroupId: !Ref HyperPodSecurityGroup
    IpProtocol: "-1"
    SourceSecurityGroupId: !Ref HyperPodSecurityGroup
```

### CIDR Sizing

- **Slurm**: 32 IPs per P5 instance
- **EKS**: 81 IPs per P5 instance (includes pod IPs)

---

# PART 2: EKS ORCHESTRATOR (HyperPod CLI)

## Prerequisites

```bash
# Install HyperPod CLI
pip install sagemaker-hyperpod

# Verify installation
hyp --help

# Also required: kubectl, helm v3
```

## EKS Cluster Creation Workflow

### MANDATORY: Pre-Flight Validation

**BEFORE running any cluster creation commands, you MUST:**

1. **Verify instance type is supported for cluster usage** (see Pre-Creation Validation Checklist)
2. **Configure 2+ Availability Zones** in config.yaml for EKS
3. **Confirm quota is sufficient** for desired instance count
4. **Verify AZ capacity** for your instance type

### Option A: Using HyperPod CLI (Recommended)

```bash
# 0. Run pre-flight validation (REQUIRED)
aws service-quotas list-service-quotas --service-code sagemaker --region us-east-1 \
  --query 'Quotas[?contains(QuotaName, `<INSTANCE_TYPE>`) && contains(QuotaName, `cluster`)].[QuotaName,Value]'

# 1. Initialize cluster configuration
hyp init cluster-stack

# 2. Edit config.yaml - CRITICAL: Ensure 2+ AZs for EKS!
# availability_zone_ids:
#   - use1-az6
#   - use1-az4

# 3. Configure cluster parameters
hyp configure --resource-name-prefix my-hyperpod

# 4. Validate configuration
hyp validate

# 5. Create cluster
hyp create --region us-east-1
```

### Option B: Using AWS CLI (Advanced)

Use when you need custom VPC configuration or `OverrideVpcConfig`:

```bash
aws sagemaker create-cluster \
  --cluster-name my-cluster \
  --orchestrator "Eks={ClusterArn=arn:aws:eks:...}" \
  --instance-groups '[{
    "InstanceGroupName": "workers",
    "InstanceType": "ml.trn1.32xlarge",
    "InstanceCount": 2,
    "OverrideVpcConfig": {
      "SecurityGroupIds": ["sg-xxx"],
      "Subnets": ["subnet-single-az"]
    },
    ...
  }]' \
  --vpc-config "SecurityGroupIds=sg-xxx,Subnets=subnet-1,subnet-2"
```

## EKS Prerequisites Setup

Before creating HyperPod cluster on EKS:

### 1. Update EKS Authentication Mode

```bash
aws eks update-cluster-config \
  --name <cluster-name> \
  --access-config authenticationMode=API_AND_CONFIG_MAP \
  --region <region>
```

### 2. Install HyperPod Helm Dependencies

```bash
git clone https://github.com/aws/sagemaker-hyperpod-cli.git
cd sagemaker-hyperpod-cli/helm_chart
helm dependencies update HyperPodHelmChart
helm install hyperpod-dependencies HyperPodHelmChart --namespace kube-system
```

Components installed: Kueue, Kubeflow Training Operator, MPI Operator, device plugins

## Job Submission (EKS)

### Using HyperPod CLI

```bash
# Create PyTorch training job
hyp create hyp-pytorch-job \
  --job-name my-training \
  --image 763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training-neuronx:2.1.2 \
  --command '[python, train.py]' \
  --instance-type ml.trn1.32xlarge \
  --node-count 2

# List jobs
hyp get jobs

# Get logs
hyp logs my-training

# Delete job
hyp delete job my-training
```

### Using kubectl Directly

```bash
# Apply PyTorchJob manifest
kubectl apply -f pytorch-job.yaml

# Monitor
kubectl get pytorchjobs
kubectl logs -f -l app=my-training
```

## HyperPod CLI Command Reference

### Cluster Management
```bash
hyp list-cluster                              # List available clusters
hyp set-cluster-context --cluster-name NAME   # Configure kubectl context
hyp get-cluster-context                       # Show current context
hyp list cluster-stack                        # List CloudFormation stacks
hyp describe cluster-stack STACK_NAME         # Stack details
hyp delete cluster-stack STACK_NAME           # Delete stack
```

### Job Management
```bash
hyp init hyp-pytorch-job                      # Initialize job config
hyp create hyp-pytorch-job [options]          # Create job
hyp get jobs                                  # List jobs
hyp logs JOB_NAME                             # View logs
hyp delete job JOB_NAME                       # Delete job
```

## EKS-Specific Errors

**"InvalidParameterException" during EKS cluster creation**
- Root cause: EKS requires subnets in at least 2 Availability Zones
- Resolution: Add a second AZ to config.yaml:
```yaml
availability_zone_ids:
  - use1-az6  # Primary
  - use1-az4  # Secondary for EKS HA
```

**"EKS clusters with CONFIG_MAP authentication mode are not supported"**
- Resolution: Update to `API_AND_CONFIG_MAP` mode (see above)

**"Amazon EKS orchestrator cluster is missing required dependencies"**
- Resolution: Install HyperPod Helm chart (see above)

**"EFA health checks did not run successfully"**
- Root cause: Multi-AZ deployment
- Resolution: Use `OverrideVpcConfig` with single subnet

**"Unable to retrieve subnets"**
- Resolution: Add EC2 VPC permissions to execution role:
  - ec2:DescribeSubnets, ec2:DescribeSecurityGroups
  - ec2:CreateNetworkInterface, ec2:DeleteNetworkInterface

## HyperPod Cluster Creation Errors

**HyperPodClusterStack CREATE_FAILED with unsupported instance type**
- Root cause: Instance type not supported for HyperPod clusters
- Example: ml.trn1.2xlarge is NOT supported, only ml.trn1.32xlarge
- Validation command:
```bash
aws service-quotas list-service-quotas --service-code sagemaker --region us-east-1 \
  --query 'Quotas[?contains(QuotaName, `<INSTANCE_TYPE>`) && contains(QuotaName, `cluster`)]'
```
- Resolution: Use a supported instance type (see Instance Types Reference)

**CloudFormation ROLLBACK_COMPLETE without clear error**
- Check CloudTrail for actual API errors:
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=sagemaker.amazonaws.com \
  --start-time "$(date -u -v-30M +%Y-%m-%dT%H:%M:%SZ)" \
  --query 'Events[*].[EventName,CloudTrailEvent]'
```

---

# PART 3: SLURM ORCHESTRATOR (AWS CLI)

## Slurm vs EKS: Key Differences

| Aspect | Slurm | EKS |
|--------|-------|-----|
| **AZ Requirement** | Single AZ OK | **2+ AZs Required** |
| **VPC Config** | Optional (uses HyperPod default) | Required |
| **Configuration Files** | `create_cluster.json` + `provisioning_parameters.json` | `config.yaml` |
| **Lifecycle Scripts** | Required (uploaded to S3) | Not required |
| **Job Submission** | SBATCH scripts | PyTorchJob via kubectl/hyp |
| **Access Method** | SSM Session Manager | kubectl |

## CRITICAL: Slurm Pre-Creation Checklist

**ALWAYS perform these validations BEFORE creating a Slurm cluster:**

### 1. Instance Type Quota Check
```bash
# Verify instance type is supported for cluster usage
aws service-quotas list-service-quotas \
  --service-code sagemaker \
  --region us-east-1 \
  --query 'Quotas[?contains(QuotaName, `<INSTANCE_TYPE>`) && contains(QuotaName, `cluster`)].[QuotaName,Value]'
```

### 2. Validate Configuration Files
```bash
# Clone awsome-distributed-training for validation script
git clone https://github.com/aws-samples/awsome-distributed-training.git
cd awsome-distributed-training/1.architectures/5.sagemaker-hyperpod/

# Run validation BEFORE cluster creation
python3 validate-config.py \
  --cluster-config create_cluster.json \
  --provisioning-parameters provisioning_parameters.json
```

The validation script checks:
- Instance group names match between files
- Subnet configurations are valid
- Security group rules (ingress/egress)
- FSx Lustre DNS name and mount name
- Cross-resource consistency

### 3. Instance Group Name Matching (CRITICAL)

**Most common Slurm gotcha**: Instance group names in `create_cluster.json` MUST match those in `provisioning_parameters.json`:

```
create_cluster.json                    provisioning_parameters.json
Instance Groups:                       Slurm Nodes:
- controller-machine          ------>  instance_group: controller-machine
- login-group                 ------>  instance_group: login-group (optional)
- compute-nodes               ------>  instance_group: compute-nodes
```

### 4. EFA Security Group (for GPU/Trainium instances)

Security group MUST allow all traffic within itself:
```bash
# Create self-referencing rule for EFA
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol all \
  --port -1 \
  --source-group sg-xxx
```

## Prerequisites

- AWS CLI v2
- Session Manager Plugin for SSM access
- Lifecycle scripts uploaded to S3

## Slurm Cluster Creation Workflow

### 1. Prepare Lifecycle Scripts

**STRONGLY Recommended**: Use the production-ready lifecycle scripts from AWS Samples:
```bash
git clone https://github.com/aws-samples/awsome-distributed-training.git
cd awsome-distributed-training/1.architectures/5.sagemaker-hyperpod/LifecycleScripts/base-config/
```

Key files in base-config:
| File | Purpose |
|------|---------|
| `lifecycle_script.py` | Primary orchestration script |
| `on_create.sh` | Initial setup during cluster creation |
| `provisioning_parameters.json` | Slurm node configuration |
| `start_slurm.sh` | Slurm daemon startup |
| `mount_fsx.sh` | FSx Lustre mounting |
| `setup_mariadb_accounting.sh` | Local Slurm accounting |
| `install_docker.sh` | Docker installation |

**Custom on_create.sh** (only if you understand lifecycle scripts well):

```bash
#!/bin/bash
set -e

# Install Neuron SDK for Trainium
. /etc/os-release
sudo tee /etc/yum.repos.d/neuron.repo > /dev/null <<EOF
[neuron]
name=Neuron YUM Repository
baseurl=https://yum.repos.neuron.amazonaws.com
enabled=1
gpgcheck=1
gpgkey=https://yum.repos.neuron.amazonaws.com/GPG-PUB-KEY-AMAZON-AWS-NEURON.PUB
EOF

sudo yum install -y aws-neuronx-collectives aws-neuronx-runtime-lib aws-neuronx-tools
```

### 2. Configure provisioning_parameters.json

```json
{
  "version": "1.0.0",
  "workload_manager": "slurm",
  "controller_group": "controller-machine",
  "worker_groups": [
    {
      "instance_group_name": "compute-nodes",
      "partition_name": "compute"
    }
  ],
  "fsx_dns_name": "fs-xxx.fsx.us-east-1.amazonaws.com",
  "fsx_mountname": "xxxxx"
}
```

### 3. Upload to S3

```bash
aws s3 cp lifecycle-scripts/ s3://my-bucket/hyperpod/lifecycle-scripts/ --recursive
```

### 4. Create Cluster

```bash
aws sagemaker create-cluster \
  --cluster-name my-slurm-cluster \
  --instance-groups '[{
    "InstanceGroupName": "controller-machine",
    "InstanceType": "ml.m5.xlarge",
    "InstanceCount": 1,
    "LifeCycleConfig": {
      "SourceS3Uri": "s3://my-bucket/hyperpod/lifecycle-scripts/",
      "OnCreate": "on_create.sh"
    },
    "ExecutionRole": "arn:aws:iam::xxx:role/HyperPodRole"
  }, {
    "InstanceGroupName": "compute-nodes",
    "InstanceType": "ml.trn1.32xlarge",
    "InstanceCount": 2,
    "LifeCycleConfig": {
      "SourceS3Uri": "s3://my-bucket/hyperpod/lifecycle-scripts/",
      "OnCreate": "on_create.sh"
    },
    "ExecutionRole": "arn:aws:iam::xxx:role/HyperPodRole"
  }]' \
  --vpc-config "SecurityGroupIds=sg-xxx,Subnets=subnet-xxx"
```

## Job Submission (Slurm)

### 1. Connect to Head Node

```bash
# Get instance ID
aws sagemaker list-cluster-nodes --cluster-name my-cluster

# Connect via SSM
aws ssm start-session --target i-xxxxx
```

### 2. Submit SBATCH Job

```bash
#!/bin/bash
#SBATCH --job-name=my-training
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --exclusive

srun python train.py
```

### 3. Monitor

```bash
squeue                    # List jobs
sinfo                     # Node status
scancel JOB_ID            # Cancel job
sacct -j JOB_ID           # Job accounting
```

## AWS CLI Reference (Slurm)

```bash
# List clusters
aws sagemaker list-clusters

# Describe cluster
aws sagemaker describe-cluster --cluster-name NAME

# List nodes
aws sagemaker list-cluster-nodes --cluster-name NAME

# Delete cluster
aws sagemaker delete-cluster --cluster-name NAME

# Update software
aws sagemaker update-cluster-software --cluster-name NAME
```

## Slurm-Specific Errors and Solutions

### 1. CloudWatch Logs Not Appearing

**Problem**: Log groups from HyperPod cluster not visible.

**Solution**: By default, logs go to HyperPod platform account. Update CloudWatch agent config:
```bash
# Edit /opt/aws/amazon-cloudwatch-agent/sagemaker_cwagent_config.json
# Update file_path to /var/log/provision/provisioning.log
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s -c \
  file:/opt/aws/amazon-cloudwatch-agent/sagemaker_cwagent_config.json
```

### 2. NCCL Parallel Training Failures

**Problem**: Jobs fail when using NCCL with Slurm.

**Root Cause**: Linux `RemoveIPC=yes` cleans up IPC resources on logout.

**Solution**: Create epilog script for cleanup:
```bash
#!/bin/bash
# /opt/slurm/etc/epilog.sh
for seg in $(ipcs -m | awk -v owner="$SLURM_JOB_USER" '$3 == owner {print $2}'); do
    ipcrm -m "$seg"
done
for file in /dev/shm/nccl-*; do
    [ -e "$file" ] && rm "$file"
done
```

Add to `slurm.conf`: `Epilog="/opt/slurm/etc/epilog.sh"`

### 3. Nodes DOWN/DRAINED After Reboot

**Problem**: Nodes become DOWN/DRAINED after reboot.

**Solutions**:
```bash
# Use Slurm reboot command (NOT sudo reboot)
scontrol reboot nextstate=resume <node_list>

# For GPU instances, increase boot timeout in slurm.conf
TimeToResume=300
```

### 4. OOM Draining Issues

**Problem**: Nodes keep draining due to out-of-memory.

**Solution**: Enable cgroups in `slurm.conf`:
```
TaskPlugin=task/cgroup
```

Configure `/opt/slurm/etc/cgroup.conf`:
```
CgroupAutomount=yes
ConstrainRAMSpace=yes
MaxRAMPercent=99
```

### 5. Docker Not Installed Across Nodes

**Problem**: Docker not available on compute nodes.

**Solution**: Run lifecycle script manually:
```bash
cd /tmp/sagemaker-lifecycle-* && cd src/utils/
srun -N <num_nodes> bash install_docker.sh
```

### 6. Slurmd Not Starting

**Problem**: Slurm daemon fails to start on nodes.

**Solution**:
```bash
ssh <node>
sudo systemctl status slurmd
sudo journalctl -xe  # diagnose
sudo systemctl start slurmd
```

### 7. FSx Lustre Not Mounting

**Problem**: FSx doesn't mount automatically.

**Solution**: Check `provisioning_parameters.json`:
- Verify `fsx_dns_name` format: `fs-xxx.fsx.us-east-1.amazonaws.com`
- Verify `fsx_mountname` matches FSx configuration

### 8. Instance Group Name Mismatch

**Problem**: Cluster creation fails or nodes not properly configured.

**Solution**: Ensure exact name match:
```json
// create_cluster.json
"InstanceGroupName": "compute-nodes"

// provisioning_parameters.json
"instance_group_name": "compute-nodes"  // MUST MATCH EXACTLY
```

---

# PART 4: TROUBLESHOOTING (Both Orchestrators)

## Common Errors

**InsufficientCapacity**
- Check quotas: `aws service-quotas get-service-quota`
- Try different AZ or region
- Consider alternative instance types

**VPCConfigurationError**
- Check subnet CIDR sizing (81 IPs per P5 for EKS)
- Verify security group rules
- Ensure NAT gateway for private subnets

**LifecycleScriptFailed**
- Check S3 bucket permissions
- Validate script syntax
- Review CloudWatch logs: `/aws/sagemaker/Clusters/<cluster>/<id>`
- **Use sample scripts**: If your custom scripts fail, try the production-ready scripts from:
  ```bash
  git clone https://github.com/aws-samples/awsome-distributed-training.git
  cd awsome-distributed-training/1.architectures/5.sagemaker-hyperpod/
  ```

**NodeUnhealthy**
- Check node status via CLI
- Review instance metrics
- Consider node replacement

**Nodes stuck in Pending/ShuttingDown cycle**
- Most common: EFA health check failure (multi-AZ)
- Solution: Single subnet with `OverrideVpcConfig`
- Check CloudWatch logs for specific error

**VPC quota exceeded (max 5 VPCs)**
- Delete unused VPCs: `aws ec2 describe-vpcs`
- Request quota increase

## Diagnostic Commands

```bash
# Check cluster status
aws sagemaker describe-cluster --cluster-name NAME

# List nodes with status
aws sagemaker list-cluster-nodes --cluster-name NAME

# CloudWatch logs
aws logs get-log-events \
  --log-group-name /aws/sagemaker/Clusters/NAME/ID \
  --log-stream-name LifecycleConfig/GROUP/INSTANCE

# EKS-specific
kubectl get nodes
kubectl get pods -A
kubectl describe pod POD_NAME
```

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
- Use Trainium for compatible models (cost-effective)
- Implement job queuing to maximize utilization

### Security
- Use private subnets for compute nodes
- Enable VPC Flow Logs for debugging
- Regularly rotate credentials and keys
