# 🚀 CI/CD Setup for GCP VM with Docker Compose via GitHub Actions

This guide explains how to set up **continuous deployment** for a Docker Compose application running on a **GCP VM**, using **GitHub Actions**. The workflow builds and deploys the application directly on the VM whenever code is pushed to the `release/prod` branch.

---

## 1. Prerequisites

1. A **GCP VM** instance running Linux with Docker and Docker Compose installed.
2. Your application is already set up on the VM using Docker Compose.
3. GitHub repository with your application code.
4. Access to create **GitHub Secrets** in the repository.

---

## 2. Generate SSH Keys

On your **local machine**, generate an SSH keypair for GitHub Actions:

```bash
ssh-keygen -t ed25519 -C "github-cicd"
```

- When prompted:

```
Enter file in which to save the key (/Users/nitish/.ssh/id_ed25519):
```

- Enter a unique path to avoid overwriting existing keys, e.g.:

```
/Users/nitish/.ssh/id_ed25519_gcp_vm
```

This creates:

- Private key: `~/.ssh/id_ed25519_gcp_vm`
- Public key: `~/.ssh/id_ed25519_gcp_vm.pub`

---

## 3. Copy Public Key to GCP VM

### Option 1: Using `ssh-copy-id`

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_gcp_vm.pub VM_USER@VM_IP
```

### Option 2: Manual method

1. Print the public key:

```bash
cat ~/.ssh/id_ed25519_gcp_vm.pub
```

2. SSH into your VM using password:

```bash
gcloud compute ssh --zone "us-east1-d" "gateway" --project "stellar-smoke-468717-t0"
```

3. Create `.ssh` directory (if it doesn’t exist):

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

4. Open `authorized_keys` in nano:

```bash
nano ~/.ssh/authorized_keys
```

5. Paste the public key, then save in nano:

   - Press `CTRL + O` → Enter → `CTRL + X` to exit.

6. Set proper permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

---

## 4. Test SSH Connection

Verify that you can connect without a password:

```bash
ssh -i ~/.ssh/id_ed25519_gcp_vm VM_USER@VM_IP
```

- `VM_USER` → your VM username (check with `whoami` on VM)
- `VM_IP` → external IP of your VM (get with `gcloud compute instances describe gateway --zone "us-east1-d" --project "stellar-smoke-468717-t0" --format='get(networkInterfaces[0].accessConfigs[0].natIP)'`)

---

## 5. Add Secrets to GitHub

Go to **Repo → Settings → Secrets and variables → Actions → New repository secret**:

| Secret Name  | Value                                            |
| ------------ | ------------------------------------------------ |
| `VM_SSH_KEY` | Private key content (`~/.ssh/id_ed25519_gcp_vm`) |
| `VM_USER`    | Your VM username                                 |
| `VM_HOST`    | External IP of your VM                           |
| `ENV_FILE`   | Contents of .env file                            |

---

## 6. Create GitHub Actions Workflow

Create `.github/workflows/deploy.yml` in your repository:

```yaml
name: Deploy to GCP VM

on:
  push:
    branches: ["release/prod"]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.VM_SSH_KEY }}

      - name: Deploy on VM
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.VM_USER }}@${{ secrets.VM_HOST }} << 'EOF'
            set -e
            cd ~/your-app-folder   # <-- replace with your actual app path
            git fetch --all
            git reset --hard origin/release/prod
            docker compose -f docker-compose.prod.yml down
            docker compose -f docker-compose.prod.yml build --no-cache
            docker compose -f docker-compose.prod.yml up -d
          EOF
```

---

### ✅ Workflow Explanation

1. **Trigger**

   - Runs only when code is pushed to the `release/prod` branch.

2. **SSH Setup**

   - Uses the SSH private key stored in GitHub Secrets to connect to the VM.

3. **Deploy Steps**

   - Navigate to your application folder.
   - Pull latest code from GitHub.
   - Stop current containers (`docker compose down`).
   - Rebuild fresh Docker images using `docker-compose.prod.yml`.
   - Start containers in detached mode (`docker compose up -d`).
