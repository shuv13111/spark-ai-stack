# spark-ai-stack

My personal local AI setup running on an NVIDIA DGX Spark. Gemma 4 as the brain, n8n as the automation layer — fully self-hosted, no API keys, no data leaving the box.

---

## What's in here

| Service | What it does |
|---|---|
| **Ollama** | Serves Gemma 4 with GPU acceleration, OpenAI-compatible API on port 11434 |
| **Gemma 4 26B** | Google's open-weight MoE model — the sweet spot for speed + quality on the Spark |
| **n8n** | Visual workflow automation that talks to Gemma 4 for AI-powered automations |

Everything runs in Docker on a shared network. One command to start, survives reboots.

---

## Prerequisites

- NVIDIA DGX Spark (128 GB unified memory, DGX OS / Ubuntu 24.04 arm64)
- Docker + NVIDIA Container Toolkit (pre-installed on DGX OS — just verify)
- ~25 GB free disk space for the 26B model

---

## Quick start

```bash
# Clone the repo
git clone https://github.com/yourusername/spark-ai-stack.git
cd spark-ai-stack

# Start Ollama + n8n
docker compose up -d

# Pull Gemma 4 (26B MoE — ~18 GB download, be patient)
docker exec -it ollama ollama pull gemma4:26b

# Test the model
docker exec -it ollama ollama run gemma4:26b "Say hello!"
```

Then open **http://localhost:5678** in your browser to access n8n.

---

## Connecting n8n to Gemma 4

1. In n8n → **Credentials → Add credential → Ollama**
2. Set Base URL to `http://ollama:11434`
3. Save — it should go green

> ⚠️ Use `http://ollama:11434`, not `localhost`. Inside the n8n container, localhost points to itself.

---

## Useful commands

```bash
# Start everything
docker compose up -d

# Stop everything
docker compose down

# Tail logs
docker compose logs -f n8n
docker compose logs -f ollama

# Pull a different model size
docker exec -it ollama ollama pull gemma4:e4b    # smaller/faster (~9.6 GB)
docker exec -it ollama ollama pull gemma4:31b    # biggest/slowest (~20 GB)

# Update images
docker compose pull && docker compose up -d

# Monitor GPU while a model is running
watch -n1 nvidia-smi
```

---

## Model size guide

| Tag | Size | Best for |
|---|---|---|
| `gemma4:e4b` | ~9.6 GB | Quick automations, low latency |
| `gemma4:12b` | ~7.6 GB | Balanced quality/speed |
| `gemma4:26b` ⭐ | ~18 GB | **Recommended** — MoE, near-31B quality at 4B speed |
| `gemma4:31b` | ~20 GB | Max quality, slower (~7 tok/s on Spark) |

---

## Memory usage (rough estimates)

| Thing | Approx. |
|---|---|
| DGX OS + Docker overhead | ~10 GB |
| n8n | ~1 GB |
| gemma4:26b loaded | ~20–28 GB |
| **Free out of 128 GB** | **~90+ GB** |

Plenty of headroom. Just don't try to run a second large model simultaneously or the Spark will grind to a halt.

---

## Enabling Gemma 4 reasoning mode

For harder tasks (math, logic, complex instructions), add `<|think|>` to the start of your system prompt in n8n to trigger chain-of-thought:

```json
{
  "role": "system",
  "content": "<|think|> Think step by step before answering."
}
```

Leave it out for fast, simple responses.

---

## Repo structure

```
spark-ai-stack/
├── docker-compose.yml        # Ollama + n8n services
└── README.md                 # You're here
```

---

## Notes to future me

- Workflows and credentials are in the `n8n_data` Docker volume — back it up occasionally with `docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine tar czf /backup/n8n-backup.tar.gz /data`
- Models live in the `ollama_models` volume — safe across `docker compose down`
- If something is eating memory and vLLM/another tool OOMs, Ollama is probably holding the model. Stop it with `docker compose stop ollama`
- `OLLAMA_KEEP_ALIVE=24h` in the compose file keeps Gemma loaded so n8n workflows don't have a cold-start delay

---

## License

Do whatever you want. It's just a personal stack.
