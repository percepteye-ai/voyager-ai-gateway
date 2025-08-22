# 🚀 CI/CD Setup for GCP VM with Docker Compose via GitHub Actions

This guide explains how to set up **continuous deployment** for a Docker Compose application running on a **GCP VM**, using **GitHub Actions**. The workflow builds and deploys the application directly on the VM whenever code is pushed to the `release/prod` branch.

---

## 1. Prerequisites

1. A **GCP VM** instance running Linux with Docker and Docker Compose installed.
2. Your application is already set up on the VM using Docker Compose.
3. GitHub repository with your application code.
4. Access to create **GitHub Secrets** in the repository.

---

## 2. Generate SSH Keys (on your local machine)

Run:

```bash
ssh-keygen -t ed25519 -C "github-cicd"
```

When prompted:

```
Enter file in which to save the key (/Users/nitish/.ssh/id_ed25519):
```

➡️ Enter a unique filename to avoid overwriting existing keys, e.g.:

```
/Users/nitish/.ssh/id_ed25519_gcp_vm
```

This creates two files:

- **Private key**: `~/.ssh/id_ed25519_gcp_vm`
- **Public key**: `~/.ssh/id_ed25519_gcp_vm.pub`

---

## 3. View the Keys

- View **public key** (safe to share, copy this into VM):

  ```bash
  cat ~/.ssh/id_ed25519_gcp_vm.pub
  ```

- View **private key** (⚠️ keep secret, only paste into GitHub Secrets):

  ```bash
  cat ~/.ssh/id_ed25519_gcp_vm
  ```

---

## 4. Add Public Key to VM

### Option 1: Using `ssh-copy-id` (easiest)

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_gcp_vm.pub VM_USER@VM_IP
```

### Option 2: Manual method

1. Print the public key:

   ```bash
   cat ~/.ssh/id_ed25519_gcp_vm.pub
   ```

2. SSH into your VM:

   ```bash
   gcloud compute ssh VM_NAME --zone "ZONE" --project "PROJECT_ID"
   ```

3. Create `.ssh` folder (if not exists):

   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   ```

4. Add the public key to `authorized_keys`:

   ```bash
   nano ~/.ssh/authorized_keys
   ```

   Paste the key, save (`CTRL+O`, Enter, `CTRL+X`).

5. Set proper permissions:

   ```bash
   chmod 600 ~/.ssh/authorized_keys
   ```

---

## 5. Test SSH Access

From your local machine, run:

```bash
ssh -i ~/.ssh/id_ed25519_gcp_vm VM_USER@VM_IP
```

✅ You should log in **without being asked for a password**.

- `VM_USER` = your VM Linux username (check with `whoami` on VM).
- `VM_IP` = external IP of VM (`gcloud compute instances describe ... --format='get(networkInterfaces[0].accessConfigs[0].natIP)'`).

---

## 6. Add Secrets to GitHub

Go to **Repo → Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret Name  | Value (example)                                            |
| ------------ | ---------------------------------------------------------- |
| `VM_SSH_KEY` | Paste contents of `~/.ssh/id_ed25519_gcp_vm` (private key) |
| `VM_USER`    | VM username (e.g. `nitish`)                                |
| `VM_HOST`    | VM external IP (e.g. `34.xxx.xxx.xxx`)                     |

---

## 7. GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

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

## ✅ Workflow Summary

1. **Trigger** – Runs when code is pushed to `release/prod`.
2. **SSH Setup** – GitHub Actions connects to VM using your SSH private key stored in secrets.
3. **Deployment** – Fetches latest code, rebuilds Docker containers, restarts app.
