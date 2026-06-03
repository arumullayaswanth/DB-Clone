# DB Clone Platform - End to End Deployment & Testing Guide

## Prerequisites

Before deploying, ensure you have:

- AWS account with RDS access
- GitHub repository with Actions enabled
- AWS IAM role configured for OIDC (GitHub Actions)
- A source database to clone from (Aurora cluster OR standard RDS instance)
- (Optional) Slack webhook URL for notifications

## Supported Database Types

| Source Type | Engines | Clone Method |
|-------------|---------|--------------|
| `aurora-cluster` | aurora-mysql, aurora-postgresql | Copy-on-write (fast) |
| `rds-instance` | mysql, postgres | Snapshot + restore |

---

## PART 1: DEPLOYMENT STEPS

### Step 1: Create the IAM Role in AWS

1. Go to AWS IAM Console > Roles > Create Role
2. Select "Web Identity" as trusted entity
3. Choose the GitHub OIDC provider (set up if not existing)
4. Attach a custom policy:

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

5. Name the role: `github-db-clone-role`
6. Copy the Role ARN (e.g., `arn:aws:iam::<account-id>:role/github-db-clone-role`)

---

### Step 2: Configure GitHub OIDC Provider (if not done)

1. Go to AWS IAM > Identity Providers > Add Provider
2. Provider type: OpenID Connect
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click "Get thumbprint" (AWS fetches it automatically)
6. Click Add Provider

Then update the IAM Role trust policy:

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

Replace `<account-id>`, `<org>`, and `<repo>` with your actual values.

---

### Step 3: Set Up GitHub Repository Secrets

Go to your repo > Settings > Secrets and variables > Actions > Secrets tab:

| Secret | Value |
|--------|-------|
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/github-db-clone-role` |
| `SLACK_WEBHOOK_URL` | Your Slack incoming webhook URL (optional) |

NOTE: Both are stored as secrets (not variables) for security.

---

### Step 4: Create the GitHub Environment with Approval Gate

1. Go to repo > Settings > Environments > New Environment
2. Name: `db-approval`
3. Check "Required reviewers"
4. Add reviewers (SRE team members)
5. Save protection rules

---

### Step 5: Verify Repository Structure

Your repo must have this structure:

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

IMPORTANT:
- `config.yaml` MUST be inside `db-clone/` folder (not at repo root)
- `.github/workflows/` MUST be at repo root (GitHub Actions requirement)
- Remove any `.DS_Store` or `__MACOSX/` files before committing

---

### Step 6: Push the Files

```bash
git add .github/ db-clone/ .gitignore README.md
git commit -m "Add DB clone platform"
git push origin main
```

---

### Step 7: Verify Scheduled Workflow is Enabled

1. Go to repo > Actions tab
2. Find "DB Clone - TTL Cleanup"
3. If disabled, click "Enable workflow"

---

### Step 8: Verify All Workflows Have Correct Permissions

Each workflow that uses AWS OIDC must have:

```yaml
permissions:
  contents: read
  id-token: write
```

All workflows in this system already include this.

---

## PART 2: END TO END TESTING

### Test 1: Create Clone - Aurora Cluster (Happy Path)

**Goal:** Verify the full Aurora clone flow works.

```bash
# 1. Create a feature branch
git checkout -b feature/db-clone-test-aurora

# 2. Edit the config file
cat > db-clone/config.yaml << 'EOF'
region: ca-central-1
source_db: test-mysql-cluster
source_type: aurora-cluster
ttl_hours: 2
instance_class: db.t3.medium
EOF

# 3. Commit and push
git add db-clone/config.yaml
git commit -m "Request Aurora DB clone for testing"
git push origin feature/db-clone-test-aurora

# 4. Open a PR
gh pr create --title "DB Clone Request - Aurora Test" --body "Testing Aurora clone"
```

**Expected Results:**
- Workflow "DB Clone - Create" appears in Actions tab
- Status shows "Waiting" (pending approval)
- After approval, workflow runs through:
  - Parse Config (reads source_type: aurora-cluster)
  - Validate Inputs (checks source_type is valid)
  - Validate Source DB Exists (checks Aurora cluster)
  - Validate Engine (auto-detects aurora-mysql/aurora-postgresql)
  - Prevent Duplicate Clone
  - Create Clone - Aurora Cluster (copy-on-write)
  - Fetch Endpoint
  - Comment on PR
  - Notify Slack
- PR gets a comment with endpoint and connection string

**Verification:**
```bash
aws rds describe-db-clusters \
  --region ca-central-1 \
  --query "DBClusters[?contains(DBClusterIdentifier, 'clone')].[DBClusterIdentifier,Status,Endpoint]" \
  --output table
```

---

### Test 2: Create Clone - RDS Instance (Happy Path)

**Goal:** Verify the RDS snapshot+restore flow works.

```bash
git checkout -b feature/db-clone-test-rds

cat > db-clone/config.yaml << 'EOF'
region: ca-central-1
source_db: test-postgres-db
source_type: rds-instance
ttl_hours: 4
instance_class: db.t3.micro
EOF

git add db-clone/config.yaml
git commit -m "Request RDS instance clone for testing"
git push origin feature/db-clone-test-rds
gh pr create --title "DB Clone Request - RDS Test" --body "Testing RDS snapshot clone"
```

**Expected Results:**
- Workflow detects `source_type: rds-instance`
- Creates a snapshot of the source instance
- Waits for snapshot to be available
- Restores instance from snapshot
- Tags include SnapshotId for later cleanup
- PR comment shows endpoint

**Verification:**
```bash
aws rds describe-db-instances \
  --region ca-central-1 \
  --query "DBInstances[?contains(DBInstanceIdentifier, 'clone')].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]" \
  --output table
```

---

### Test 3: Connect to the Clone

**Goal:** Verify the clone is usable.

```bash
# Aurora MySQL
mysql -h <ENDPOINT> -P 3306 -u <username> -p

# PostgreSQL (RDS instance)
psql -h <ENDPOINT> -p 5432 -U <username> -d <dbname>

# Run a simple query
SHOW DATABASES;
SELECT COUNT(*) FROM some_table;
```

---

### Test 4: Validation Failures (Input Validation)

#### Test 4a: Invalid source_type

```yaml
region: ca-central-1
source_db: test-db
source_type: invalid-type
ttl_hours: 24
instance_class: db.t3.medium
```

**Expected:** Fails with: `ERROR: Invalid source_type: invalid-type`

#### Test 4b: TTL exceeds 48 hours

```yaml
region: ca-central-1
source_db: test-db
source_type: aurora-cluster
ttl_hours: 72
instance_class: db.t3.medium
```

**Expected:** Fails with: `ERROR: TTL cannot exceed 48 hours`

#### Test 4c: Missing fields

```yaml
region: ca-central-1
```

**Expected:** Fails with: `ERROR: Missing required fields`

#### Test 4d: Invalid region format

```yaml
region: invalid
source_db: test-db
source_type: aurora-cluster
ttl_hours: 24
instance_class: db.t3.medium
```

**Expected:** Fails with: `ERROR: Invalid region format`

---

### Test 5: Source DB Validation

#### Test 5a: Non-existent Aurora cluster

```yaml
region: ca-central-1
source_db: fake-cluster-doesnt-exist
source_type: aurora-cluster
ttl_hours: 4
instance_class: db.t3.medium
```

**Expected:** Fails with: `ERROR: Aurora cluster 'fake-cluster-doesnt-exist' does not exist`

#### Test 5b: Non-existent RDS instance

```yaml
region: ca-central-1
source_db: fake-instance-doesnt-exist
source_type: rds-instance
ttl_hours: 4
instance_class: db.t3.micro
```

**Expected:** Fails with: `ERROR: RDS instance 'fake-instance-doesnt-exist' does not exist`

#### Test 5c: Unsupported engine

Point to a DB with an unsupported engine (e.g., Oracle, SQL Server).

**Expected:** Fails with: `ERROR: Unsupported engine: oracle-ee`

---

### Test 6: Duplicate Clone Prevention

1. Complete Test 1 (clone exists for a PR)
2. Push another commit to the same PR branch:

```bash
git checkout feature/db-clone-test-aurora
echo "# trigger" >> db-clone/config.yaml
git add db-clone/config.yaml
git commit -m "Trigger re-run"
git push
```

**Expected:** Fails with: `ERROR: A clone already exists for this PR`

---

### Test 7: Extend TTL

1. Go to Actions > "DB Clone - Extend" > Run workflow
2. Fill in:
   - cluster_name: `<clone-name-from-test-1>`
   - region: `ca-central-1`
   - extend_hours: `12`
3. Run workflow

**Expected:**
- Validates name contains '-clone-'
- Validates Platform, ManagedBy, Expiry tags
- Warns if you're not the original requester
- Updates Expiry tag

**Verification:**
```bash
CLUSTER_ARN=$(aws rds describe-db-clusters \
  --region ca-central-1 \
  --db-cluster-identifier <CLONE_NAME> \
  --query 'DBClusters[0].DBClusterArn' \
  --output text)

aws rds list-tags-for-resource \
  --region ca-central-1 \
  --resource-name "$CLUSTER_ARN" \
  --query "TagList[?Key=='Expiry'].Value" \
  --output text
```

---

### Test 8: Extend Validation Failures

#### Test 8a: Non-clone name

Run "DB Clone - Extend" with cluster_name: `production-db`

**Expected:** Fails with error about missing '-clone-'.

#### Test 8b: Extend beyond 48 hours

Run with extend_hours: `72`

**Expected:** Fails with error about max 48 hours.

#### Test 8c: Non-existent cluster

Run with cluster_name: `fake-clone-12345`

**Expected:** Fails at "Verify Clone Exists" step.

#### Test 8d: Missing tags (manually created clone-like DB)

If a DB has `-clone-` in the name but no Platform/ManagedBy tags:

**Expected:** Fails with tag validation error.

---

### Test 9: Manual Destroy - Aurora Cluster

1. Go to Actions > "DB Clone - Destroy" > Run workflow
2. Fill in:
   - clone_name: `<aurora-clone-name>`
   - region: `ca-central-1`
   - source_type: `aurora-cluster`
3. Run workflow

**Expected:**
- Safety checks pass (name, Platform, ManagedBy, Expiry tags)
- Instances deleted
- Cluster deleted
- Summary printed

---

### Test 10: Manual Destroy - RDS Instance

1. Go to Actions > "DB Clone - Destroy" > Run workflow
2. Fill in:
   - clone_name: `<rds-clone-name>`
   - region: `ca-central-1`
   - source_type: `rds-instance`
3. Run workflow

**Expected:**
- Safety checks pass
- Instance deleted
- Snapshot cleaned up (if SnapshotId tag exists)
- Summary printed

---

### Test 11: Destroy Safety Failures

#### Test 11a: Non-clone name

Run with clone_name: `production-database`

**Expected:** Fails with: `SAFETY ERROR: Name must contain '-clone-'`

#### Test 11b: Missing ManagedBy tag

Run against a DB that has `-clone-` in name but no ManagedBy tag.

**Expected:** Fails with: `SAFETY ERROR: Missing ManagedBy=db-clone-system tag`

---

### Test 12: PR Close Auto-Destroy

```bash
# 1. Create a clone (new branch + PR + approve)
git checkout -b feature/db-clone-test-pr-close
# ... edit config, push, open PR, approve, wait for clone

# 2. Close the PR
gh pr close <PR_NUMBER>
```

**Expected:**
- "DB Clone - PR Close" workflow triggers
- Scans regions: [us-east-1, ca-central-1]
- Finds clone by PR tag + validates Platform + ManagedBy tags
- Deletes clone
- Posts comment on PR confirming destruction

---

### Test 13: TTL Cleanup (Scheduled)

Option A: Wait for a clone to expire.

Option B: Manually trigger:
1. Go to Actions > "DB Clone - TTL Cleanup" > Run workflow
2. Click "Run workflow"

**Expected:**
- Scans regions: [us-east-1, us-west-2, eu-west-1, ca-central-1]
- Finds expired clones (Platform + ManagedBy tags valid)
- Deletes expired clones
- Only sends Slack if something was actually deleted
- Reports active clones with remaining time

---

### Test 14: Slack Notification Behavior

#### Test 14a: With SLACK_WEBHOOK_URL secret set

- Create a clone successfully
- **Expected:** Slack message received

#### Test 14b: Without SLACK_WEBHOOK_URL secret

- **Expected:** Workflow logs: "Slack webhook not configured. Skipping notification." (exit 0, no failure)

#### Test 14c: Cleanup with nothing to delete

- Trigger cleanup manually when no clones are expired
- **Expected:** No Slack notification sent (only fires when deletions happen)

---

### Test 15: Failure Comment on PR

**Goal:** Verify developers see helpful error in PR when workflow fails.

1. Submit a config with a non-existent source DB
2. After approval, workflow fails at validation

**Expected:** PR gets a comment:
```
## DB Clone Failed

Possible causes:
- Source DB does not exist in the specified region
- Unsupported engine
- Invalid source_type
- Duplicate clone already exists for this PR
- AWS IAM permissions issue
- KMS key access denied (for encrypted databases)

Please check workflow logs or contact SRE.
```


---

## PART 3: CHECKLIST

```
DEPLOYMENT:
[ ] IAM Role created with full policy (RDS Read + Aurora Clone + RDS Snapshot + Delete + KMS)
[ ] OIDC provider configured (https://token.actions.githubusercontent.com)
[ ] IAM Role trust policy restricts to your repo
[ ] GitHub secret AWS_ROLE_ARN set
[ ] GitHub secret SLACK_WEBHOOK_URL set (optional)
[ ] GitHub environment db-approval created with reviewers
[ ] config.yaml is inside db-clone/ folder (not at repo root)
[ ] .github/workflows/ is at repo root
[ ] No .DS_Store or __MACOSX/ files in repo
[ ] All workflows have permissions: id-token: write
[ ] Scheduled cleanup workflow enabled in Actions tab

TESTING - AURORA CLUSTER:
[ ] Test 1: Aurora clone creation (copy-on-write) - PASS
[ ] Test 3: Connect to Aurora clone - PASS
[ ] Test 6: Duplicate prevention for Aurora PR - PASS
[ ] Test 9: Manual destroy Aurora cluster - PASS

TESTING - RDS INSTANCE:
[ ] Test 2: RDS instance clone (snapshot+restore) - PASS
[ ] Test 3: Connect to RDS clone - PASS
[ ] Test 10: Manual destroy RDS instance + snapshot cleanup - PASS

TESTING - VALIDATIONS:
[ ] Test 4a: Invalid source_type rejected - PASS
[ ] Test 4b: TTL > 48h rejected - PASS
[ ] Test 4c: Missing fields rejected - PASS
[ ] Test 4d: Invalid region rejected - PASS
[ ] Test 5a: Non-existent Aurora cluster rejected - PASS
[ ] Test 5b: Non-existent RDS instance rejected - PASS
[ ] Test 5c: Unsupported engine rejected - PASS
[ ] Test 6: Duplicate clone rejected - PASS

TESTING - EXTEND:
[ ] Test 7: TTL extension works with tag validation - PASS
[ ] Test 8a: Extend non-clone rejected - PASS
[ ] Test 8b: Extend > 48h rejected - PASS
[ ] Test 8c: Extend non-existent rejected - PASS
[ ] Test 8d: Extend without proper tags rejected - PASS

TESTING - DESTROY:
[ ] Test 9: Aurora destroy works - PASS
[ ] Test 10: RDS destroy + snapshot cleanup works - PASS
[ ] Test 11a: Non-clone name rejected - PASS
[ ] Test 11b: Missing ManagedBy tag rejected - PASS

TESTING - AUTO CLEANUP:
[ ] Test 12: PR close auto-destroy (region matrix) - PASS
[ ] Test 13: TTL cleanup finds and deletes expired - PASS

TESTING - NOTIFICATIONS:
[ ] Test 14a: Slack notification received on create - PASS
[ ] Test 14b: Slack skipped gracefully when not configured - PASS
[ ] Test 14c: Cleanup Slack only fires on actual deletion - PASS
[ ] Test 15: Failure comment posted on PR - PASS

[ ] All clones cleaned up after testing
[ ] All test snapshots cleaned up (for rds-instance tests)
```

---

## PART 4: TROUBLESHOOTING

| Problem | Cause | Fix |
|---------|-------|-----|
| Workflow not triggered | PR does not modify `db-clone/config.yaml` | Ensure config is in `db-clone/` folder |
| "Waiting for deployment" forever | No one approved | Check `db-approval` environment reviewers |
| AWS credential error | Role ARN wrong or trust policy incorrect | Verify OIDC config and role trust policy |
| "Invalid source_type" | Typo in config | Use exactly `aurora-cluster` or `rds-instance` |
| "Source DB does not exist" | Wrong name or wrong region | Check actual DB name in AWS console |
| "Unsupported engine" | DB is Oracle/SQL Server/etc. | Only aurora-mysql, aurora-postgresql, mysql, postgres |
| "Clone already exists for PR" | Previous run created one | Destroy existing clone first |
| Snapshot restore is slow | Normal for RDS instances | Can take 10-30 min depending on DB size |
| KMS error on encrypted DB | IAM role missing KMS permissions | Add KMS actions to the policy |
| PR close didn't destroy | Clone in a region not in the matrix | Add the region to pr-close matrix |
| Cleanup Slack is noisy | Fires every 2 hours | Only fires now when something is deleted |
| Extend denied wrong user | Requester check shows warning | Only original requester should extend (warning only) |
| Destroy fails - missing tags | Clone was created before tagging | Manually delete via AWS console |

---

## PART 5: CLEANUP AFTER TESTING

After all tests are complete, ensure no resources are left behind:

```bash
# List all Aurora clone clusters
aws rds describe-db-clusters \
  --region ca-central-1 \
  --query "DBClusters[?contains(DBClusterIdentifier, 'clone')].[DBClusterIdentifier,Status]" \
  --output table

# List all RDS clone instances
aws rds describe-db-instances \
  --region ca-central-1 \
  --query "DBInstances[?contains(DBInstanceIdentifier, 'clone')].[DBInstanceIdentifier,DBInstanceStatus]" \
  --output table

# List leftover snapshots
aws rds describe-db-snapshots \
  --region ca-central-1 \
  --query "DBSnapshots[?contains(DBSnapshotIdentifier, 'snapshot')].[DBSnapshotIdentifier,Status]" \
  --output table

# If any remain, destroy them via the workflow or AWS CLI
# Or trigger the cleanup workflow manually
```

Delete test branches:
```bash
git branch -D feature/db-clone-test-aurora
git branch -D feature/db-clone-test-rds
git branch -D feature/db-clone-test-pr-close
git push origin --delete feature/db-clone-test-aurora
git push origin --delete feature/db-clone-test-rds
git push origin --delete feature/db-clone-test-pr-close
```
