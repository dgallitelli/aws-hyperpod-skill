# CLAUDE.md - HyperPod Skill Repository

## Repository Overview

This repository contains the Amazon SageMaker HyperPod skill for Claude Code. It provides comprehensive guidance for creating and managing HyperPod clusters with EKS or Slurm orchestration.

## Key Files

- `plugins/aws-hyperpod/skills/sagemaker-hyperpod/SKILL.md` - Main skill definition with frontmatter
- `.claude-plugin/marketplace.json` - Plugin manifest with MCP server configuration

## Skill Triggers

The skill activates when users mention:
- "hyperpod", "hyp", "ml-cluster"
- Creating HyperPod or distributed training clusters
- EKS or Slurm for ML training
- GPU/Trainium cluster setup

## Architecture

```
.claude-plugin/
└── marketplace.json          # MCP server config (SageMaker AI)

plugins/aws-hyperpod/skills/sagemaker-hyperpod/
├── SKILL.md                  # Main skill (~400 lines)
├── orchestrators/
│   ├── eks/                  # EKS setup, jobs, troubleshooting
│   └── slurm/                # Slurm setup, jobs, troubleshooting
├── references/               # Prerequisites, IAM, networking, etc.
├── scripts/                  # Validation shell scripts
└── examples/                 # Config file examples
```

## MCP Server

The skill uses the SageMaker AI MCP server:
- `manage_hyperpod_stacks` - Stack management
- `manage_hyperpod_cluster_nodes` - Node operations

## Development

When modifying this skill:

1. **SKILL.md frontmatter** - Keep allowed-tools and hooks current
2. **Scripts** - Ensure they work across macOS and Linux
3. **Examples** - Mark placeholder values clearly with `<REPLACE_*>`
4. **Documentation** - Keep orchestrator guides synchronized

## Testing

Test skill triggers with:
- "Create a HyperPod cluster with 4 P5 instances"
- "Help me set up Slurm for distributed training"
- "Troubleshoot my failing HyperPod cluster"
