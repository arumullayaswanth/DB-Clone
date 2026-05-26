# 🚀 DB Clone System

A self-service, safe, and automated system for developers to create temporary copies of production databases for testing.

## Overview

Instead of manually cloning databases in the AWS console (risky, error-prone), developers use this system to:
1. Edit a config file
2. Open a PR
3. Get SRE approval
4. Receive a ready-to-use database endpoint

The clone is automatically destroyed when the PR closes or TTL expires.

---

## 🔁 Full Lifecycle

```
Developer edits config.yaml
        ↓
PR opened (feature/db-clone-request)
        ↓
Approval Gate (SRE reviews)
        ↓
Create workflow runs
        ↓
AWS Clone created + tagged
        ↓
Endpoint returned (PR comment + Slack)
        ↓
Developer uses DB
        ↓
(Optional: extend TTL)
        ↓
Cleanup happens:
├── Manual destroy
├── PR closed (auto)
└── TTL expired (scheduled)
```

---

## 📋 How to Request a Clone

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

## ⚙️ Configuration

| Field | Description | Constraints |
|-------|-------------|-------------|
| `region` | AWS region | Valid AWS region format |
| `source_db` | Source cluster identifier | Must exist in the region |
| `ttl_hours` | Time-to-live in hours | 1-48 hours |

---

## 🔐 Approval Gate

The `db-approval` GitHub Environment must be configured with:
- **Required reviewers**: SRE team members (e.g., Emmett)
- **Wait timer**: Optional (e.g., 0 minutes)

This ensures no clone is created without explicit approval.

---

## 🏷️ Tagging Strategy

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

## 📂 Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `db-clone-create.yml` | PR opened/updated | Creates the clone |
| `db-clone-extend.yml` | Manual dispatch | Extends TTL |
| `db-clone-destroy.yml` | Manual dispatch | Destroys a specific clone |
| `db-clone-pr-close.yml` | PR closed | Auto-destroys clone |
| `db-clone-cleanup.yml` | Scheduled (every 2h) | Destroys expired clones |

---

## 🛡️ Safety Features

1. **Approval gate** - No clone without SRE sign-off
2. **TTL enforcement** - Max 48 hours, auto-cleanup
3. **Name validation** - Must contain `-clone-` for destroy
4. **Tag validation** - Must have `Platform=developer-tools` and `Expiry`
5. **Auto-cleanup on PR close** - No orphaned resources
6. **Scheduled cleanup** - Catches anything missed

---

## 🔧 Setup Requirements

### GitHub Secrets
- `AWS_ROLE_ARN` - IAM role with RDS permissions

### GitHub Variables
- `SLACK_WEBHOOK_URL` (optional) - For Slack notifications

### GitHub Environment
- `db-approval` - With required reviewers configured

### AWS IAM Permissions Required
```json
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
    "rds:AddTagsToResource"
  ],
  "Resource": "*"
}
```

---

## 🚨 Troubleshooting

| Issue | Solution |
|-------|----------|
| Workflow not triggering | Ensure PR modifies `db-clone/config.yaml` |
| Approval not appearing | Check `db-approval` environment config |
| Clone creation fails | Verify AWS credentials and source DB exists |
| Cleanup not running | Check scheduled workflow is enabled |
| Cannot destroy | Verify cluster has required tags |
