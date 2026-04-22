# Scenario: Secret Accidentally Committed to Git

**Type:** Security Incident
**Difficulty:** Medium
**Frequency:** High — tests security instincts and process knowledge

## The Question

*"A developer just committed an AWS access key to a public GitHub repo. What do you do? Walk me through the full response."*

---

## What the Interviewer Is Testing

- Do you understand that **assume compromised** is the right first posture?
- Do you revoke before investigating (not investigate first)?
- Do you know the full blast radius (Git history, forks, caches)?
- Do you have a prevention mindset, not just a response mindset?
- Do you know the tools?

---

## The Critical Mistake to Avoid

**Wrong:** "First, I'd remove the secret from Git and rewrite history."

**Right:** "First, I'd revoke the secret. The secret must be treated as compromised from the moment it was committed — possibly before you even noticed. Removing from Git doesn't help if it was already cloned/scraped."

---

## Response Playbook

### Step 1: REVOKE immediately (< 5 minutes)

```bash
# AWS access key
aws iam delete-access-key --access-key-id AKIAIOSFODNN7EXAMPLE --user-name ci-user

# Or: console → IAM → Users → ci-user → Security Credentials → Delete key

# Issue a new key for the legitimate use immediately after
aws iam create-access-key --user-name ci-user
```

**Revoke before anything else.** The key is already in git history, potentially already scraped by GitHub secret-scanning bots or malicious actors.

### Step 2: Check for abuse (concurrent with step 1)

```bash
# AWS: check CloudTrail for API calls using this key
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=ci-user \
  --start-time "2026-04-21T00:00:00Z" \
  --end-time "2026-04-21T23:59:59Z"

# Look for:
# - Calls from unexpected IPs / regions
# - Resource creation (EC2, S3 buckets, IAM users)
# - Data exfiltration (large GetObject calls on S3)
# - Privilege escalation attempts (CreateUser, AttachPolicy)
```

### Step 3: Scope the exposure

```bash
# How long was the key in the repo?
git log --all --full-history -- path/to/file-with-secret
# First commit with the secret = exposure start time

# Was the repo public or private?
# Public = assume immediately scraped by automated bots
# Private = lower risk but not zero (insider threat, CI logs)

# Did anyone fork / clone the repo?
# GitHub API: GET /repos/{owner}/{repo}/forks

# Was the secret in CI/CD logs?
# Check GitHub Actions logs, Jenkins logs for the key value
```

### Step 4: Clean Git history (after revocation + scope assessment)

```bash
# Option A: git filter-repo (preferred — faster, safer than filter-branch)
pip install git-filter-repo
git filter-repo --path secrets.env --invert-paths

# Option B: BFG Repo Cleaner
java -jar bfg.jar --delete-files secrets.env
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force

# After rewriting history:
# - Force-push to all branches
# - Ask all collaborators to re-clone (their local copies still have the history)
# - Rotate the key (already done in step 1)
```

> Note: If the repo was public, removing from history doesn't help — it was likely scraped within minutes. History cleanup is hygiene for private repos or to prevent future clones from seeing it.

### Step 5: Notify and document

```
- Notify security team (mandatory in most companies)
- File a security incident report
- Check if the exposure triggers compliance obligations (SOC 2, PCI DSS)
- Communicate to affected stakeholders if data was accessed
```

### Step 6: Prevent recurrence

```bash
# Pre-commit hook (blocks commit if a secret pattern is found)
pip install detect-secrets
detect-secrets scan > .secrets.baseline
# Add to .pre-commit-config.yaml:
#   - repo: https://github.com/Yelp/detect-secrets
#     hooks:
#       - id: detect-secrets

# CI secret scanning (fail the build)
# GitHub: Settings → Code security → Secret scanning → Enable
# GitLab: Settings → Security → Secret Detection → Enable
# TruffleHog in CI:
trufflehog git https://github.com/org/repo --only-verified

# Developer education: show this exact scenario in onboarding
# Process: never use real credentials in dev/test; use IAM roles, not keys
```

---

## AWS-Specific: Use IAM Roles, Not Keys

The root cause of "secret in code" is often that developers use access keys for local dev or CI. The fix is to eliminate static keys entirely:

```
Bad:  Developer has AKID/secret in ~/.aws/credentials or .env file
Good: Developer authenticates via AWS SSO (temporary credentials)

Bad:  CI pipeline uses static access key in GitHub Secrets
Good: CI uses OIDC → GitHub Actions requests temporary token from AWS IAM

# GitHub Actions OIDC (no static key needed):
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1
```

---

## Common Follow-Up Questions

**Q: "The repo is private. Do you still need to revoke?"**  
A: Yes. Assume compromised regardless. Private repos can be accessed by all developers, contractors, CI systems — any of which could be compromised. Revoke, check CloudTrail, then decide on severity.

**Q: "What if the secret was only in the Git history (not the current codebase)?"**  
A: Still compromised. Git history is fully accessible to anyone who clones the repo. Revoke immediately, clean history, force-push.

**Q: "How do you prevent this systematically?"**  
A: Pre-commit hooks (detect-secrets), CI secret scanning, eliminate static keys in favor of IAM roles + OIDC, developer training, mandatory secret rotation, GitHub secret scanning alerts.

## Sources
- [[devops/topics/security-devsecops]]
- [[devops/concepts/secrets-management]]
