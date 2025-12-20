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

**Fix:** Request quota increase:

```bash
aws service-quotas request-service-quota-increase \
  --service-code lambda \
  --quota-code L-B99A9384 \
  --desired-value 1000 \
  --region eu-west-1
```

### Issue 3: Account Name Not Found for Undefined

**Error:** `Account Name not found for undefined. Validate that the emails in the parameter ManagementAccountEmail of the AWSAccelerator-InstallerStack and account configs (accounts-config.yaml) match the correct account emails shown in AWS Organizations.`

**Cause:** This is a chicken-and-egg bug in LZA v1.13+. The bootstrap stage tries to look up account IDs for all mandatory accounts (Management, LogArchive, Audit) by querying AWS Organizations. On a fresh deployment, only the Management account exists - LogArchive and Audit haven't been created yet. When the code can't find an account, it returns `undefined`.

**Root Cause in Code:** The error message in `accounts-config.ts` has a bug - it prints the undefined `accountId` variable instead of the account `name`:

```typescript
// Bug: prints "undefined"
throw new Error(`Account Name not found for ${accountId}...`)
// Should be:
throw new Error(`Account Name not found for ${name}...`)
```

**Fix:** Manually create the LogArchive and Audit accounts before running the pipeline:

```bash
# Create LogArchive account
aws organizations create-account \
  --email "YOUR_LOG_ARCHIVE_EMAIL" \
  --account-name "LogArchive" \
  --iam-user-access-to-billing ALLOW

# Create Audit account
aws organizations create-account \
  --email "YOUR_AUDIT_EMAIL" \
  --account-name "Audit" \
  --iam-user-access-to-billing ALLOW

# Check creation status (wait for SUCCEEDED)
aws organizations describe-create-account-status \
  --create-account-request-id <REQUEST_ID_FROM_ABOVE>

# Get the Security OU ID
aws organizations list-organizational-units-for-parent --parent-id <ROOT_ID>

# Move accounts to Security OU
aws organizations move-account \
  --account-id <LOG_ARCHIVE_ACCOUNT_ID> \
  --source-parent-id <ROOT_ID> \
  --destination-parent-id <SECURITY_OU_ID>

aws organizations move-account \
  --account-id <AUDIT_ACCOUNT_ID> \
  --source-parent-id <ROOT_ID> \
  --destination-parent-id <SECURITY_OU_ID>

# Then re-run the pipeline
aws codepipeline start-pipeline-execution \
  --name AWSAccelerator-Pipeline \
  --region eu-west-1
```

**Note:** This issue is related to [GitHub Issue #944](https://github.com/awslabs/landing-zone-accelerator-on-aws/issues/944).

### Issue 4: Lambda Concurrency Insufficient in Child Accounts

**Error:**

```text
Lambda concurrency limit for account 545586473833 in region eu-west-1 is insufficient
Lambda concurrency limit for account 511949651909 in region eu-west-1 is insufficient
```

**Cause:** After creating the LogArchive and Audit accounts, they have the default Lambda concurrency quota (10), which is insufficient for LZA deployment. The quota increase from Issue 2 only applied to the Management account.

**Fix:** Request Lambda quota increase in each child account. However, cross-account access is tricky:

**Problem:** The `OrganizationAccountAccessRole` created in new accounts only trusts the management account root principal. IAM users cannot assume this role directly, even with `AdministratorAccess`.

**Solution:** Use root user password reset to access child accounts:

1. Go to <https://signin.aws.amazon.com/>
2. Select **Root user**
3. Enter the child account email (e.g., `your+log@email.com`)
4. Click **Forgot password?**
5. Complete password reset via email
6. Log in as root user
7. Go to **Service Quotas** → **AWS Lambda** → **Concurrent executions**
8. Request increase to **1000**
9. Repeat for each child account (LogArchive, Audit)

**Alternative (if you have AWS SSO/IAM Identity Center):**
- Go to AWS Organizations Console → AWS accounts
- Click on the account → **Access account**
- Request quota from there

**Note:** Quota increases may take a few minutes to be approved. If the pipeline fails again, wait and retry.

## Current Status: IN PROGRESS

Pipeline running after creating mandatory accounts manually.

## Next Steps

1. ~~Wait for Lambda quota increase~~ ✅ Done
2. ~~Create LogArchive and Audit accounts manually~~ ✅ Done
3. ~~Re-run pipeline~~ ✅ Done
4. Wait for pipeline to complete
5. Approve when email arrives
6. Upload custom config to S3
7. Run pipeline again with our config

## Useful Commands

```bash
# Check pipeline status
aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline --region eu-west-1

# List created accounts
aws organizations list-accounts

# Check quota request status
aws service-quotas list-requested-service-quota-change-history --service-code lambda --region eu-west-1
```
