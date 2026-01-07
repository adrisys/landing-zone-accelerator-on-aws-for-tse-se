# AWS Landing Zone Accelerator (LZA) - Proof of Concept

## Overview

This tutorial documents our deployment of AWS Landing Zone Accelerator in "cheap-mode" - a minimal, cost-effective multi-account AWS environment.

### What is LZA?

LZA is an AWS solution that automates the deployment of a secure, multi-account AWS environment following AWS best practices.

| Component | Description |
|-----------|-------------|
| **LZA Source Code** | CDK application at `awslabs/landing-zone-accelerator-on-aws` |
| **Our Config** | YAML files that define our AWS environment |

**We don't modify the LZA source code** - we only provide configuration files.

## Architecture

```
LZA Source Code (GitHub)          Our Config (S3)
awslabs/landing-zone-accelerator  *.yaml files
= The CDK "engine"                = Our instructions
         |                                |
         +----------- reads --------------+
                        |
                        v
              Our AWS Environment
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
| Branch Name | `release/v1.14.2` |
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
3. Enter the child account email (e.g., `ourteam+log@email.com`)
4. Click **Forgot password?**
5. Complete password reset via email
6. Log in as root user
7. Go to **Service Quotas** → **AWS Lambda** → **Concurrent executions**
8. Request increase to **1000**
9. Repeat for each child account (LogArchive, Audit)

**Alternative (if we have AWS SSO/IAM Identity Center):**
- Go to AWS Organizations Console → AWS accounts
- Click on the account → **Access account**
- Request quota from there

**Note:** Quota increases may take a few minutes to be approved. If the pipeline fails again, wait and retry.

## Recommendation: Use Control Tower + LZA

For new deployments, consider using **AWS Control Tower with LZA** instead of standalone LZA.

### Why Control Tower + LZA?

| Standalone LZA | Control Tower + LZA |
|----------------|---------------------|
| We manage everything | AWS manages base landing zone |
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
| Fresh start, learning | Standalone LZA (this tutorial) |
| Already have Control Tower | Add LZA on top |
| Need maximum flexibility | Standalone LZA |
| Want AWS support | Control Tower + LZA |

**Note:** Migrating from standalone LZA to Control Tower is complex. Choose the approach before deploying.

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

Now all new accounts created in our organization will automatically get a Lambda quota increase request.

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
| **Built-in directory** | Free | Small teams |
| Managed AD | ~$100+/month | Enterprise with AD |
| External IdP (Okta, Azure AD) | Varies | Existing SSO |

### Setup Steps

#### Step 1: Enable Identity Center (Manual - Before Pipeline)

LZA cannot enable Identity Center - we must do it manually:

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

Access the SSO portal at:
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

## Account Strategy: Single vs Per-Participant

For workshops with multiple participants:

| Factor | Single Shared Account | Account Per Participant ✅ |
|--------|----------------------|---------------------------|
| **Security** | Users see each other's resources | Full isolation |
| **Experiments** | Can interfere with each other | Clean sandbox per person |
| **Cost tracking** | Hard to attribute | Per-participant billing |
| **Cleanup** | Complex (whose resources?) | Nuke entire account |
| **Compliance** | Harder to audit | Clear audit trail |

**Recommendation:** For NATO/government, use account per participant for proper isolation.

---

## Networking: Centralized vs Decentralized

### When to Use Centralized Networking (Hub & Spoke)

| Scenario | Why Centralized? |
|----------|------------------|
| On-premises connectivity | One VPN/Direct Connect shared by all accounts |
| Centralized egress | Control all internet traffic through one firewall |
| Shared services | DNS, Active Directory, CI/CD accessible to all |
| Network inspection | All traffic flows through security appliances |
| Compliance | All traffic logged/inspected in one place |

### Decentralized (Isolated Accounts)

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Participant1 │  │ Participant2 │  │ Participant3 │
│  ┌───────┐   │  │  ┌───────┐   │  │  ┌───────┐   │
│  │  VPC  │   │  │  │  VPC  │   │  │  │  VPC  │   │
│  └───────┘   │  │  └───────┘   │  │  └───────┘   │
└──────────────┘  └──────────────┘  └──────────────┘
       │                 │                 │
       ✗ No connection between accounts ✗
```

- Each account = isolated island
- Each participant creates own VPC if needed
- No Transit Gateway cost (~$36/attachment/month)
- No shared NAT Gateway cost

### Centralized (Hub & Spoke)

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Participant1 │  │ Participant2 │  │ Participant3 │
│  ┌───────┐   │  │  ┌───────┐   │  │  ┌───────┐   │
│  │  VPC  │   │  │  │  VPC  │   │  │  │  VPC  │   │
│  └───┬───┘   │  │  └───┬───┘   │  │  └───┬───┘   │
└──────┼───────┘  └──────┼───────┘  └──────┼───────┘
       │                 │                 │
       └────────────┬────┴─────────────────┘
                    │
           ┌────────┴────────┐
           │ Transit Gateway │  (Network Account)
           └────────┬────────┘
                    │
           ┌────────┴────────┐
           │   Perimeter     │
           │  ┌──────────┐   │
           │  │ Firewall │───┼──► Internet
           │  └──────────┘   │
           └─────────────────┘
                    │
              On-Premises (VPN)
```

### Decision Matrix

| If we need... | Centralized? |
|----------------|--------------|
| Participants to talk to each other | ✅ Yes |
| Shared database/service all use | ✅ Yes |
| VPN to on-prem network | ✅ Yes |
| Inspect all traffic for compliance | ✅ Yes |
| Just isolated experiments | ❌ No |
| Participants work independently | ❌ No |

### Cost Comparison

| Setup | Monthly Cost |
|-------|-------------|
| Decentralized (5 accounts, each with NAT) | 5 × $32 = ~$160 |
| Centralized (TGW + 5 attachments + NAT pair) | ~$250 |
| Decentralized (no NAT, public subnets only) | **$0** |

---

## Infrastructure OU: Network & Perimeter Accounts

If we use hub & spoke, Infrastructure OU holds the networking accounts:

```
                            Root
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   ┌─────────┐          ┌─────────────┐        ┌──────────┐
   │Security │          │Infrastructure│       │Workloads │
   │   OU    │          │     OU       │       │    OU    │
   └────┬────┘          └──────┬───────┘       └────┬─────┘
        │                      │                    │
   ┌────┴────┐         ┌───────┴───────┐      ┌────┴────────┐
   │         │         │               │      │             │
   Log    Audit    Network       Perimeter    Participant
   Archive                                     Accounts
```

### What Lives Where

| Account | Contains | Purpose |
|---------|----------|---------|
| **Network** | Transit Gateway, VPN/Direct Connect, DNS | Core connectivity |
| **Perimeter** | Firewall, NAT Gateway, IDS/IPS | Security inspection |
| **Participant** | Their VPC (connected to TGW) | Workloads |

### Why Separate Network & Perimeter?

```
Separation of duties:
- Network team → Network account (connectivity)
- Security team → Perimeter account (inspection/filtering)
```

### Alternative: Combined (Simpler)

```
┌─────────────────────────────────────────┐
│            Network Account              │
│                                         │
│  ┌─────────────┐    ┌─────────────────┐ │
│  │     TGW     │    │  Perimeter VPC  │ │
│  │    VPN/DX   │◄──►│  - Firewall     │ │
│  │             │    │  - NAT          │ │
│  └─────────────┘    └─────────────────┘ │
└─────────────────────────────────────────┘
```

| Option | Accounts | Best For |
|--------|----------|----------|
| **Separate** | Network + Perimeter | Enterprise/Production |
| **Combined** | Network only | Smaller deployments |

---

## Network Account vs Shared Services Account

These are different accounts with different purposes. Even if the same team manages both, keep them separate for better audit trails and blast radius reduction.

### Network Account

**Purpose:** Connectivity infrastructure - the "roads and pipes" of your environment

| Resource | What it does |
|----------|--------------|
| **Transit Gateway** | Central hub connecting all VPCs across accounts |
| **VPN / Direct Connect** | Connection to on-prem / NATO networks |
| **Route 53 Resolver** | Centralized DNS for all accounts |
| **Route 53 Private Hosted Zones** | Internal DNS (e.g., `*.internal.nato`) |
| **IPAM** | IP address management (prevents CIDR conflicts) |
| **VPC Endpoints (shared)** | Private access to AWS services (S3, SSM, etc.) |
| **Network Firewall** | (Or in Perimeter account) Traffic inspection |

### Shared Services Account

**Purpose:** Common applications/tools that multiple accounts use

| Resource | What it does |
|----------|--------------|
| **Active Directory** | User authentication (if using AD) |
| **CI/CD Pipelines** | Jenkins, GitLab, CodePipeline for deployments |
| **Container Registry** | ECR or private registry for Docker images |
| **Artifact Repository** | Artifactory, Nexus (packages, binaries) |
| **Bastion Hosts** | Jump servers for SSH access (if needed) |
| **Monitoring Tools** | Grafana, Prometheus dashboards |
| **Secrets Manager** | Shared secrets across accounts |
| **Service Catalog** | Pre-approved infrastructure templates |

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Network Account                              │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐ │
│  │   Transit   │  │     VPN      │  │    Route 53 Resolver    │ │
│  │   Gateway   │◄─┤  to NATO     │  │    (DNS for all)        │ │
│  └──────┬──────┘  └──────────────┘  └─────────────────────────┘ │
│         │                                                        │
└─────────┼────────────────────────────────────────────────────────┘
          │
          │ (VPC attachments)
          │
┌─────────┼────────────────────────────────────────────────────────┐
│         ▼           Shared Services Account                      │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐ │
│  │   CI/CD     │  │   Container  │  │   Active Directory      │ │
│  │  (GitLab)   │  │   Registry   │  │   (if needed)           │ │
│  └─────────────┘  └──────────────┘  └─────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
          │
          │ (Used by)
          ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Workload Accounts                             │
│   Participant1        Participant2        Participant3           │
└──────────────────────────────────────────────────────────────────┘
```

### For NATO Minimal Setup

Start small, add more as needs grow:

| Network Account | Shared Services Account |
|-----------------|------------------------|
| Transit Gateway | CI/CD pipeline (optional) |
| VPN to NATO (if needed) | Container registry (if using containers) |
| Route 53 Resolver | — |

### Same Team, Separate Accounts?

Even if the same team manages both accounts, keep them separate:

| Reason | Why it matters for NATO |
|--------|------------------------|
| **Audit trail** | Clear logs showing who accessed what |
| **Blast radius** | Compromise of one doesn't affect the other |
| **Compliance** | Easier to demonstrate separation of duties |
| **Future-proofing** | If teams split later, no migration needed |

**Cost:** Separate accounts add ~$4-6/month (Config in extra account). Worth it for NATO-grade security.

---

## Best Practice Architecture (Government/NATO Grade)

We want to do it the right way from the start, so it's ready for sensitive workloads.

### Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                   ROOT                                      │
│                                     │                                       │
│         ┌───────────────────────────┼───────────────────────────┐           │
│         │                           │                           │           │
│         ▼                           ▼                           ▼           │
│    ┌─────────┐              ┌──────────────┐             ┌──────────┐       │
│    │Security │              │Infrastructure│             │ Workloads│       │
│    │   OU    │              │      OU      │             │    OU    │       │
│    └────┬────┘              └──────┬───────┘             └────┬─────┘       │
│         │                          │                          │            │
│    ┌────┴────┐              ┌──────┴──────┐            ┌──────┴──────┐     │
│    │         │              │             │            │             │     │
│  Log      Audit         Network     Perimeter      Participants     │     │
│  Archive                                                             │     │
│    │         │              │             │                          │     │
│    │    ┌────┴────┐    ┌────┴────┐   ┌────┴────┐                    │     │
│    │    │GuardDuty│    │   TGW   │   │Firewall │                    │     │
│    │    │Sec Hub  │    │   VPN   │   │  NAT    │                    │     │
│    │    │Config   │    │  IPAM   │   │DNS FW   │                    │     │
│    │    │Inspector│    │Route 53 │   │  WAF    │                    │     │
│    │    │Macie    │    │Endpoints│   │         │                    │     │
│    │    │Detective│    └─────────┘   └─────────┘                    │     │
│    │    │Access   │                                                  │     │
│    │    │Analyzer │                                                  │     │
│    │    └─────────┘                                                  │     │
│    │                                                                 │     │
│    └── All logs centralized here:                                   │     │
│        - CloudTrail (all API calls)                                 │     │
│        - VPC Flow Logs (all network traffic)                        │     │
│        - Config snapshots                                           │     │
│        - GuardDuty findings                                         │     │
│        - Security Hub findings                                       │     │
│        - DNS query logs                                             │     │
│        - Firewall logs                                              │     │
│        - ALB/NLB access logs                                        │     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Full Best Practices Checklist

#### 🔴 High Priority (Must Have)

| Category | Service | Purpose |
|----------|---------|---------|
| **Foundation** | Control Tower + LZA | Account management & governance |
| **Identity** | Identity Center + External IdP | SSO with Entra ID, Okta, etc. |
| **Networking** | Hub & Spoke with TGW | Centralized traffic control |
| **Threat Detection** | GuardDuty | Detect threats across all accounts |
| **Compliance** | Security Hub | Aggregate & prioritize security findings |
| **Compliance** | AWS Config + Rules | Check resource compliance continuously |
| **Audit** | CloudTrail Org Trail | All API calls logged centrally |
| **Network Logging** | VPC Flow Logs | All network traffic logged |
| **Access Analysis** | IAM Access Analyzer | Find unintended external access |
| **Encryption** | KMS Customer Managed Keys | Control over encryption keys |
| **Encryption** | EBS Default Encryption | All volumes encrypted automatically |
| **Data Protection** | S3 Block Public Access | Org-wide prevention of public buckets |
| **SCPs** | Comprehensive Guardrails | Prevent dangerous actions |

#### 🟡 Medium Priority (Should Have)

| Category | Service | Purpose |
|----------|---------|---------|
| **DNS** | Route 53 Resolver | Centralized DNS for all accounts |
| **DNS** | Route 53 DNS Firewall | Block malicious domains |
| **Private Access** | VPC Endpoints | Access AWS services without internet |
| **Vulnerability** | Inspector | Scan EC2/ECR for vulnerabilities |
| **Backup** | AWS Backup | Centralized backup policies |
| **Instance Mgmt** | SSM Session Manager | No SSH keys, audited access |
| **Instance Security** | IMDSv2 Enforcement | Prevent metadata credential theft |
| **IP Management** | IPAM | Manage CIDR allocation centrally |
| **Network Analysis** | Network Access Analyzer | Find unintended network paths |
| **Egress Control** | NAT Gateway (HA) | Controlled internet egress |
| **Firewall** | Network Firewall or 3rd Party | Deep packet inspection |

#### 🟢 Additional (Nice to Have)

| Category | Service | Purpose |
|----------|---------|---------|
| **Data Classification** | Macie | Find sensitive data in S3 |
| **Investigation** | Detective | Security investigation & visualization |
| **DDoS** | Shield Advanced | Advanced DDoS protection |
| **Web Apps** | WAF | Protect web applications |
| **Governance** | Service Catalog | Pre-approved infrastructure products |
| **Cost** | Budgets & Alerts | Cost governance |
| **Tagging** | Tag Policies | Enforce tagging standards |

### What LZA Configures Automatically

```yaml
# security-config.yaml enables:
centralSecurityServices:
  delegatedAdminAccount: Audit

  guardDuty:
    enable: true
    exportConfiguration:
      destinationBucket: central-logs

  securityHub:
    enable: true
    standards:
      - AWS Foundational Security Best Practices
      - CIS AWS Foundations Benchmark

  macie:
    enable: true

  detective:
    enable: true

accessAnalyzer:
  enable: true

awsConfig:
  enableConfigurationRecorder: true
  ruleSets:
    - rules:
        - name: ec2-instance-no-public-ip
        - name: encrypted-volumes
        - name: s3-bucket-ssl-requests-only
        - name: root-account-mfa-enabled
        - name: iam-user-mfa-enabled
        # ... 50+ more rules

cloudWatch:
  metricSets:
    - metrics:
        - filterName: RootAccountUsage
        - filterName: UnauthorizedAPICalls
        - filterName: IAMPolicyChanges
        # ... security metrics with alarms

keyManagementService:
  keySets:
    - name: Central-Key
      deploymentTargets: ALL
```

### What We'll Deploy

```
1. Control Tower (foundation)
   └── Creates Security OU, LogArchive, Audit automatically
   └── Baseline guardrails enabled

2. LZA on top (customization)
   ├── Infrastructure OU
   │   ├── Network account
   │   │   ├── Transit Gateway
   │   │   ├── Site-to-Site VPN (if needed)
   │   │   ├── Route 53 Resolver
   │   │   ├── IPAM
   │   │   └── VPC Endpoints (shared)
   │   │
   │   └── Perimeter account
   │       ├── Network Firewall / 3rd party firewall
   │       ├── NAT Gateways (HA)
   │       ├── DNS Firewall
   │       └── WAF (if web apps)
   │
   ├── Security OU
   │   ├── LogArchive account
   │   │   ├── CloudTrail logs (all accounts)
   │   │   ├── VPC Flow Logs (all accounts)
   │   │   ├── Config snapshots
   │   │   ├── GuardDuty findings
   │   │   ├── DNS query logs
   │   │   └── Firewall logs
   │   │
   │   └── Audit account
   │       ├── GuardDuty (delegated admin)
   │       ├── Security Hub (delegated admin)
   │       ├── Config (delegated admin)
   │       ├── Inspector (delegated admin)
   │       ├── Macie (delegated admin)
   │       ├── Detective (delegated admin)
   │       ├── IAM Access Analyzer
   │       └── Security dashboards
   │
   ├── Workloads OU
   │   └── Participant accounts
   │       ├── VPC (created by platform team)
   │       ├── TGW attachment
   │       ├── EBS encryption enabled
   │       ├── IMDSv2 required
   │       ├── SSM for instance access
   │       └── SCPs prevent networking changes
   │
   └── Identity Center
       └── Connected to external IdP
       └── Permission sets per role
```

### Cost Estimate (Full Best Practice)

| Component | Monthly Cost |
|-----------|-------------|
| Control Tower | Free |
| GuardDuty (5 accounts) | ~$30-100/mo (usage based) |
| Security Hub | ~$10-50/mo (usage based) |
| Config | ~$20-50/mo (usage based) |
| Transit Gateway attachments (6) | ~$216/mo |
| NAT Gateway (2 AZs) | ~$64/mo |
| Network Firewall | ~$300/mo |
| VPN (if needed) | ~$36/mo |
| VPC Endpoints (shared) | ~$50/mo |
| CloudWatch Logs storage | ~$30/mo |
| S3 log storage | ~$20/mo |
| **Total estimate** | **~$500-900/mo** |

Note: Costs vary based on usage, data transfer, and number of accounts.

---

## Questions for Alejandro

| Question | Why It Matters |
|----------|----------------|
| What IdP are we going to use? | Determines Identity Center config (Entra ID, Okta, etc.) |
| How many participants do we have? | Number of accounts & TGW attachments |
| Do participants share resources? | TGW route tables configuration |
| Do participants need to connect to NATO on-prem systems? | If yes → Site-to-Site VPN in Network account |
| Do participants need to access private resources (no public IP)? | If yes → Client VPN |
| Which firewall vendor (if any)? | Perimeter account config (AWS Network Firewall, FortiGate, Palo Alto, etc.) |

---

## Control Tower + LZA Deployment (Recommended)

This section documents deploying AWS Control Tower first, then LZA on top of it.

### Control Tower vs Organizations: Comprehensive Comparison

The following comparison helps you decide which deployment method is right for your use case:

| Aspect | Without Control Tower (Organizations only) | With Control Tower |
| ------ | ------------------------------------------- | ------------------ |
| **Account Creation** | LZA creates accounts directly via Organizations API | Control Tower Account Factory creates accounts with guardrails pre-applied |
| **Identity/SSO** | Manual IAM setup per account | AWS Identity Center automatically configured with SSO portal |
| **Guardrails** | Only what you define in LZA config (SCPs, Config rules) | Control Tower mandatory guardrails + detective controls + LZA additions |
| **Account Baselines** | LZA deploys baselines via CloudFormation | Control Tower applies its own baseline + LZA adds on top |
| **Drift Detection** | Manual / custom | Control Tower dashboard shows governance drift |
| **Dashboard** | None (just CloudFormation stacks) | Control Tower console shows account compliance status |
| **Log Archive/Audit** | You configure them in LZA | Control Tower requires and auto-configures these accounts |
| **Cost** | Minimal (just LZA resources) | Same + Control Tower overhead (minimal additional cost) |
| **Complexity** | Simpler initial setup | More moving parts, but better long-term governance |
| **Recovery** | Easier to tear down/rebuild | Harder to fully remove Control Tower |

**Recommendations:**

- **Without Control Tower**: Better for dev/test environments, experimentation, or when you want full control and simpler teardown
- **With Control Tower**: Recommended for production environments, enterprise deployments, and when you need governance visibility and centralized SSO

### Prerequisites

1. **Clean AWS Organization** - Only management account (or fresh organization)
2. **Two unique email addresses** for LogArchive and Audit accounts
3. **No existing OUs** named "Security" (Control Tower creates this)
4. **IAM Identity Center** must NOT be enabled (Control Tower enables it)

### Step 1: Enable Control Tower

1. Go to **AWS Console** → **Control Tower**
2. Click **Set up landing zone**
3. Configure the following:

| Setting | Value |
|---------|-------|
| Home Region | `eu-west-1` (or your preferred) |
| Additional Regions | Select regions to govern (optional) |
| Foundational OU | `Security` (default) |
| Additional OU | `Sandbox` (optional) |
| Log Archive email | `yourname+log2@gmail.com` |
| Audit email | `yourname+security2@gmail.com` |
| CloudTrail | Keep enabled (default) |
| S3 log retention | Default is fine |
| KMS encryption | Optional (adds cost) |

4. Review and click **Set up landing zone**
5. ☕ **Wait 45-60 minutes** for setup to complete

### Step 2: Verify Control Tower Setup

After completion, verify in AWS Organizations:

```
Root
├── Security OU (created by Control Tower)
│   ├── LogArchive (created by Control Tower)
│   └── Audit (created by Control Tower)
└── Sandbox OU (if you selected it)
```

Also verify:
- **IAM Identity Center** is now enabled
- **CloudTrail** organization trail is created
- **AWS Config** is enabled in all accounts

### Step 3: Request Lambda Quota Increase

Control Tower doesn't automatically increase Lambda quota. Request it for all accounts:

```bash
# In Management account (run from us-east-1 for quota template)
aws service-quotas associate-service-quota-template --region us-east-1

aws service-quotas put-service-quota-increase-request-into-template \
  --service-code lambda \
  --quota-code L-B99A9384 \
  --desired-value 1000 \
  --aws-region eu-west-1 \
  --region us-east-1

# Verify
aws service-quotas list-service-quota-increase-requests-in-template --region us-east-1
```

New accounts will auto-request quota increase. For existing accounts (LogArchive, Audit), request manually via console or reset password to access each account.

### Step 4: Create GitHub Token Secret

```bash
aws secretsmanager create-secret \
  --name accelerator/github-token \
  --secret-string "ghp_YOUR_GITHUB_TOKEN" \
  --region eu-west-1
```

### Step 5: Update LZA Configuration

Update `config/global-config.yaml`:

```yaml
homeRegion: eu-west-1

controlTower:
  enable: true
  controls: []

logging:
  account: LogArchive
  cloudtrail:
    enable: false  # Control Tower already created org trail
    # ... rest of config
```

Update `config/accounts-config.yaml` to match Control Tower account emails:

```yaml
mandatoryAccounts:
  - name: Management
    email: your-management-email@example.com
  - name: LogArchive
    email: yourname+log2@gmail.com  # Must match Control Tower
  - name: Audit
    email: yourname+security2@gmail.com  # Must match Control Tower
```

### Step 6: Deploy LZA Installer Stack

1. Go to **CloudFormation** → **Create Stack**
2. Use S3 URL: `https://solutions-reference.s3.amazonaws.com/landing-zone-accelerator-on-aws/latest/AWSAccelerator-InstallerStack.template`
3. Configure:

| Parameter | Value |
|-----------|-------|
| Stack Name | `AWSAccelerator-InstallerStack` |
| **Control Tower Environment** | **Yes** ← Important! |
| Source Location | `github` |
| Repository Owner | `awslabs` |
| Repository Name | `landing-zone-accelerator-on-aws` |
| Branch Name | `release/v1.14.2` |
| Configuration Repository Location | `s3` |
| Use Existing Config Repository | `No` |
| Enable Approval Stage | `Yes` |

4. Deploy and wait for pipeline

### Step 7: Upload Configuration

After the Installer stack creates the S3 config bucket:

```bash
# Find the config bucket
aws s3 ls | grep accelerator-config

# Upload your config files
aws s3 sync ./config s3://aws-accelerator-config-ACCOUNT_ID-REGION/
```

### Step 8: Run the Pipeline

1. Go to **CodePipeline** → **AWSAccelerator-Pipeline**
2. Approve when prompted (if approval stage enabled)
3. Wait for all stages to complete (~45-60 minutes)

### Control Tower + LZA Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CONTROL TOWER                           │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ - Account Factory (creates accounts)                    ││
│  │ - Managed Guardrails (SCPs)                             ││
│  │ - Landing Zone baseline                                 ││
│  │ - IAM Identity Center                                   ││
│  │ - CloudTrail org trail                                  ││
│  │ - AWS Config (required)                                 ││
│  └─────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│                          LZA                                │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ - Additional OUs (Infrastructure, Workloads)            ││
│  │ - Custom SCPs                                           ││
│  │ - Networking (TGW, VPCs, VPN)                           ││
│  │ - Security services (GuardDuty, Security Hub, etc.)     ││
│  │ - IAM roles and policies                                ││
│  │ - Custom Config rules                                   ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### What Control Tower Manages vs LZA

| Resource | Managed By |
|----------|------------|
| Security OU | Control Tower |
| LogArchive account | Control Tower |
| Audit account | Control Tower |
| CloudTrail org trail | Control Tower |
| AWS Config baseline | Control Tower |
| IAM Identity Center instance | Control Tower |
| Baseline guardrails | Control Tower |
| Additional OUs | LZA |
| Additional accounts | LZA (via Account Factory) |
| Custom SCPs | LZA |
| Networking | LZA |
| Security services (GuardDuty, etc.) | LZA |
| Permission sets | LZA |
| Custom Config rules | LZA |

### Troubleshooting Control Tower + LZA

#### Issue: "Account email already exists"

**Cause:** Email was used by a closed/suspended account.

**Fix:** Use different email alias (e.g., `+log2` instead of `+log`).

#### Issue: Control Tower setup fails

**Cause:** Pre-existing resources conflict with Control Tower.

**Fix:**

- Delete any existing SCPs (except FullAWSAccess)
- Delete any existing OUs named "Security" or "Sandbox"
- Disable Identity Center if enabled
- Remove any Config recorders/rules

#### Issue: LZA pipeline fails with Control Tower enabled

**Cause:** Config mismatch between Control Tower accounts and LZA config.

**Fix:** Ensure `accounts-config.yaml` emails exactly match Control Tower account emails.

### Migrating from Standalone LZA to Control Tower + LZA

> **Note:** The following issues were encountered specifically because we had previously deployed standalone LZA (without Control Tower) and then attempted to deploy Control Tower + LZA. If you're starting fresh with Control Tower, you won't encounter these problems.

#### Issue: Suspended accounts block Control Tower account creation

**Error:** When enabling Control Tower, it tries to create LogArchive and Audit accounts, but the emails were already used by accounts from the previous standalone LZA deployment that are now suspended/closed.

**Cause:** AWS account emails cannot be reused for 90 days after account closure. The old LogArchive and Audit accounts from standalone LZA still exist in a suspended state.

**Fix:** Use different email aliases for the new Control Tower accounts:

```
# Old (standalone LZA)
adrilab.mail+log@gmail.com      → Suspended
adrilab.mail+security@gmail.com → Suspended

# New (Control Tower)
adrilab.mail+log2@gmail.com      → New LogArchive
adrilab.mail+security2@gmail.com → New Audit
```

#### Issue: "Account not in configuration" validation error

**Error:**

```
Found account with id 545586473833 in OU Root that is not in the configuration.
Account with Id 545586473833 and email adrilab.mail+log@gmail.com is not in the accounts
configuration and is not a member of an ignored OU.
```

**Cause:** The suspended accounts from the previous standalone LZA deployment still exist in AWS Organizations (in the Root OU). LZA validates that all accounts in the organization are either:
1. Defined in `accounts-config.yaml`, OR
2. In an OU marked with `ignore: true`

Since the suspended accounts aren't in our new config and aren't in an ignored OU, validation fails.

**Fix:**

1. Create a "Suspended" OU to hold the old accounts:

```bash
# Get Root ID
aws organizations list-roots --query 'Roots[0].Id' --output text
# Example: r-xxxx

# Create Suspended OU
aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Suspended"
# Note the OU ID (e.g., ou-xxxx-xxxxxxxx)
```

2. Move suspended accounts to the Suspended OU:

```bash
aws organizations move-account \
  --account-id 545586473833 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-xxxxxxxx

aws organizations move-account \
  --account-id 511949651909 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-xxxxxxxx

aws organizations move-account \
  --account-id 231222198517 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-xxxxxxxx
```

3. Update `organization-config.yaml` to ignore the Suspended OU:

```yaml
organizationalUnits:
  - name: Security
  - name: Infrastructure
  - name: Workloads
  - name: Suspended
    ignore: true  # LZA will ignore accounts in this OU
```

4. Delete the failed CloudFormation stack (if any):

```bash
# Check for failed stacks
aws cloudformation list-stacks \
  --stack-status-filter ROLLBACK_COMPLETE \
  --query 'StackSummaries[?starts_with(StackName, `AWSAccelerator`)].StackName'

# Disable termination protection and delete
aws cloudformation update-termination-protection \
  --no-enable-termination-protection \
  --stack-name AWSAccelerator-PrepareStack-ACCOUNT_ID-REGION

aws cloudformation delete-stack \
  --stack-name AWSAccelerator-PrepareStack-ACCOUNT_ID-REGION
```

5. Upload updated config and restart the pipeline:

```bash
# Zip and upload config
cd config && zip -r ../aws-accelerator-config.zip . && cd ..
aws s3 cp aws-accelerator-config.zip \
  s3://aws-accelerator-config-ACCOUNT_ID-REGION/zipped/aws-accelerator-config.zip

# Restart pipeline
aws codepipeline start-pipeline-execution --name AWSAccelerator-Pipeline
```

#### Issue: Identity Center user automatically created

**Observation:** After enabling Control Tower, a user was automatically created in AWS Identity Center.

**Cause:** This is expected behavior. Control Tower automatically sets up AWS Identity Center (formerly AWS SSO) and creates an initial admin user based on the email provided during Control Tower setup.

**This is a benefit, not a problem:**

| Before (standalone LZA) | After (Control Tower) |
| ----------------------- | --------------------- |
| Manual IAM users per account | Single Identity Center user with SSO |
| Separate credentials per account | One login for all accounts |
| Manual role management | Permission Sets applied centrally |

You'll receive an email at the admin email address with instructions to set up your password and access the SSO portal.

### Cost Breakdown (Control Tower + LZA Minimal)

| Service | Monthly Cost |
|---------|-------------|
| Control Tower | Free |
| AWS Config (3 accounts) | ~$6-9 |
| CloudTrail | Free (1 trail) |
| S3 (logs) | ~$1-2 |
| IAM Identity Center | Free |
| **Total baseline** | **~$7-11/month** |

Additional costs if you enable:
| Service | Additional Cost |
|---------|-----------------|
| GuardDuty | ~$10-30/month |
| Security Hub | ~$10-30/month |
| Transit Gateway | ~$36/month + attachments |
| NAT Gateway | ~$32/month per AZ |

