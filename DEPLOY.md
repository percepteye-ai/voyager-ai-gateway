# Voyager AI Gateway Deployment on GCP VM

This guide explains how to deploy the Voyager AI Gateway (with database and Nginx reverse proxy) to a Google Cloud VM using Docker Compose.  
You will be able to access the service over **HTTP and HTTPS** (self-signed certificates).

---

## 1. Prerequisites

- A running **GCP VM** (at least 2 vCPU & 4 GB RAM).
- Install Docker & Docker Compose:

  ```bash
  # Update
  sudo apt-get update
  sudo apt-get install -y ca-certificates curl gnupg lsb-release

  # Setup Docker GPG key
  sudo install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  sudo chmod a+r /etc/apt/keyrings/docker.gpg

  # Add Docker repo
  echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

  # Install Docker + Compose
  sudo apt-get update
  sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

  # Verify
  docker --version
  docker compose version

  sudo usermod -aG docker $USER
  ```

  > ⚠️ Logout/login again to apply the docker group changes.

- **Firewall rules**: Allow HTTP (80) and HTTPS (443) traffic (Alternatively, this can be enabled via an option when creating the instance):

  ```bash
  gcloud compute firewall-rules create allow-http-https \
    --allow tcp:80,tcp:443 --source-ranges=0.0.0.0/0 \
    --target-tags=voyager_ai_gw
  ```

- **Self-signed TLS certificate** (for HTTPS without a domain):  
  Run this in your project root on the VM:

  ```bash
  mkdir -p certs
  openssl req -x509 -nodes -newkey rsa:2048 \
    -keyout certs/selfsigned.key \
    -out certs/selfsigned.crt \
    -days 365 \
    -subj "/CN=$(curl -s ifconfig.me)"
  ```

  > Note: Browsers and clients will warn that this certificate is **not trusted**. This is expected when using self-signed certs on an IP address. For production, use a real domain + Let’s Encrypt.

---

## 2. Clone the repository on the VM

```bash
git clone -b release/prod git@github.com:percepteye-ai/voyager-ai-gateway.git
cd voyager-ai-gateway
```

---

## 3. Environment variables

Upload your `.env` file into the project:

```bash
mv .env voyager-ai-gateway/
```

---

## 4. Deployment

Build and start all services (Voyager AI Gateway app, db, nginx):

```bash
sudo docker compose -f docker-compose.prod.yml up -d --build

docker compose logs -f
```

---

## 5. Access the app

- From your browser:

  ```
  https://<EXTERNAL_IP>/
  ```

  You will see a **“Not Secure / Unsafe” warning** because of the self-signed certificate. Accept and proceed.

- From curl (bypassing SSL verification):

  ```bash
  curl --insecure --location 'https://<EXTERNAL_IP>/chat/completions' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer <API_KEY>' \
  --data '{
    "model": "gpt-5",
    "messages": [
      {"role": "user", "content": "what llm are you"}
    ]
  }'
  ```

---

## 6. Updating & redeploying

When new code is pushed:

```bash
cd voyager-ai-gateway
git pull origin release/prod
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
sudo docker compose logs -f voyager_ai_gw
sudo docker compose logs -f db
sudo docker compose logs -f nginx_proxy
```

### Restart services

```bash
sudo docker compose down
sudo docker compose up -d --build
```

### Common issues

- **Browser shows “unsafe”** → expected with self-signed certificates on IPs. Use `--insecure` in curl or accept in browser.
- **Port not reachable** → ensure GCP firewall allows `80,443`.
- **Container unhealthy** → check that the app binds to `0.0.0.0` and the healthcheck path in `docker-compose.yml` is valid.
- **.env not loading** → ensure it’s in the project root and referenced in `docker-compose.yml`.
- **Disk space issues** → prune Docker resources:
  ```bash
  sudo docker system prune -af --volumes
  ```

---

## 8. Production Notes

- For production, register a **domain name** and update your Nginx config with your domain.
- Replace the self-signed certs with **Let’s Encrypt** (automatic free SSL).
- This removes security warnings and ensures full HTTPS trust.
