# Secure Local Deployment: Ollama + Open WebUI

This guide covers a secure and reliable setup for [Ollama](https://ollama.com) and [Open WebUI](https://github.com/open-webui/open-webui) with Docker.

---

## 1. Preparation

- **Update OS and Docker:**
  - For Arch: `sudo pacman -Syu`
  - For Ubuntu/Debian: `sudo apt update && sudo apt upgrade`
  - Update Docker (if needed): `sudo pacman -Syu docker`

- **Install Ollama:**  
  `curl -fsSL https://ollama.com/install.sh | sh`

- **(Recommended) Create non-root user for Ollama:**
  ```bash
  sudo useradd -r -m -d /var/lib/ollama -s /usr/bin/nologin ollama
  sudo chown -R ollama:ollama /var/lib/ollama
  ```

## 2. Restrict Ollama Network Access

- By default, Ollama listens on `127.0.0.1` (localhost).  
- For Docker access, set Ollama to `0.0.0.0` and restrict with a firewall.

  Example (UFW):
  ```bash
  sudo ufw allow from 127.0.0.1 to any port 11434
  sudo ufw allow from 172.17.0.0/16 to any port 11434
  sudo ufw deny 11434
  ```

## 3. Configure Ollama systemd Service

- Find Ollama’s service file (`/usr/lib/systemd/system/ollama.service` or `/etc/systemd/system/ollama.service`):
  The path of the file is here
  ```bash
  sudo systemctl status ollama
  ```
- Edit the file:  
  Add above `ExecStart`:
  ```
  Environment="OLLAMA_HOST=0.0.0.0"
  ```
- Reload and restart:
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart ollama
  ```
- Verify:
  ```bash
  ss -tuln | grep 11434
  ```

## 4. Test Ollama

- From host:  
  `curl http://localhost:11434`
- From Docker:  
  `docker run --rm curlimages/curl:latest curl http://host.docker.internal:11434/api/version`

## 5. Run Open WebUI

- **Set secret key:**  
  `export WEBUI_SECRET_KEY=$(openssl rand -hex 32)`

- **(Optional) Update Open WebUI:**  
  `docker pull ghcr.io/open-webui/open-webui:latest`

- **Run:**  
  ```bash
  docker run -d -p 3000:8080 \
    --add-host=host.docker.internal:host-gateway \
    -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
    -e WEBUI_SECRET_KEY=$WEBUI_SECRET_KEY \
    -v open-webui:/app/backend/data \
    --name open-webui \
    --restart always \
    ghcr.io/open-webui/open-webui:latest
  ```
- Access: [http://localhost:3000](http://localhost:3000)

## 6. Secure Open WebUI

- Change default admin password after first login.
- Never expose port 3000 to the public internet.
- For remote access, use VPN or SSH tunnel.

## 7. Monitor Logs

- Open WebUI: `docker logs -f open-webui`
- Ollama: `sudo journalctl -u ollama`

## 8. Troubleshooting

- Ensure Ollama listens on `0.0.0.0`
- In Open WebUI, use `http://host.docker.internal:11434` for Ollama
- Check firewall rules
- Test with `curl` from both host and Docker
