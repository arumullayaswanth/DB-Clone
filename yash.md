# DB Clone System - End to End Deployment & Testing Guide

## Prerequisites

Before deploying, ensure you have:

- AWS account with RDS access
- GitHub repository with Actions enabled
- AWS IAM role configured for OIDC (GitHub Actions)
- A source Aurora MySQL cluster to clone from
- (Optional) Slack webhook URL for notifications

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

5. Name the role: `db-clone-github-actions-role`
6. Copy the Role ARN (e.g., `arn:aws:iam::123456789012:role/db-clone-github-actions-role`)

---

### Step 2: Configure GitHub OIDC Provider (if not done)

1. Go to AWS IAM > Identity Providers > Add Provider
2. Provider type: OpenID Connect
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click Add Provider

---

### Step 3: Set Up GitHub Repository Secrets

Go to your repo > Settings > Secrets and variables > Actions:

| Secret | Value |
|--------|-------|
| `AWS_ROLE_ARN` | `arn:aws:iam::123456789012:role/db-clone-github-actions-role` |

---

### Step 4: Set Up GitHub Repository Variables

Go to your repo > Settings > Secrets and variables > Actions > Variables tab:

| Variable | Value |
|----------|-------|
| `SLACK_WEBHOOK_URL` | Your Slack incoming webhook URL (optional) |

---

### Step 5: Create the GitHub Environment with Approval Gate

1. Go to repo > Settings > Environments > New Environment
2. Name: `db-approval`
3. Check "Required reviewers"
4. Add reviewers (SRE team members, e.g., Emmett)
5. Save protection rules

---

### Step 6: Push the Workflow Files

Ensure the following files are in your repository:

```
db-clone/
  config.yaml
  .github/workflows/
    db-clone-create.yml
    db-clone-extend.yml
    db-clone-destroy.yml
    db-clone-pr-close.yml
    db-clone-cleanup.yml
```

Push to main branch:

```bash
git add db-clone/
git commit -m "Add DB clone system workflows"
git push origin main
```

---

### Step 7: Verify Scheduled Workflow is Enabled

1. Go to repo > Actions tab
2. Find "DB Clone - TTL Cleanup"
3. If disabled, click "Enable workflow"

---

## PART 2: END TO END TESTING

### Test 1: Create a Clone (Happy Path)

**Goal:** Verify the full create flow works.

```bash
# 1. Create a feature branch
git checkout -b feature/db-clone-test-1

# 2. Edit the config file
cat > db-clone/config.yaml << 'EOF'
region: us-east-1
source_db: cm-mysql-cluster-prd-us-emailtrack
ttl_hours: 2
EOF

# 3. Commit and push
git add db-clone/config.yaml
git commit -m "Request DB clone for testing"
git push origin feature/db-clone-test-1

# 4. Open a PR via GitHub UI or CLI
gh pr create --title "DB Clone Request - Test" --body "Testing clone system"
```

**Expected Results:**
- Workflow "DB Clone - Create" appears in Actions tab
- Status shows "Waiting" (pending approval)
- Approve the deployment in the environment review
- After approval, workflow runs through all steps
- PR gets a comment with cluster name, endpoint, and connection string
- Slack notification is sent (if configured)

**Verification:**
```bash
# Check the clone exists in AWS
aws rds describe-db-clusters \
  --region us-east-1 \
  --query "DBClusters[?contains(DBClusterIdentifier, 'clone')].[DBClusterIdentifier,Status,Endpoint]" \
  --output table

# Verify tags
CLUSTER_ARN=$(aws rds describe-db-clusters \
  --region us-east-1 \
  --db-cluster-identifier <CLONE_NAME> \
  --query 'DBClusters[0].DBClusterArn' \
  --output text)

aws rds list-tags-for-resource \
  --region us-east-1 \
  --resource-name "$CLUSTER_ARN" \
  --output table
```

**Expected Tags:**
- Platform = developer-tools
- Requester = your-github-username
- PR = PR number
- Expiry = timestamp (2 hours from creation)
- SourceDB = cm-mysql-cluster-prd-us-emailtrack
- ManagedBy = db-clone-system

---

### Test 2: Connect to the Clone

**Goal:** Verify the clone is usable.

```bash
# Use the endpoint from the PR comment
mysql -h <ENDPOINT> -P 3306 -u <username> -p

# Run a simple query
SHOW DATABASES;
SELECT COUNT(*) FROM some_table;
```

**Expected Results:**
- Connection succeeds
- Data matches production (point-in-time snapshot)

---

### Test 3: Validation Failures

**Goal:** Verify input validation catches bad configs.

#### Test 3a: TTL exceeds 48 hours

```yaml
region: us-east-1
source_db: cm-mysql-cluster-prd-us-emailtrack
ttl_hours: 72
```

**Expected:** Workflow fails at "Validate Inputs" step with error message about TTL.

#### Test 3b: Invalid region format

```yaml
region: invalid-region
source_db: cm-mysql-cluster-prd-us-emailtrack
ttl_hours: 24
```

**Expected:** Workflow fails at "Validate Inputs" step with error about region format.

#### Test 3c: Missing fields

```yaml
region: us-east-1
```

**Expected:** Workflow fails at "Validate Inputs" step with error about missing fields.

---

### Test 4: Extend TTL

**Goal:** Verify TTL extension works.

1. Go to Actions > "DB Clone - Extend" > Run workflow
2. Fill in:
   - cluster_name: `<clone-name-from-test-1>`
   - region: `us-east-1`
   - extend_hours: `12`
3. Run workflow

**Verification:**
```bash
# Check updated Expiry tag
aws rds list-tags-for-resource \
  --region us-east-1 \
  --resource-name "$CLUSTER_ARN" \
  --query "TagList[?Key=='Expiry'].Value" \
  --output text
```

**Expected:** Expiry tag updated to 12 hours from now.

---

### Test 5: Manual Destroy

**Goal:** Verify manual destroy with safety checks.

#### Test 5a: Destroy a valid clone

1. Go to Actions > "DB Clone - Destroy" > Run workflow
2. Fill in:
   - cluster_name: `<clone-name-from-test-1>`
   - region: `us-east-1`
3. Run workflow

**Expected:**
- Safety checks pass (name contains -clone-, tags valid)
- Instances deleted
- Cluster deleted
- Summary printed

#### Test 5b: Try to destroy a non-clone name

1. Run "DB Clone - Destroy" with:
   - cluster_name: `production-database`

**Expected:** Fails at "Safety Checks" with error about missing '-clone-' in name.

---

### Test 6: PR Close Auto-Destroy

**Goal:** Verify auto-cleanup on PR close.

```bash
# 1. Create a new clone (repeat Test 1 with a new branch)
git checkout -b feature/db-clone-test-pr-close

# Edit config, commit, push, open PR, approve, wait for clone

# 2. Close the PR
gh pr close <PR_NUMBER>
```

**Expected:**
- "DB Clone - PR Close" workflow triggers
- Finds the clone by PR tag
- Deletes instances and cluster
- Posts a comment on the PR confirming destruction

**Verification:**
```bash
# Confirm clone no longer exists
aws rds describe-db-clusters \
  --region us-east-1 \
  --db-cluster-identifier <CLONE_NAME> 2>&1 | grep "not found"
```

---

### Test 7: TTL Cleanup (Scheduled)

**Goal:** Verify expired clones are automatically destroyed.

Option A: Wait for the clone from Test 1 to expire (if TTL was set to 2 hours).

Option B: Manually trigger the cleanup:
1. Go to Actions > "DB Clone - TTL Cleanup" > Run workflow
2. Click "Run workflow"

**Expected:**
- Scans all regions in the matrix
- Finds clones with expired Expiry tags
- Deletes expired clones
- Reports active clones with remaining time

**Verification:**
```bash
# Check workflow logs for output like:
# EXPIRED: xxx-clone-20250526-120000 (expired at 2025-05-26T14:00:00Z)
# Destroyed: xxx-clone-20250526-120000
```

---

### Test 8: Extend Validation Failures

**Goal:** Verify extend workflow rejects bad inputs.

#### Test 8a: Non-clone name

Run "DB Clone - Extend" with cluster_name: `production-db`

**Expected:** Fails with error about missing '-clone-'.

#### Test 8b: Extend beyond 48 hours

Run "DB Clone - Extend" with extend_hours: `72`

**Expected:** Fails with error about max 48 hours.

#### Test 8c: Non-existent cluster

Run "DB Clone - Extend" with cluster_name: `fake-clone-12345`

**Expected:** Fails at "Verify Clone Exists" step.

---

## PART 3: CHECKLIST

Use this checklist to confirm everything is working:

```
[ ] IAM Role created with correct permissions
[ ] OIDC provider configured in AWS
[ ] GitHub secret AWS_ROLE_ARN set
[ ] GitHub environment db-approval created with reviewers
[ ] Workflow files pushed to main branch
[ ] Scheduled cleanup workflow enabled

[ ] Test 1: Clone creation - PASS
[ ] Test 2: Clone connectivity - PASS
[ ] Test 3a: TTL > 48h rejected - PASS
[ ] Test 3b: Invalid region rejected - PASS
[ ] Test 3c: Missing fields rejected - PASS
[ ] Test 4: TTL extension works - PASS
[ ] Test 5a: Manual destroy works - PASS
[ ] Test 5b: Non-clone name rejected - PASS
[ ] Test 6: PR close auto-destroy - PASS
[ ] Test 7: TTL cleanup finds expired - PASS
[ ] Test 8a: Extend non-clone rejected - PASS
[ ] Test 8b: Extend > 48h rejected - PASS
[ ] Test 8c: Extend non-existent rejected - PASS

[ ] Slack notifications received (if configured)
[ ] PR comments posted correctly
[ ] All clones cleaned up after testing
```

---

## PART 4: TROUBLESHOOTING

| Problem | Cause | Fix |
|---------|-------|-----|
| Workflow not triggered | PR does not modify `db-clone/config.yaml` | Ensure the file path matches exactly |
| "Waiting for deployment" forever | No one approved | Check environment reviewers |
| AWS credential error | Role ARN wrong or trust policy incorrect | Verify OIDC config and role trust |
| Clone creation timeout | Source DB too large or region issue | Check RDS console for status |
| Cleanup not running | Scheduled workflow disabled | Enable in Actions tab |
| Destroy fails on tags | Clone was created before tagging was added | Manually delete via AWS console |
| Slack not working | Webhook URL not set or invalid | Check SLACK_WEBHOOK_URL variable |

---

## PART 5: CLEANUP AFTER TESTING

After all tests are complete, ensure no resources are left behind:

```bash
# List all clones
aws rds describe-db-clusters \
  --region us-east-1 \
  --query "DBClusters[?contains(DBClusterIdentifier, 'clone')].[DBClusterIdentifier,Status]" \
  --output table

# If any remain, destroy them manually via the workflow or AWS CLI
# Or trigger the cleanup workflow manually
```

Delete test branches:
```bash
git branch -D feature/db-clone-test-1
git branch -D feature/db-clone-test-pr-close
git push origin --delete feature/db-clone-test-1
git push origin --delete feature/db-clone-test-pr-close
```
