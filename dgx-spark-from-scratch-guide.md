# DGX Spark From Scratch: Gemma 4 + n8n — The Complete Zero-to-Running Guide

Everything from unboxing the Spark to a working n8n automation powered by a local Gemma 4 — including OS setup, dev tools (git, VS Code), and every file you need to create. No steps assumed.

---

# Phase 0 — Unboxing & First Boot

1. **Connect peripherals BEFORE power.** Plug in monitor (HDMI/DP via USB-C), keyboard, mouse, and Ethernet (recommended for the big model downloads ahead). The Spark has **no power button** — it boots the moment you connect power, so plug the power cable in last.
2. **Complete the DGX OS first-boot wizard.** DGX OS is Ubuntu 24.04 (arm64) with NVIDIA's full driver/CUDA/Docker stack preinstalled. Create your user account — remember it; **only this first account can install system updates**.
3. **Connect to your network.** Ethernet just works; for Wi-Fi use the Ubuntu network settings in the top-right corner.
4. **Find and note your Spark's IP** (you'll need it later for n8n):

```bash
hostname -I
# e.g. 192.168.1.50 — write this down. Referred to as <SPARK_IP> below.
```

> **Optional — headless use:** if you'd rather drive the Spark from your laptop, enable SSH (`sudo systemctl enable --now ssh`) and connect with `ssh youruser@<SPARK_IP>`. NVIDIA also offers the **NVIDIA Sync** app, which auto-discovers Sparks on your network. Everything in this guide works identically over SSH.

# Phase 1 — Update the System

```bash
# Update all OS packages
sudo apt update && sudo apt full-upgrade -y

# Reboot if the kernel or driver was updated
sudo reboot
```

Also open the **DGX Dashboard** (app grid, or `http://localhost:11000` on the Spark) and install any pending NVIDIA platform updates. NVIDIA ships major DGX OS updates twice a year plus security patches in between.

# Phase 2 — Verify the GPU + Docker Stack

DGX OS comes with Docker and the NVIDIA Container Toolkit preinstalled. Verify rather than reinstall:

```bash
# 1. GPU is visible to the OS
nvidia-smi

# 2. Docker is installed
docker --version

# 3. NVIDIA Container Toolkit is installed
nvidia-ctk --version

# 4. THE key test: GPU passthrough into a container
sudo docker run --rm --gpus all nvcr.io/nvidia/cuda:13.0.1-base-ubuntu24.04 nvidia-smi
```

If step 4 prints the same GPU table as step 1, containers can use the GPU. 

**If** `nvidia-ctk` is older than 1.17.4, update it:

```bash
sudo apt update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

**Run Docker without sudo** (recommended — the rest of this guide assumes it):

```bash
sudo usermod -aG docker $USER
newgrp docker

# verify it works without sudo now
docker ps
```

# Phase 3 — Install Dev Tools

## 3.1 Git

```bash
sudo apt install -y git curl wget

# Identify yourself to git
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

## 3.2 VS Code

The Spark is **arm64**, so you need the arm64 build — the regular x86 .deb will refuse to install.

**Option A — VS Code directly on the Spark** (you're using its desktop):

```bash
cd ~/Downloads
wget -O code_arm64.deb "https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-arm64"
sudo apt install -y ./code_arm64.deb

# Launch it
code
```

Useful extensions to install (Ctrl+Shift+X inside VS Code):
- **Docker** (ms-azuretools.vscode-docker) — manage your containers visually
- **YAML** (redhat.vscode-yaml) — linting for the compose file you're about to write

**Option B — VS Code on your laptop, editing the Spark remotely** (best of both worlds):

1. On your laptop's VS Code, install the **Remote - SSH** extension (ms-vscode-remote.remote-ssh).
2. `Ctrl+Shift+P` → "Remote-SSH: Connect to Host" → `youruser@<SPARK_IP>`.
3. VS Code installs its server component on the Spark automatically and you edit files there as if local. Install the Docker extension *on the remote* too.

Either way, you now have a proper editor for the files below.

# Phase 4 — Create the Project

```bash
mkdir -p ~/spark-ai-stack && cd ~/spark-ai-stack
git init
code .   # opens the folder in VS Code (skip if using nano/vim)
```

## 4.1 Create `docker-compose.yml`

In VS Code, create a new file named `docker-compose.yml` in the project root and paste this in (or use the heredoc below if you're terminal-only):

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"          # exposed to host so you can curl/test it
    volumes:
      - ollama_models:/root/.ollama
    environment:
      - OLLAMA_KEEP_ALIVE=24h   # keep the model warm in memory (no cold starts)
      - OLLAMA_NUM_PARALLEL=2   # concurrent requests
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - ai-net

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest   # official multi-arch image, arm64 OK
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - GENERIC_TIMEZONE=America/New_York
      - TZ=America/New_York
      - N8N_SECURE_COOKIE=false              # allow login over plain http on your LAN
      - WEBHOOK_URL=http://<SPARK_IP>:5678/  # ← replace with your actual Spark IP
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - ai-net
    depends_on:
      - ollama

volumes:
  ollama_models:
  n8n_data:

networks:
  ai-net:
    driver: bridge
```

**Replace `<SPARK_IP>`** with the IP you noted in Phase 0 (e.g. `http://192.168.1.50:5678/`).

Terminal-only alternative (creates the same file):

```bash
cat > ~/spark-ai-stack/docker-compose.yml << 'EOF'
# ... (paste the YAML above) ...
EOF
```

## 4.2 Create `.gitignore`

```bash
cat > ~/spark-ai-stack/.gitignore << 'EOF'
.env
*.log
EOF
```

## 4.3 First commit

```bash
cd ~/spark-ai-stack
git add .
git commit -m "initial: ollama + n8n compose stack for dgx spark"
```

# Phase 5 — Start the Stack

```bash
cd ~/spark-ai-stack
docker compose up -d

# Both should show "running"
docker compose ps
```

First run downloads the Ollama (~1 GB) and n8n (~400 MB) images. Watch progress with `docker compose logs -f` if curious (Ctrl+C exits logs without stopping containers).

# Phase 6 — Download Gemma 4

Gemma 4 comes in five sizes. On the Spark, the smart pick is **26B A4B (Mixture-of-Experts)** — near-31B quality, but only ~4B parameters active per token, so it's dramatically faster on the Spark's memory bandwidth. The 31B dense model fits fine but decodes at roughly 7 tok/s on this hardware, which feels sluggish in interactive workflows.

```bash
# Recommended: 26B MoE (~18 GB download — grab a coffee)
docker exec -it ollama ollama pull gemma4:26b

# Optional smaller/faster model for quick workflows (~9.6 GB)
docker exec -it ollama ollama pull gemma4:e4b
```

Other tags if you want them: `gemma4:e2b`, `gemma4:12b`, `gemma4:31b`, plus QAT variants (`gemma4:26b-qat`, ~16 GB) that trade a sliver of quality for less memory.

Sanity-check from the CLI:

```bash
docker exec -it ollama ollama run gemma4:26b "Say hello in five words."
```

First response includes model load time (tens of seconds); after that it stays warm thanks to `OLLAMA_KEEP_ALIVE`.

# Phase 7 — Test the API

n8n will talk to Ollama over HTTP, so verify the endpoint:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4:26b",
  "messages": [{"role": "user", "content": "Reply with the single word: ready"}],
  "stream": false
}'
```

Expect a JSON response containing `"content": "ready"`. While it generates, watch the GPU in a second terminal:

```bash
watch -n1 nvidia-smi
```

**Optional — Gemma 4 thinking mode** for harder reasoning (math, complex logic). Add a system message starting with the `<|think|>` token:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4:26b",
  "messages": [
    {"role": "system", "content": "<|think|> Think step by step before answering."},
    {"role": "user", "content": "What is 17 * 23?"}
  ],
  "stream": false
}'
```

Leave the token out for fast simple responses; include it when quality matters more than speed.

# Phase 8 — Set Up n8n (Full Walkthrough)

## 8.1 Onboarding

1. Open **`http://<SPARK_IP>:5678`** from any machine on your network (or `http://localhost:5678` on the Spark itself).
2. n8n shows its **owner account setup** screen on first visit. Enter an email, name, and password. This account is stored locally in the `n8n_data` Docker volume — nothing is sent anywhere.
3. Skip or answer the short onboarding questionnaire — your choice.

## 8.2 Connect n8n to Gemma 4

1. Left sidebar → **Credentials** → **Add credential**.
2. Search for **Ollama** and select it.
3. Set **Base URL** to exactly:
   ```
   http://ollama:11434
   ```
   This works because both containers share the `ai-net` Docker network, so the hostname `ollama` resolves to the Ollama container.
   > ⚠️ **Do not use `localhost:11434` here.** Inside the n8n container, localhost means the n8n container itself — the connection will be refused.
4. Click **Save**. n8n tests the connection and should show green.

## 8.3 Build your first AI workflow

1. **Workflows → Add workflow.**
2. Click **+** and add a **Chat Trigger** node — this gives you a built-in chat panel for testing.
3. Add an **AI Agent** node and connect the Chat Trigger's output to it.
4. On the AI Agent node, click the **Chat Model** connector beneath it → choose **Ollama Chat Model** → select your Ollama credential → pick **gemma4:26b** from the model dropdown.
5. (Optional) Open the Ollama Chat Model node's options and set the **System Message**. Prefix it with `<|think|>` if you want reasoning mode.
6. Click **Open chat** at the bottom and send a message. The reply is generated entirely on your Spark — watch `nvidia-smi` spike.
7. **Save** the workflow (Ctrl+S) and toggle it **Active** if it uses a trigger that should run unattended.

## 8.4 Calling Gemma 4 from a raw HTTP Request node

For structured outputs (classification, extraction) it's often cleaner to skip the agent abstraction and hit Ollama directly. Add an **HTTP Request** node:

- **Method:** `POST`
- **URL:** `http://ollama:11434/api/chat`
- **Body Content Type:** JSON
- **Body:**

```json
{
  "model": "gemma4:26b",
  "messages": [
    {
      "role": "system",
      "content": "You are a classifier. Respond ONLY with valid JSON: {\"category\": string, \"urgency\": \"low\"|\"medium\"|\"high\"}"
    },
    {
      "role": "user",
      "content": "{{ $json.emailBody }}"
    }
  ],
  "format": "json",
  "stream": false
}
```

The `"format": "json"` flag makes Ollama constrain output to valid JSON — ideal for feeding downstream nodes.

**Starter workflow idea:** *Email triage* — **IMAP Email Trigger → HTTP Request (the classifier above) → IF node on urgency → Slack/Telegram notification for "high"**. Runs 24/7 on your desk, zero API cost.

# Phase 9 — Persistence & Day-to-Day Operations

Both services use `restart: unless-stopped`, and Docker starts at boot on DGX OS, so the stack survives reboots automatically. Verify once:

```bash
sudo systemctl is-enabled docker   # should print "enabled"
```

Daily driver commands:

```bash
cd ~/spark-ai-stack
docker compose down                            # stop everything
docker compose up -d                           # start everything
docker compose logs -f n8n                     # tail n8n logs
docker compose logs -f ollama                  # tail ollama logs
docker compose pull && docker compose up -d    # update to latest images
```

Backups — your workflows/credentials live in the `n8n_data` volume:

```bash
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/n8n-backup-$(date +%F).tar.gz /data
```

---

# Memory Budgeting Cheat Sheet

Everything shares one 128 GB unified pool — weights, KV cache, n8n, OS, Docker.

| Component | Approx. usage |
|---|---|
| DGX OS + desktop + Docker | ~8–12 GB |
| n8n container | ~0.5–1 GB |
| gemma4:26b loaded + KV cache | ~20–30 GB (grows with context length) |
| **Headroom remaining** | **~85+ GB** |

Plenty of room — just don't load a second large model (or a big image-gen model) simultaneously without doing the math. When total usage approaches 128 GB the Spark grinds to a halt rather than failing gracefully.

# Troubleshooting

**n8n credential test fails / "connection refused"** — You used `localhost:11434` as the Base URL. Use `http://ollama:11434`.

**Port 11434 already in use when starting compose** — The Spark ships with a *native* Ollama preinstalled and its service may be running. Disable it so the containerized one owns the port: `sudo systemctl disable --now ollama`, then `docker compose up -d` again. (Or use the native one instead — see the appendix.)

**`ollama pull` slow or fails midway** — It's an 18 GB download. Check disk space (`df -h`) and retry; pulls resume where they left off.

**Responses feel slow** — The first request after idle includes model load (tens of seconds). `OLLAMA_KEEP_ALIVE=24h` prevents repeats. If steady-state generation is still slow, you may be running `gemma4:31b` dense — switch to `26b` (MoE) or `e4b`.

**VS Code .deb won't install** — You grabbed the x86_64 build. Re-download with the arm64 URL from Phase 3.2.

**Another GPU tool (vLLM, ComfyUI) OOMs** — Ollama is holding the model in the shared pool. `docker compose stop ollama` or lower `OLLAMA_KEEP_ALIVE`.

# Appendix — Using the Preinstalled Native Ollama Instead

If you'd rather run Ollama on bare metal (slightly less overhead; NVIDIA's own playbooks use it this way):

1. Delete the `ollama` service block from `docker-compose.yml` (keep n8n, volumes, network).
2. Make native Ollama listen on all interfaces so the n8n container can reach it:

```bash
sudo systemctl edit ollama
```

Add:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_KEEP_ALIVE=24h"
```

```bash
sudo systemctl restart ollama
ollama pull gemma4:26b
```

3. In the n8n service in compose, add:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

4. In the n8n Ollama credential, use `http://host.docker.internal:11434` as the Base URL.

Everything else in the guide is identical.
