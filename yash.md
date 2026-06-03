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

NOTE: Slack is stored as a secret (not a variable) for security.

---

### Step 4: Create the GitHub Environment with Approval Gate

1. Go to repo > Settings > Environments > New Environment
2. Name: `db-approval`
3. Check "Required reviewers"
4. Add reviewers (SRE team members, e.g., Emmett)
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
git commit -m "Add DB clone system"
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

### Test 1: Create a Clone (Happy Path)

**Goal:** Verify the full create flow works including new validations.

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
- After approval, workflow runs through all steps:
  - Parse Config
  - Validate Inputs
  - Generate Clone Name
  - Calculate Expiry
  - Configure AWS Credentials
  - Validate Source DB Exists (NEW)
  - Validate Source DB Engine (NEW)
  - Prevent Duplicate Clone for PR (NEW)
  - Create DB Cluster Clone
  - Create DB Instance
  - Wait for Cluster Available
  - Wait for Instance Available
  - Fetch Endpoint
  - Comment on PR
  - Notify Slack
- PR gets a comment with cluster name, endpoint, and connection string
- Slack notification is sent (if SLACK_WEBHOOK_URL secret is set)

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

### Test 3: Validation Failures (Input Validation)

**Goal:** Verify input validation catches bad configs.

#### Test 3a: TTL exceeds 48 hours

```yaml
region: us-east-1
source_db: cm-mysql-cluster-prd-us-emailtrack
ttl_hours: 72
```

**Expected:** Workflow fails at "Validate Inputs" step with: `ERROR: TTL cannot exceed 48 hours`

#### Test 3b: Invalid region format

```yaml
region: invalid-region
source_db: cm-mysql-cluster-prd-us-emailtrack
ttl_hours: 24
```

**Expected:** Workflow fails at "Validate Inputs" step with: `ERROR: Invalid region format`

#### Test 3c: Missing fields

```yaml
region: us-east-1
```

**Expected:** Workflow fails at "Validate Inputs" step with: `ERROR: Missing required fields`

---

### Test 4: Source DB Validation (NEW)

**Goal:** Verify source DB existence and engine checks work.

#### Test 4a: Non-existent source DB

```yaml
region: us-east-1
source_db: fake-database-that-doesnt-exist
ttl_hours: 24
```

**Expected:** Workflow fails at "Validate Source DB Exists" with: `ERROR: Source DB 'fake-database-that-doesnt-exist' does not exist`

#### Test 4b: Wrong engine (if you have a non-MySQL Aurora cluster)

```yaml
region: us-east-1
source_db: some-postgresql-cluster
ttl_hours: 24
```

**Expected:** Workflow fails at "Validate Source DB Engine" with: `ERROR: Only aurora-mysql is supported`

---

### Test 5: Duplicate Clone Prevention (NEW)

**Goal:** Verify that a second clone cannot be created for the same PR.

1. Complete Test 1 (clone already exists for a PR)
2. Push another commit to the same PR branch:

```bash
git checkout feature/db-clone-test-1
echo "# trigger" >> db-clone/config.yaml
git add db-clone/config.yaml
git commit -m "Trigger re-run"
git push
```

**Expected:** Workflow fails at "Prevent Duplicate Clone for PR" with: `ERROR: A clone already exists for this PR`

---

### Test 6: Extend TTL

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

### Test 7: Manual Destroy

**Goal:** Verify manual destroy with safety checks.

#### Test 7a: Destroy a valid clone

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

#### Test 7b: Try to destroy a non-clone name

1. Run "DB Clone - Destroy" with:
   - cluster_name: `production-database`

**Expected:** Fails at "Safety Checks" with error about missing '-clone-' in name.

---

### Test 8: PR Close Auto-Destroy

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

### Test 9: TTL Cleanup (Scheduled)

**Goal:** Verify expired clones are automatically destroyed.

Option A: Wait for a clone to expire (if TTL was set to 2 hours).

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
# EXPIRED: xxx-clone-20250603-120000 (expired at 2025-06-03T14:00:00Z)
# Destroyed: xxx-clone-20250603-120000
```

---

### Test 10: Extend Validation Failures

**Goal:** Verify extend workflow rejects bad inputs.

#### Test 10a: Non-clone name

Run "DB Clone - Extend" with cluster_name: `production-db`

**Expected:** Fails with error about missing '-clone-'.

#### Test 10b: Extend beyond 48 hours

Run "DB Clone - Extend" with extend_hours: `72`

**Expected:** Fails with error about max 48 hours.

#### Test 10c: Non-existent cluster

Run "DB Clone - Extend" with cluster_name: `fake-clone-12345`

**Expected:** Fails at "Verify Clone Exists" step.

---

### Test 11: Slack Notification (Secret-based)

**Goal:** Verify Slack notifications work using secrets.

#### Test 11a: With SLACK_WEBHOOK_URL secret set

- Complete a full clone creation
- **Expected:** Slack message received with clone details

#### Test 11b: Without SLACK_WEBHOOK_URL secret set

- Remove or don't set the SLACK_WEBHOOK_URL secret
- Complete a full clone creation
- **Expected:** Workflow logs show "Slack webhook not configured. Skipping notification." and step passes (exit 0)


---

## PART 3: CHECKLIST

Use this checklist to confirm everything is working:

```
DEPLOYMENT:
[ ] IAM Role created with correct permissions
[ ] OIDC provider configured in AWS (https://token.actions.githubusercontent.com)
[ ] IAM Role trust policy restricts to your repo
[ ] GitHub secret AWS_ROLE_ARN set
[ ] GitHub secret SLACK_WEBHOOK_URL set (optional)
[ ] GitHub environment db-approval created with reviewers
[ ] config.yaml is inside db-clone/ folder (not at repo root)
[ ] .github/workflows/ is at repo root
[ ] No .DS_Store or __MACOSX/ files in repo
[ ] All workflows have permissions: id-token: write
[ ] Scheduled cleanup workflow enabled in Actions tab

TESTING:
[ ] Test 1: Clone creation (happy path) - PASS
[ ] Test 2: Clone connectivity - PASS
[ ] Test 3a: TTL > 48h rejected - PASS
[ ] Test 3b: Invalid region rejected - PASS
[ ] Test 3c: Missing fields rejected - PASS
[ ] Test 4a: Non-existent source DB rejected - PASS
[ ] Test 4b: Wrong engine rejected - PASS
[ ] Test 5: Duplicate clone for same PR rejected - PASS
[ ] Test 6: TTL extension works - PASS
[ ] Test 7a: Manual destroy works - PASS
[ ] Test 7b: Non-clone name rejected - PASS
[ ] Test 8: PR close auto-destroy - PASS
[ ] Test 9: TTL cleanup finds expired - PASS
[ ] Test 10a: Extend non-clone rejected - PASS
[ ] Test 10b: Extend > 48h rejected - PASS
[ ] Test 10c: Extend non-existent rejected - PASS
[ ] Test 11a: Slack notification received - PASS
[ ] Test 11b: Slack skipped gracefully when not configured - PASS

[ ] PR comments posted correctly
[ ] All clones cleaned up after testing
```

---

## PART 4: TROUBLESHOOTING

| Problem | Cause | Fix |
|---------|-------|-----|
| Workflow not triggered | PR does not modify `db-clone/config.yaml` | Ensure config is in `db-clone/` folder, not repo root |
| "Waiting for deployment" forever | No one approved | Check `db-approval` environment reviewers |
| AWS credential error | Role ARN wrong or trust policy incorrect | Verify OIDC config and role trust policy |
| "Source DB does not exist" | `source_db` value is wrong | Check the actual cluster name in AWS RDS console |
| "Only aurora-mysql supported" | Source is PostgreSQL or standard RDS | Only Aurora MySQL clusters can be cloned |
| "Clone already exists for PR" | Previous run already created one | Destroy the existing clone first, or close and reopen PR |
| Clone creation timeout | Source DB too large or region issue | Check RDS console for clone status |
| Cleanup not running | Scheduled workflow disabled | Enable in Actions tab |
| Destroy fails on tags | Cluster missing required tags | Manually delete via AWS console |
| Slack not working | `SLACK_WEBHOOK_URL` secret not set or invalid | Check secret value in repo settings |
| Config at wrong path | `config.yaml` is at repo root | Move it to `db-clone/config.yaml` |

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
