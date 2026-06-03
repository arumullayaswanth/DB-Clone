# DB Clone System

A self-service, safe, and automated system for developers to create temporary copies of production databases for testing.

## Overview

Instead of manually cloning databases in the AWS console (risky, error-prone), developers use this system to:
1. Edit a config file
2. Open a PR
3. Get SRE approval
4. Receive a ready-to-use database endpoint

The clone is automatically destroyed when the PR closes or TTL expires.

---

## Full Lifecycle

```
Developer edits config.yaml
        |
PR opened (feature/db-clone-request)
        |
Approval Gate (SRE reviews)
        |
Create workflow runs
        |
  - Validates inputs
  - Validates source DB exists
  - Validates engine is aurora-mysql
  - Checks no duplicate clone for PR
        |
AWS Clone created + tagged
        |
Endpoint returned (PR comment + Slack)
        |
Developer uses DB
        |
(Optional: extend TTL)
        |
Cleanup happens:
+-- Manual destroy
+-- PR closed (auto)
+-- TTL expired (scheduled)
```

---

## Repository Structure

```
DB-CLONE/
+-- .github/workflows/
|   +-- db-clone-create.yml
|   +-- db-clone-extend.yml
|   +-- db-clone-destroy.yml
|   +-- db-clone-pr-close.yml
|   +-- db-clone-cleanup.yml
+-- db-clone/
|   +-- config.yaml
+-- .gitignore
+-- README.md
```

---

## How to Request a Clone

1. Create a new branch: `feature/db-clone-request`
2. Edit `db-clone/config.yaml`:

```yaml
region: us-east-1
source_db: cm-mysql-cluster-prd-us-emailtrack
ttl_hours: 24
```

3. Open a PR
4. Wait for SRE approval
5. Clone endpoint will be posted as a PR comment

---

## Configuration

| Field | Description | Constraints |
|-------|-------------|-------------|
| `region` | AWS region | Valid AWS region format (e.g., us-east-1) |
| `source_db` | Source cluster identifier | Must exist in the region, must be aurora-mysql |
| `ttl_hours` | Time-to-live in hours | 1-48 hours |

---

## Approval Gate

The `db-approval` GitHub Environment must be configured with:
- **Required reviewers**: SRE team members (e.g., Emmett)
- **Wait timer**: Optional (e.g., 0 minutes)

This ensures no clone is created without explicit approval.

---

## Tagging Strategy

Every clone is tagged with:

| Tag | Value | Purpose |
|-----|-------|---------|
| `Platform` | `developer-tools` | Identifies managed resources |
| `Requester` | GitHub username | Ownership tracking |
| `PR` | PR number | Links clone to PR |
| `Expiry` | ISO timestamp | Powers TTL cleanup |
| `SourceDB` | Source identifier | Audit trail |
| `ManagedBy` | `db-clone-system` | Safety filter |

---

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `db-clone-create.yml` | PR opened/updated | Creates the clone |
| `db-clone-extend.yml` | Manual dispatch | Extends TTL |
| `db-clone-destroy.yml` | Manual dispatch | Destroys a specific clone |
| `db-clone-pr-close.yml` | PR closed | Auto-destroys clone |
| `db-clone-cleanup.yml` | Scheduled (every 2h) | Destroys expired clones |

---

## Safety Features

1. **Approval gate** - No clone without SRE sign-off
2. **TTL enforcement** - Max 48 hours, auto-cleanup
3. **Source DB validation** - Fails early if source DB does not exist
4. **Engine validation** - Only aurora-mysql is supported
5. **Duplicate prevention** - Only one clone per PR allowed
6. **Name validation** - Must contain `-clone-` for destroy
7. **Tag validation** - Must have `Platform=developer-tools` and `Expiry`
8. **Auto-cleanup on PR close** - No orphaned resources
9. **Scheduled cleanup** - Catches anything missed

---

## Setup Requirements

### GitHub Secrets
- `AWS_ROLE_ARN` - IAM role ARN for OIDC (e.g., `arn:aws:iam::<account-id>:role/github-db-clone-role`)
- `SLACK_WEBHOOK_URL` - Slack incoming webhook URL (optional, for notifications)

### GitHub Environment
- `db-approval` - With required reviewers configured

### AWS IAM Permissions Required
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:RestoreDBClusterToPointInTime",
        "rds:CreateDBInstance",
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:DescribeDBClusters",
        "rds:DescribeDBInstances",
        "rds:ListTagsForResource",
        "rds:AddTagsToResource",
        "rds:RemoveTagsFromResource"
      ],
      "Resource": "*"
    }
  ]
}
```

### OIDC Trust Policy for the IAM Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:*"
        }
      }
    }
  ]
}
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Workflow not triggering | Ensure PR modifies `db-clone/config.yaml` |
| Approval not appearing | Check `db-approval` environment config |
| Clone creation fails | Verify AWS credentials and source DB exists |
| "Source DB does not exist" | Verify the `source_db` value matches an actual cluster in the region |
| "Only aurora-mysql supported" | Source cluster must be Aurora MySQL, not PostgreSQL or standard RDS |
| "Clone already exists for PR" | A previous run already created a clone; destroy it first or close the PR |
| Cleanup not running | Check scheduled workflow is enabled in Actions tab |
| Cannot destroy | Verify cluster has required tags (Platform, Expiry) |
| Slack not working | Check `SLACK_WEBHOOK_URL` secret is set correctly |
