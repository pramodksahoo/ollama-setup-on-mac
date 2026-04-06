#  Running LLMs Locally on macOS (Apple Silicon) —  Setup Guide

**Tested on:** MacBook Pro (M3 / M4) · macOS Sequoia · Ollama 0.20.x · Docker Desktop 4.x
**Last Updated:** April 2026

---

## 📌 Overview

This guide explains how to run Large Language Models (LLMs) locally on macOS using:

* **Ollama (native)** → GPU acceleration via Apple Metal
* **Docker services** → UI and API layers (Open WebUI + LiteLLM)

### Architecture

```
macOS Host
├── Ollama (native, Metal GPU) → http://localhost:11434
│
└── Docker
    ├── Open WebUI  → http://localhost:3000
    └── LiteLLM     → http://localhost:4000
```

---

## ⚠️ Key Principles

* ✅ Ollama must run **natively on macOS (NOT inside Docker)**
* ✅ Docker connects via:

  ```
  http://host.docker.internal:11434
  ```
* ✅ Ollama must bind to:

  ```
  0.0.0.0:11434
  ```
* ❌ Do NOT run Ollama inside Docker (no GPU → slow performance)

---

## 🧰 Prerequisites

* macOS Ventura (13+) or later
* Homebrew installed
* Docker Desktop (Apple Silicon)
* Minimum 8 GB RAM (16 GB+ recommended)

---

## 🚀 Step 1 — Install Ollama

```bash
brew install ollama
```

Verify:

```bash
which ollama
ollama --version
```

---

## ⚠️ Important: Determine Correct Ollama Path

```bash
which ollama
realpath $(which ollama)
```

### Examples:

| System         | Path                       |
| -------------- | -------------------------- |
| Apple Silicon  | `/opt/homebrew/bin/ollama` |
| Custom / Intel | `/usr/local/bin/ollama`    |

👉 Always use `which ollama` output
👉 Never use `/Cellar/...` path

---

## ⚙️ Step 2 — Configure Ollama Service

### Stop Existing Services

```bash
brew services stop ollama
pkill ollama
lsof -i :11434
```

---

### Remove Old Config

```bash
rm -f ~/Library/LaunchAgents/com.ollama.serve.plist
rm -f ~/Library/LaunchAgents/homebrew.mxcl.ollama.plist
```

---

### Create launchd Service

Replace `<OLLAMA_PATH>`:

```bash
cat > ~/Library/LaunchAgents/com.ollama.serve.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.ollama.serve</string>

    <key>ProgramArguments</key>
    <array>
        <string><OLLAMA_PATH></string>
        <string>serve</string>
    </array>

    <key>EnvironmentVariables</key>
    <dict>
        <key>OLLAMA_HOST</key>
        <string>0.0.0.0</string>
        <key>OLLAMA_KEEP_ALIVE</key>
        <string>-1</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/ollama.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/ollama.err</string>
</dict>
</plist>
EOF
```

Replace path:

```bash
sed -i '' 's|<OLLAMA_PATH>|/usr/local/bin/ollama|g' \
~/Library/LaunchAgents/com.ollama.serve.plist
```

---

### Start Service

```bash
launchctl load ~/Library/LaunchAgents/com.ollama.serve.plist
sleep 3
```

---

### Verify

```bash
lsof -i :11434
```

✅ Correct:

```
*:11434
```

---

### Test API

```bash
curl http://localhost:11434/api/tags
```

---

## 📦 Step 3 — Pull Models

```bash
ollama pull gemma3:4b
ollama pull gemma4:latest
ollama pull qwen2.5-coder:14b
ollama pull deepseek-r1:14b
```

Check:

```bash
ollama list
```

---

## 🧠 Model Recommendations

| RAM    | Models            |
| ------ | ----------------- |
| 8 GB   | gemma3:4b         |
| 16 GB  | gemma4            |
| 24 GB+ | qwen2.5-coder:14b |
| 36 GB+ | llama3            |

---

## 🐳 Step 4 — Docker Setup

```bash
mkdir ~/ollama-stack && cd ~/ollama-stack
```

---

### docker-compose.yml

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
    volumes:
      - open-webui:/app/backend/data
    restart: unless-stopped

  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    ports:
      - "4000:4000"
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    restart: unless-stopped

volumes:
  open-webui:
```

---

### litellm_config.yaml

```yaml
model_list:
  - model_name: gemma3
    litellm_params:
      model: ollama/gemma3:4b
      api_base: http://host.docker.internal:11434
```

---

### Start Docker

```bash
docker compose up -d
docker compose logs -f
```

---

## ✅ Step 5 — Verification

```bash
curl http://localhost:11434/api/tags
curl http://localhost:4000/health
curl http://localhost:4000/v1/models
```

Open UI:

```
http://localhost:3000
```

---

## 🛠️ Troubleshooting

| Issue         | Cause             | Fix                        |
| ------------- | ----------------- | -------------------------- |
| Not reachable | localhost binding | Use `0.0.0.0`              |
| Wrong path    | hardcoded path    | use `which ollama`         |
| Docker error  | host config issue | use `host.docker.internal` |
| Slow          | running in Docker | run native                 |

---

## 🔧 Useful Commands

```bash
ollama list
ollama run gemma3:4b
ollama ps
ollama rm <model>

cat /tmp/ollama.err

docker compose logs
docker compose restart
```

---

## ✅ Summary

* Run Ollama natively
* Use correct binary path dynamically
* Bind to `0.0.0.0`
* Connect Docker via `host.docker.internal`

---
