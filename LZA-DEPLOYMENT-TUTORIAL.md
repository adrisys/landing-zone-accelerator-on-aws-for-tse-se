# AWS Landing Zone Accelerator (LZA) - Proof of Concept

## Overview

This tutorial documents our deployment of AWS Landing Zone Accelerator in "cheap-mode" - a minimal, cost-effective multi-account AWS environment.

### What is LZA?

LZA is an AWS solution that automates the deployment of a secure, multi-account AWS environment following AWS best practices.

| Component | Description |
|-----------|-------------|
| **LZA Source Code** | CDK application at `awslabs/landing-zone-accelerator-on-aws` |
| **Your Config** | YAML files that define your AWS environment |

**You don't modify the LZA source code** - you only provide configuration files.

## Architecture

```
LZA Source Code (GitHub)          Your Config (S3)
awslabs/landing-zone-accelerator  *.yaml files
= The CDK "engine"                = Your instructions
         |                                |
         +----------- reads --------------+
                        |
                        v
              Your AWS Environment
              Accounts, OUs, IAM, CloudTrail...
```

## Cheap-Mode Configuration

We disabled expensive services to minimize costs:

| Service | Status | Reason |
|---------|--------|--------|
| Transit Gateway | Disabled | ~$36/month base cost |
| NAT Gateways | Disabled | ~$32/month per AZ |
| VPC Flow Logs | Disabled | Storage costs |
| GuardDuty | Disabled | Per-event charges |
| Macie | Disabled | S3 scanning costs |
| Security Hub | Disabled | Per-check charges |
| Detective | Disabled | Data analysis costs |
| AWS Config Rules | Disabled | Per-rule evaluation |
| VPCs | None created | No workloads yet |

### What We Kept (Free/Minimal Cost)

- AWS Organizations (free)
- CloudTrail (1 trail free)
- AWS Config Recorder (required by LZA)
- S3 Block Public Access (free)
- EBS Default Encryption (free)
- IAM Password Policy (free)

## Configuration Files

Located in `config/`:

| File | Purpose |
|------|---------|
| `accounts-config.yaml` | Defines Management, LogArchive, Audit accounts |
| `global-config.yaml` | Regions, log retention, Control Tower settings |
| `iam-config.yaml` | IAM roles, Identity Center (empty for cheap-mode) |
| `network-config.yaml` | VPCs, subnets, TGW (minimal - just deletes default VPCs) |
| `organization-config.yaml` | OUs and SCPs |
| `security-config.yaml` | Security services configuration |
| `replacements-config.yaml` | Template variables |

## Deployment Steps

### Prerequisites

1. AWS Management Account with Organizations enabled
2. GitHub Personal Access Token with `repo` scope
3. Three email addresses for mandatory accounts

### Step 1: Create GitHub Token Secret

```bash
aws secretsmanager create-secret \
  --name accelerator/github-token \
  --secret-string "ghp_YOUR_GITHUB_TOKEN" \
  --region eu-west-1
```

The secret name MUST be `accelerator/github-token` - it's hardcoded in LZA.

### Step 2: Deploy Installer Stack

1. Go to CloudFormation - Create Stack
2. Use S3 URL: `https://solutions-reference.s3.amazonaws.com/landing-zone-accelerator-on-aws/latest/AWSAccelerator-InstallerStack.template`

### Step 3: Stack Parameters

| Parameter | Value |
|-----------|-------|
| Stack Name | `AWSAccelerator-InstallerStack` |
| Source Location | `github` |
| Repository Owner | `awslabs` |
| Repository Name | `landing-zone-accelerator-on-aws` |
| Branch Name | `release/v1.14.1` |
| Control Tower Environment | `No` |
| Configuration Repository Location | `s3` |
| Use Existing Config Repository | `No` |
| Enable Approval Stage | `Yes` |

### Step 4: Wait for Pipeline

Pipeline stages: Source - Build - Review - Approval - Deploy

Total time: ~45-60 minutes

## Issues Encountered

### Issue 1: Secret Not Found

**Error:** Secrets Manager can't find the specified secret

**Cause:** Created secret as `github-token` instead of `accelerator/github-token`

**Fix:** Create secret with correct name (see Step 1)

### Issue 2: Lambda Concurrency Insufficient

**Error:** Lambda concurrency for pipeline account in home region eu-west-1 is insufficient

**Cause:** Default Lambda quota too low for LZA

**Fix:** Request quota increase (see Current Status)

## Current Status: PENDING

### Blocker: Lambda Concurrency Quota

```bash
# Check current quota
aws service-quotas get-service-quota \
  --service-code lambda \
  --quota-code L-B99A9384 \
  --region eu-west-1

# Request increase to 1000
aws service-quotas request-service-quota-increase \
  --service-code lambda \
  --quota-code L-B99A9384 \
  --desired-value 1000 \
  --region eu-west-1
```

### After Quota Approved

```bash
aws codepipeline start-pipeline-execution \
  --name AWSAccelerator-Pipeline \
  --region eu-west-1
```

## Next Steps

1. Wait for Lambda quota increase
2. Re-run pipeline
3. Approve when email arrives
4. Upload custom config to S3
5. Run pipeline again with our config

## Useful Commands

```bash
# Check pipeline status
aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline --region eu-west-1

# List created accounts
aws organizations list-accounts

# Check quota request status
aws service-quotas list-requested-service-quota-change-history --service-code lambda --region eu-west-1
```

