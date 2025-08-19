````markdown
# LiteLLM Gateway Deployment on GCP VM

This guide covers deploying the LiteLLM Gateway application (with database) to a Google Cloud VM using Docker Compose.

---

## 1. Prerequisites

- A running GCP VM instance (2 vCPU & 4 GB RAM)
- Docker & Docker Compose installed:
  ```bash
  sudo apt update
  sudo apt install -y docker.io
  sudo apt install -y docker-compose-plugin
  sudo usermod -aG docker $USER
  ```
````

- Generate a self-signed certificate on your VM
  Run this once in your project root:

  ```bash
  mkdir -p certs
  openssl req -x509 -nodes -newkey rsa:2048 \
    -keyout certs/selfsigned.key \
    -out certs/selfsigned.crt \
    -days 365 \
    -subj "/CN=$(curl -s ifconfig.me)"
  ```

---

## 2. Clone the repository on the VM

```bash
git clone git@github.com:percepteye-ai/voyager-ai-gateway.git
cd voyager-ai-gateway
```

---

## 3. Environment variables

Upload `.env` file in the project

```bash
mv .env voyager-ai-gateway/
```

---

## 4. Deployment

```bash
sudo docker compose up -d --build
```

---

## 5. Access the app

Find the **External IP** of your VM from the GCP Console or:

Open in browser:

```
http://<EXTERNAL_IP>:4000
```

---

## 6. Updating & redeploying

When code changes are pushed to GitHub:

```bash
cd voyager-ai-gateway
git pull origin main
sudo docker compose up -d --build
```

---

## 7. Troubleshooting

### Check running containers

```bash
sudo docker compose ps
```

### View logs

```bash
sudo docker compose logs -f litellm
sudo docker compose logs -f db
```

### Restart services

```bash
sudo docker compose down
sudo docker compose up -d --build
```

### Common issues:

- **Port not reachable:** Ensure GCP firewall allows `4000`.
- **Container unhealthy:** Check healthcheck URL in `docker-compose.yml` is correct and app binds to `0.0.0.0`.
- **.env not loading:** Ensure `env_file` path in `docker-compose.yml` is correct and file is in project root.
- **No space left on device (Docker):**

  - Docker can accumulate unused images, containers, volumes, and build cache, leading to disk space issues.
  - To free up space, run:

    ```bash
    sudo docker system prune -af --volumes
    ```

  - This will remove:
    - Unused containers
    - Unused networks
    - Unused images
    - Unused build cache
    - Unused volumes
  - **Warning:** This deletes all unused data. Make sure you do not need any stopped containers, unused images, or volumes before running this command.
