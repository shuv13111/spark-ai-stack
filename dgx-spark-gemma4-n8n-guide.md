# Running Gemma 4 + n8n Locally on NVIDIA DGX Spark — Complete Guide

Everything from a fresh DGX Spark to a working n8n instance that uses Gemma 4 as its AI brain, fully offline-capable after the initial downloads.

**Architecture:** Ollama serves Gemma 4 with GPU acceleration, n8n runs as a Docker container next to it, and they talk over a private Docker network. Both survive reboots.

---

## Step 0 — Verify what's already on the box

DGX OS (Ubuntu 24.04 arm64) ships with the NVIDIA driver stack, Docker, and the NVIDIA Container Toolkit preinstalled. Confirm before touching anything:

```bash
# GPU is visible
nvidia-smi

# Docker is present
docker --version

# NVIDIA Container Toolkit is present
nvidia-ctk --version

# GPU passthrough into containers works
docker run --rm --gpus all nvcr.io/nvidia/cuda:13.0.1-base-ubuntu24.04 nvidia-smi
```

If that last command prints the same GPU table as the bare `nvidia-smi`, your container GPU passthrough is good. If `nvidia-ctk` reports a version older than 1.17.4, update it:

```bash
sudo apt update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

Optional but recommended — run Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## Step 1 — Create the project structure

```bash
mkdir -p ~/ai-stack && cd ~/ai-stack
```

## Step 2 — Write the Docker Compose file

We'll run **both Ollama and n8n in Docker** on a shared network. This keeps networking simple (containers reach each other by service name) and makes the whole stack one command to start/stop.

> **Note:** The Spark ships with Ollama preinstalled natively. If you prefer to use that instead of the containerized one, see the "Alternative: native Ollama" section at the end. Don't run both at once — they'll fight over port 11434 and memory.

Create `~/ai-stack/docker-compose.yml`:

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
      - OLLAMA_KEEP_ALIVE=24h   # keep the model loaded in memory
      - OLLAMA_NUM_PARALLEL=2   # concurrent requests; raise if needed
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
      - N8N_SECURE_COOKIE=false     # allows login over plain http on your LAN
      - WEBHOOK_URL=http://<SPARK_IP>:5678/   # replace with your Spark's LAN IP
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

Replace `<SPARK_IP>` with your Spark's LAN IP (find it with `hostname -I`).

## Step 3 — Start the stack

```bash
cd ~/ai-stack
docker compose up -d
docker compose ps    # both containers should show "running"
```

## Step 4 — Pull Gemma 4

Gemma 4 comes in five sizes on Ollama. On the Spark, the smart pick is the **26B A4B (Mixture-of-Experts)** — near-31B quality but only ~4B active parameters per token, so it's dramatically faster on the Spark's memory bandwidth. The 31B dense model fits in memory fine but decodes at only ~7 tok/s on this hardware, which feels sluggish for interactive workflows.

```bash
# Recommended: 26B MoE (~18 GB download)
docker exec -it ollama ollama pull gemma4:26b

# Optional smaller/faster fallback for quick workflows (~9.6 GB)
docker exec -it ollama ollama pull gemma4:e4b
```

Other available tags if you want them: `gemma4:e2b`, `gemma4:12b`, `gemma4:31b`, plus QAT variants (e.g. `gemma4:26b-qat`, ~16 GB) that trade a little quality for less memory.

Quick sanity test from the CLI:

```bash
docker exec -it ollama ollama run gemma4:26b "Say hello in five words."
```

## Step 5 — Test the API

n8n will talk to Ollama over HTTP. Verify the endpoint works from the host:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4:26b",
  "messages": [{"role": "user", "content": "Reply with the single word: ready"}],
  "stream": false
}'
```

You should get a JSON response with `"content": "ready"` (or similar). While it generates, watch GPU usage in another terminal with `watch -n1 nvidia-smi` — you should see memory and utilization climb.

**Optional — enable Gemma 4's thinking mode** for harder reasoning tasks by adding a system message starting with the `<|think|>` token:

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

Leave the token out for fast, simple responses; include it for math, complex logic, or document analysis.

## Step 6 — Set up n8n

1. Open `http://<SPARK_IP>:5678` in a browser (or `http://localhost:5678` if you're on the Spark's desktop).
2. Create the owner account when prompted (stored locally, nothing leaves the box).

### Connect n8n to Gemma 4

1. In n8n, go to **Credentials → Add credential → Ollama**.
2. Set **Base URL** to:
   ```
   http://ollama:11434
   ```
   This works because both containers share the `ai-net` network — `ollama` resolves to the container. **Do not use `localhost`** here; inside the n8n container, localhost is n8n itself.
3. Save. n8n will test the connection and should show green.

### Build a first AI workflow

1. **New workflow** → add a **Chat Trigger** node (gives you a built-in chat UI for testing).
2. Add an **AI Agent** node (or **Basic LLM Chain** for simple cases) and connect the trigger to it.
3. Under the agent's **Chat Model**, select **Ollama Chat Model**, pick your Ollama credential, and choose `gemma4:26b` from the model dropdown.
4. Click **Open chat** (bottom panel) and send a message. The reply is generated entirely on your Spark.

From here, anything in n8n's node library can feed the model: webhooks, schedules, IMAP email, RSS, HTTP requests, databases. A classic starter is *Email summarizer*: **IMAP Email Trigger → AI Agent (summarize, classify urgency) → Slack/Telegram notification**.

### Calling Gemma 4 from a raw HTTP Request node

If you ever want to skip the LangChain-style nodes and hit Ollama directly (e.g., for structured JSON output), use an **HTTP Request** node:

- **Method:** POST
- **URL:** `http://ollama:11434/api/chat`
- **Body (JSON):**

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

The `"format": "json"` flag makes Ollama constrain output to valid JSON — very useful for feeding downstream nodes.

## Step 7 — Make it survive reboots

Both services use `restart: unless-stopped`, so Docker brings them back automatically as long as the Docker daemon starts at boot (it does by default on DGX OS). Verify:

```bash
sudo systemctl is-enabled docker   # should print "enabled"
```

Day-to-day operations:

```bash
docker compose down              # stop everything
docker compose up -d             # start everything
docker compose logs -f n8n       # tail n8n logs
docker compose logs -f ollama    # tail ollama logs
docker compose pull && docker compose up -d   # update images
```

n8n workflows/credentials live in the `n8n_data` volume and models in `ollama_models`, so updates and restarts don't lose anything.

---

## Memory budgeting cheat sheet

Everything shares one 128 GB unified pool — weights, KV cache, n8n, OS, Docker.

| Component | Approx. usage |
|---|---|
| DGX OS + desktop + Docker | ~8–12 GB |
| n8n container | ~0.5–1 GB |
| gemma4:26b loaded + KV cache | ~20–30 GB (grows with context length) |
| **Headroom remaining** | **~85+ GB** |

You have plenty of room. Just avoid loading a second large model (or a big image-gen model) simultaneously unless you've done the math — once total usage approaches the full 128 GB, the Spark grinds to a halt rather than failing gracefully.

## Troubleshooting

**n8n can't reach Ollama ("connection refused")** — You used `localhost:11434` in the credential. Use `http://ollama:11434` (compose setup) or `http://<SPARK_IP>:11434` (native Ollama).

**`ollama pull` is slow or fails** — It's a ~18 GB download; check disk space with `df -h` and just retry — pulls resume.

**Responses feel slow** — First request after idle includes model load time (tens of seconds for 26B). `OLLAMA_KEEP_ALIVE=24h` prevents reloading. If steady-state generation is still slow, you may be on `gemma4:31b` dense — switch to `26b` (MoE) or `e4b`.

**Port 11434 already in use when starting compose** — The preinstalled native Ollama service is running. Disable it: `sudo systemctl disable --now ollama`, then `docker compose up -d` again.

**vLLM/other GPU workloads OOM later** — Ollama is holding the model in the shared pool. Stop it first or lower `OLLAMA_KEEP_ALIVE`.

---

## Alternative: use the preinstalled native Ollama instead of the container

If you'd rather keep Ollama on bare metal (slightly less overhead, NVIDIA's playbooks use it this way):

1. Remove the `ollama` service block from the compose file (keep n8n).
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

3. In the n8n compose service, add:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

4. In the n8n Ollama credential, use `http://host.docker.internal:11434` as the Base URL.

Everything else in the guide is identical.
