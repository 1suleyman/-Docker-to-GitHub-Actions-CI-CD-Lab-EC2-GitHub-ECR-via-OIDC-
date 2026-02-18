# 🚀 Docker to GitHub Actions CI/CD Lab (EC2 → GitHub → ECR via OIDC)

In this lab, I implemented a **CI/CD pipeline using GitHub Actions** to build and push a Docker image to **Amazon ECR**, using **secure OIDC authentication instead of static AWS credentials**.

---

## 📋 Lab Overview

### 🎯 Goal

Build a secure CI/CD workflow that:

* Clones a Docker workshop app
* Tests it locally on EC2
* Pushes source code to GitHub
* Uses GitHub Actions to:

  * Build Docker image
  * Authenticate to AWS via OIDC
  * Push image to Amazon ECR

---

### 📚 Learning Outcomes

* Understand Git local vs remote configuration
* Use `git remote`, `git add`, `git push -u`
* Configure SSH authentication for GitHub from EC2
* Understand HTTPS vs SSH Git authentication
* Set up GitHub OIDC identity provider in AWS
* Create IAM role with trust policy for GitHub
* Understand trust policy vs permission policy
* Lock IAM trust to specific repo and branch
* Push Docker images securely to ECR without access keys

---

# 🛠 Step-by-Step Journey

---

## 🔹 Phase 1 — Prepare the Application Repository

### Step 1: Clone Docker Workshop App

```bash
git clone <docker-workshop-url>
cd getting-started-todo-app
```

✔ Confirms app + Dockerfile exist locally.

---

### Step 2: Test Locally on EC2

```bash
docker build -t getting-started .
docker run -p 3000:3000 getting-started
```

✔ Confirm:

* App runs successfully
* Security group allows port 3000
* Accessible externally if required

---

### Step 3: Confirm Git Status

```bash
git status
git remote -v
```

✔ Verifies:

* Folder is a Git repository
* Remote origin is correctly configured

---

## 🔹 Phase 2 — Push to Your Own GitHub Repository

### Important: Do NOT Initialise GitHub Repo With README

If you already have local commit history:

❌ Initialising GitHub with README creates conflicting commit histories
✔ Create empty repository instead

---

### Step 4: Remove Original Remote (If Cloned)

```bash
git remote remove origin
```

✔ Detaches from original source

---

### Step 5: Add Your Own Remote (SSH Recommended)

```bash
git remote add origin git@github.com:1suleyman/docker-workshop-node.git
```

Confirm:

```bash
git remote -v
```

---

## 🔹 Phase 3 — Fix GitHub Authentication (SSH Setup on EC2)

Initially encountered:

```
Permission denied (publickey)
```

### Root Cause:

EC2 had no SSH key to authenticate to GitHub.

---

### Step 6: Generate SSH Key

```bash
ssh-keygen -t ed25519 -C "github-ec2"
```

Confirm:

```bash
ls -la ~/.ssh
```

---

### Step 7: Add Public Key to GitHub

```bash
cat ~/.ssh/id_ed25519.pub
```

Paste into:

GitHub → Settings → SSH and GPG Keys → New SSH Key

---

### Step 8: Test Authentication

```bash
ssh -T git@github.com
```

✔ Confirm successful authentication

---

### Step 9: Push Code

```bash
git push -u origin main
```

Explanation:

| Component  | Meaning                 |
| ---------- | ----------------------- |
| `git push` | Sends commits to remote |
| `origin`   | Remote nickname         |
| `main`     | Branch                  |
| `-u`       | Sets upstream tracking  |

---

# 🔐 Phase 4 — Configure AWS ECR + OIDC (Secure CI/CD)

---

## Step 10: Create ECR Repository

AWS Console → ECR → Create repository

Example:

```
docker-workshop-node
```

<img width="465" height="60" alt="• Successfully created private repository, docker-workshop-node" src="https://github.com/user-attachments/assets/7221eafa-d2d8-467c-ac05-c9320b60e7f8" />

---

## Step 11: Create GitHub OIDC Identity Provider (One-Time Setup)

IAM → Identity Providers → Add provider

* Provider type: OpenID Connect
* Provider URL:

```
https://token.actions.githubusercontent.com
```

* Audience:

```
sts.amazonaws.com
```

✔ This establishes AWS trust in GitHub’s identity system.

---

## Step 12: Create IAM Role for GitHub Actions

IAM → Create Role → Web Identity

Select:

* Provider: GitHub OIDC
* Repository: 1suleyman/docker-workshop-node
* Branch: main

AWS auto-generates trust policy.

---

### 🔎 Trust Policy (Security Core)

Critical section:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  },
  "StringLike": {
    "token.actions.githubusercontent.com:sub": "repo:1suleyman/docker-workshop-node:ref:refs/heads/main"
  }
}
```

✔ This prevents any other repo from assuming this role.

---

## Step 13: Attach Permissions to Role

Attach ECR permissions such as:

* AmazonEC2ContainerRegistryFullAccess
  (or preferably a least-privilege custom policy)

---

### 🔎 Important Distinction

| Policy Type       | Purpose                 |
| ----------------- | ----------------------- |
| Trust Policy      | Who can assume the role |
| Permission Policy | What the role can do    |

---

# ⚙️ Phase 5 — GitHub Actions Workflow

Workflow handles:

1. Checkout code
2. Authenticate to AWS via OIDC
3. Login to ECR
4. Build Docker image
5. Push image to ECR

Secure authentication achieved without storing AWS keys in GitHub.

<img width="705" height="627" alt="Amazon Elastic" src="https://github.com/user-attachments/assets/96916b27-5b79-4b5e-84ee-4773ad701bcf" />

---

# 🧠 Key Architectural Concepts Learned

* Git is distributed — push & fetch are intentional
* `origin` is just a nickname for a remote
* SSH is superior to HTTPS + passwords
* GitHub no longer supports password authentication
* OIDC eliminates static AWS credentials
* Trust policy restricts identity
* `sub` condition is the major security boundary
* IAM separates identity trust from capability permissions
* CI/CD pipelines should use temporary credentials

---

# ✅ Key Commands Summary

| Task                  | Command                                    |
| --------------------- | ------------------------------------------ |
| Check Git repo status | `git status`                               |
| View remote config    | `git remote -v`                            |
| Remove remote         | `git remote remove origin`                 |
| Add SSH remote        | `git remote add origin git@github.com:...` |
| Generate SSH key      | `ssh-keygen -t ed25519`                    |
| Push with tracking    | `git push -u origin main`                  |
| Test SSH auth         | `ssh -T git@github.com`                    |

---

# 💡 Notes / Lessons Learned

* GitHub password authentication is deprecated
* SSH requires a key on the actual machine pushing
* `authorized_keys` is for incoming SSH, not outgoing Git auth
* Do not initialize GitHub repo with README when pushing existing history
* OIDC removes need for AWS_ACCESS_KEY_ID secrets
* Lock IAM trust policy to specific repo + branch
* CI pipelines should use short-lived credentials
* Trust policy mistakes can expose AWS resources

---

# 🔒 Security Takeaways

Static access keys = long-lived risk
OIDC = temporary, scoped credentials
Trust policy `sub` condition = biggest protection boundary

Modern CI/CD should always use federated identity.

---

# 📚 References

* AWS IAM OIDC Documentation
* GitHub Actions OIDC Integration
* Amazon ECR Authentication Guide
* Git SSH Authentication Guide
