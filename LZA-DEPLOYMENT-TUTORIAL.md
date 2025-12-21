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

## Recommendation: Use Control Tower + LZA

For new deployments, consider using **AWS Control Tower with LZA** instead of standalone LZA.

### Why Control Tower + LZA?

| Standalone LZA | Control Tower + LZA |
|----------------|---------------------|
| You manage everything | AWS manages base landing zone |
| Lambda quota issues in new accounts | Control Tower handles account provisioning |
| Manual guardrails setup | AWS-managed guardrails included |
| More flexibility | Better AWS support |
| Complex troubleshooting | Fewer edge cases |

### How It Works

```
┌─────────────────────────────────────────────────────┐
│                  Control Tower                       │
│  - Account Factory (creates accounts)               │
│  - Managed guardrails                               │
│  - Landing zone baseline                            │
├─────────────────────────────────────────────────────┤
│                      LZA                            │
│  - Additional customizations                        │
│  - Advanced networking (TGW, VPCs)                  │
│  - Security hardening beyond CT guardrails          │
│  - Custom SCPs, Config rules                        │
└─────────────────────────────────────────────────────┘
```

### Configuration Change

To use Control Tower with LZA, set in `global-config.yaml`:

```yaml
controlTower:
  enable: true
```

### When to Use What

| Scenario | Recommendation |
|----------|----------------|
| Fresh start, enterprise | Control Tower + LZA |
| Fresh start, learning/PoC | Standalone LZA (this tutorial) |
| Already have Control Tower | Add LZA on top |
| Need maximum flexibility | Standalone LZA |
| Want AWS support | Control Tower + LZA |

**Note:** Migrating from standalone LZA to Control Tower is complex. Choose your approach before deploying.

---

## Issue 5: Lambda Quota Automation for New Accounts

**Problem:** Every new account needs Lambda concurrency quota increased to 1000, which doesn't scale.

**Solution:** Use AWS Service Quotas Request Template to automatically request quota increases for all new accounts:

```bash
# Enable quota template (must run from us-east-1)
aws service-quotas associate-service-quota-template --region us-east-1

# Add Lambda quota to template
aws service-quotas put-service-quota-increase-request-into-template \
  --service-code lambda \
  --quota-code L-B99A9384 \
  --desired-value 1000 \
  --aws-region eu-west-1 \
  --region us-east-1

# Verify
aws service-quotas list-service-quota-increase-requests-in-template --region us-east-1
```

Now all new accounts created in your organization will automatically get a Lambda quota increase request.

**Note:** Quota requests are still subject to AWS approval, but reasonable values (like 1000) are typically auto-approved.

---

## IAM Identity Center (SSO)

### Why Identity Center?

The standard for LZA/enterprise environments is **no IAM users** - use Identity Center for all human access:

| IAM Users | Identity Center |
|-----------|-----------------|
| Long-term credentials (password) | Temporary credentials (session) |
| MFA optional per user | MFA enforced centrally |
| Manage in each account | Manage in one place |
| Access keys can leak | No access keys |

### Identity Source Options

| Option | Cost | Best For |
|--------|------|----------|
| **Built-in directory** | Free | Small teams, PoC |
| Managed AD | ~$100+/month | Enterprise with AD |
| External IdP (Okta, Azure AD) | Varies | Existing SSO |

### Setup Steps

#### Step 1: Enable Identity Center (Manual - Before Pipeline)

LZA cannot enable Identity Center - you must do it manually:

1. AWS Console → **IAM Identity Center**
2. Click **Enable**
3. Choose **"Enable with AWS Organizations"**

#### Step 2: Configure in iam-config.yaml

```yaml
identityCenter:
  name: IdentityCenter
  delegatedAdminAccount: Audit  # Manages Identity Center
  identityCenterPermissionSets:
    - name: AdministratorAccess
      policies:
        awsManaged:
          - arn:aws:iam::aws:policy/AdministratorAccess
      sessionDuration: 60
    - name: ReadOnlyAccess
      policies:
        awsManaged:
          - arn:aws:iam::aws:policy/ReadOnlyAccess
      sessionDuration: 60
    - name: PowerUserAccess
      policies:
        awsManaged:
          - arn:aws:iam::aws:policy/PowerUserAccess
      sessionDuration: 60
  identityCenterAssignments: []  # Configure manually in console
```

#### Step 3: Run Pipeline

Deploy the config - LZA will:
- Delegate Identity Center administration to Audit account
- Create the permission sets

#### Step 4: Create Users and Assignments (Post-Deployment)

1. Log into **Audit account** (delegated admin)
2. Go to **IAM Identity Center**
3. **Users** → Create user (e.g., `adri@example.com`)
4. **Groups** → Create group (e.g., `Admins`)
5. Add user to group
6. **AWS accounts** → Select all accounts → Assign access
7. Choose group → Choose permission set (AdministratorAccess)

#### Step 5: Login via SSO Portal

Access your SSO portal at:
```
https://d-xxxxxxxxxx.awsapps.com/start
```

Select account → Select role → Access console or get CLI credentials.

### CLI Access with Identity Center

```bash
# Configure SSO profile
aws configure sso
# SSO session name: my-sso
# SSO start URL: https://d-xxxxxxxxxx.awsapps.com/start
# SSO region: eu-west-1
# Choose account and role

# Use the profile
aws s3 ls --profile my-sso

# Or set as default
export AWS_PROFILE=my-sso
```

---

## Current Status: IN PROGRESS

Pipeline running with Identity Center configuration.

## Next Steps

1. ~~Wait for Lambda quota increase~~ ✅ Done
2. ~~Create LogArchive and Audit accounts manually~~ ✅ Done
3. ~~Re-run pipeline~~ ✅ Done
4. ~~Set up Lambda quota template for new accounts~~ ✅ Done
5. ~~Enable Identity Center manually~~ ✅ Done
6. Wait for pipeline to complete
7. Create users/groups in Identity Center
8. Assign access to accounts

## Useful Commands

```bash
# Check pipeline status
aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline --region eu-west-1

# List created accounts
aws organizations list-accounts

# Check quota request status
aws service-quotas list-requested-service-quota-change-history --service-code lambda --region eu-west-1

# List Identity Center instances
aws sso-admin list-instances --region eu-west-1
```
