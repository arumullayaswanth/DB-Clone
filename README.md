# DB Clone Platform

A self-service, safe, and automated platform for developers to create temporary copies of databases for testing.

Supports both Aurora clusters and standard RDS instances — developers don't need to know the AWS implementation details.

## Supported Databases

| Source Type | Engines | Clone Method |
|-------------|---------|--------------|
| `aurora-cluster` | aurora-mysql, aurora-postgresql | Copy-on-write (fast, cheap) |
| `rds-instance` | mysql, postgres | Snapshot + restore |

---

## Full Lifecycle

```
Developer edits db-clone/config.yaml
        |
PR opened (feature/db-clone-request)
        |
Approval Gate (SRE reviews)
        |
Create workflow runs:
  - Validates inputs + source_type
  - Checks source DB exists
  - Detects engine automatically
  - Validates engine is supported
  - Prevents duplicate clone for PR
  - Branches: Aurora clone OR Snapshot restore
        |
Clone created + tagged
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
<repo-root>/
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

For Aurora cluster:
```yaml
region: ca-central-1
source_db: my-aurora-cluster
source_type: aurora-cluster
ttl_hours: 24
instance_class: db.t3.medium
```

For standard RDS instance:
```yaml
region: ca-central-1
source_db: my-postgres-db
source_type: rds-instance
ttl_hours: 4
instance_class: db.t3.micro
```

3. Open a PR
4. Wait for SRE approval
5. Clone endpoint will be posted as a PR comment

---

## Configuration

| Field | Description | Constraints |
|-------|-------------|-------------|
| `region` | AWS region | Valid AWS region format |
| `source_db` | Source cluster/instance identifier | Must exist in the region |
| `source_type` | `aurora-cluster` or `rds-instance` | Determines clone strategy |
| `ttl_hours` | Time-to-live in hours | 1-48 hours |
| `instance_class` | DB instance class | e.g., db.t3.medium, db.r5.large |

---

## Approval Gate

The `db-approval` GitHub Environment must be configured with:
- **Required reviewers**: SRE team members
- **Wait timer**: Optional

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
| `SourceType` | aurora-cluster / rds-instance | Clone strategy used |
| `ManagedBy` | `db-clone-system` | Safety filter |
| `SnapshotId` | Snapshot name (rds-instance only) | Cleanup reference |

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
4. **Engine detection** - Auto-detects engine, validates it's supported
5. **Duplicate prevention** - Only one clone per PR allowed
6. **Name validation** - Must contain `-clone-` for destroy
7. **Tag validation** - Must have Platform, ManagedBy, and Expiry tags
8. **Requester check** - Extend warns if not the original requester
9. **Auto-cleanup on PR close** - Scans multiple regions by tag
10. **Scheduled cleanup** - Catches anything missed, only notifies on deletion
11. **Snapshot cleanup** - For RDS instances, removes temporary snapshots

---

## Setup Requirements

### GitHub Secrets
- `AWS_ROLE_ARN` - IAM role ARN for OIDC
- `SLACK_WEBHOOK_URL` - Slack incoming webhook URL (optional)

### GitHub Environment
- `db-approval` - With required reviewers configured

### AWS IAM Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RDSRead",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBClusters",
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusterSnapshots",
        "rds:DescribeDBSnapshots",
        "rds:DescribeDBSubnetGroups",
        "rds:DescribeDBEngineVersions",
        "rds:ListTagsForResource"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AuroraClusterClone",
      "Effect": "Allow",
      "Action": [
        "rds:RestoreDBClusterToPointInTime",
        "rds:RestoreDBClusterFromSnapshot",
        "rds:CreateDBInstance",
        "rds:AddTagsToResource"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDSInstanceCloneViaSnapshot",
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBSnapshot",
        "rds:RestoreDBInstanceFromDBSnapshot",
        "rds:AddTagsToResource",
        "rds:DeleteDBSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDSDelete",
      "Effect": "Allow",
      "Action": [
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KMSForEncryptedDatabases",
      "Effect": "Allow",
      "Action": [
        "kms:DescribeKey",
        "kms:CreateGrant",
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
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
| "Source DB does not exist" | Verify `source_db` matches actual name in the region |
| "Unsupported engine" | Only aurora-mysql, aurora-postgresql, mysql, postgres supported |
| "Invalid source_type" | Use `aurora-cluster` or `rds-instance` |
| "Clone already exists for PR" | Destroy existing clone first |
| Snapshot restore slow | Standard RDS snapshots take longer than Aurora clones |
| KMS error | Ensure IAM role has KMS permissions for encrypted DBs |
| Cleanup not running | Enable scheduled workflow in Actions tab |
| Slack not working | Check `SLACK_WEBHOOK_URL` secret |
