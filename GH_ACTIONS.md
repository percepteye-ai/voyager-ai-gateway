Here’s your updated README with clear instructions about **adding the public key via the VM instance metadata**, which avoids overwriting issues and ensures GitHub Actions works reliably:

---

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

```bash
ssh-keygen -t ed25519 -C "admin"
```

- When prompted:

```
Enter file in which to save the key (/Users/nitish/.ssh/id_ed25519):
```

- Enter a **unique filename** to avoid overwriting existing keys, e.g.:

```
/Users/nitish/.ssh/id_ed25519_gcp_vm
```

This creates:

- **Private key**: `~/.ssh/id_ed25519_gcp_vm`
- **Public key**: `~/.ssh/id_ed25519_gcp_vm.pub`

---

## 3. View the Keys

- **Public key** (safe to share, copy this into VM metadata):

```bash
cat ~/.ssh/id_ed25519_gcp_vm.pub
```

- **Private key** (⚠️ keep secret, only paste into GitHub Secrets):

```bash
cat ~/.ssh/id_ed25519_gcp_vm
```

---

## 4. Add Public Key to GCP VM Instance (Persistent Method)

> ⚠️ Do **not** manually edit `authorized_keys`. GCP automatically overwrites it. Use metadata for persistent keys.

1. Go to **Google Cloud Console → Compute Engine → VM Instances**.
2. Click on your VM → **Edit** → **SSH Keys** → **Add item**.
3. Paste the **contents of your public key** (`id_ed25519_gcp_vm.pub`).
4. Save.

✅ This key is now **permanently recognized by the VM**, even after restarts or deployments.

---

## 5. Test SSH Access

From your local machine:

```bash
ssh -i ~/.ssh/id_ed25519_gcp_vm admin@VM_IP
```

- `VM_USER` → 'admin'

- `VM_IP` → external IP of your VM

- If it connects without asking for a password, the key works.

---

## 6. Add Secrets to GitHub

Go to **Repo → Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret Name  | Value                                            |
| ------------ | ------------------------------------------------ |
| `VM_SSH_KEY` | Private key content (`~/.ssh/id_ed25519_gcp_vm`) |
| `VM_USER`    | admin                                            |
| `VM_HOST`    | External IP of your VM                           |
| `ENV_FILE`   | Contents of `.env` file                          |

---

## ✅ Workflow Summary

1. **Trigger** – Runs on push to `release/prod` or manually via workflow dispatch.
2. **SSH Setup** – GitHub Actions connects to VM using the SSH private key stored in secrets.
3. **Deployment** – Fetches latest code, deploys `.env`, rebuilds Docker containers, restarts app.
