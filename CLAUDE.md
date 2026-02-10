# CLAUDE.md - HyperPod Skill Repository

## Repository Overview

This repository contains the Amazon SageMaker HyperPod skill for Claude Code. It provides comprehensive guidance for creating and managing HyperPod clusters with EKS or Slurm orchestration.

## Key Files

- `SKILL.md` - Main skill definition with frontmatter
- `orchestrators/` - EKS and Slurm specific guides
- `references/` - Prerequisites, IAM, networking docs
- `scripts/` - Validation shell scripts
- `examples/` - Config file examples

## Skill Triggers

The skill activates when users mention:
- "hyperpod", "hyp", "ml-cluster"
- Creating HyperPod or distributed training clusters
- EKS or Slurm for ML training
- GPU/Trainium cluster setup

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
