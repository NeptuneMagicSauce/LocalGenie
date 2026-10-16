![Local Genie](../images/local.genie.png)

# setting up a local agent

## 1. dedicated WSL instance

```bash
wsl.exe --install -d Debian --name AgentHost
wsl.exe -d AgentHost
```

Remove its auto-mounts, make sure it has systemd in `/etc/wsl.conf`:

```init
[boot]
systemd=true

[automount]
root = /
options = "metadata"
mountFsTab = false
```

```bash
sudo apt update
```

## 2. docker

```bash
sudo apt install ca-certificates curl jq
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo groupadd docker
sudo usermod -aG docker $USER
wsl.exe --terminate AgentHost
```

make sure it works:

```bash
systemctl status docker
docker run hello-world
docker images
docker image rm $ID...
```

## 3. nvidia-container-toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg   && curl -s -L https://nvidia.github.io/libnvidia-container/experimental/deb/nvidia-container-toolkit.list |     sed 's#deb [^ ]* #&[signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] #' |     sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

make sure it works:

```bash
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi
docker image rm ...
```

## 4. ollama

inference server

```bash
mkdir ~/ollama
cd ~/ollama
cat > docker-compose.yml << EOF
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_storage:/root/.ollama
    environment:
      # Fine-tune GPU and context memory parameters here
      - OLLAMA_CONTEXT_LENGTH=65536
      - OLLAMA_FLASH_ATTENTION=1
      - OLLAMA_KV_CACHE_TYPE=q8_0
      - OLLAMA_KEEP_ALIVE=24h
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

volumes:
  ollama_storage:
    name: ollama_storage
EOF

docker compose up -d
```

it is setup to be auto-started with the wsl instance

## 5. start and stop

### START

```bash
wsl.exe -d AgentHost true
```

### STOP

```bash
wsl.exe --terminate AgentHost
```

## 6. models

### hugging face CLI

for browsing

```bash
curl -LsSf https://hf.co/cli/install.sh | bash

hf models ls --num-parameters "min:6B,max:16B" --search "code" --sort downloads

hf models list Qwen/Qwen2.5-Coder-14B-Instruct-GGUF

ollama pull qwen2.5-coder:14b-instruct-q5_K_M
hf.co/Qwen/Qwen2.5-Coder-14B-Instruct-GGUF:Q4_K_M

ollama pull qwen3:14b
```

## 7. opencode

it's the agentic harness

### bare-metal (inside the WSL dedicated instance) -> NO

```
curl -fsSL https://opencode.ai/install | bash
```

then, run opencode, type command `/models`, and see this model is listed at the bottom of the list

direct launch ollama -> opencode, !! can not work when ollama is containerized but opencode is not
```
ollama ollama launch opencode
```

### test ollama API
Hit the ollama API:
/v1 is OpenAI API
/api is ollama native AIP
```
curl -s http://127.0.0.1:11434/v1/tags | jq
curl -s http://127.0.0.1:11434/api/tags | jq
curl -s http://127.0.0.1:11434/v1/models | jq
```

### configure model(s) from ollam
`~/.config/opencode/opencode.json` like this:

```bash
mkdir -p ~/.config/opencode
cat << 'EOF' > ~/.config/opencode/opencode.json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama",
      "options": {
        "baseURL": "http://127.0.0.1:11434/v1"
      },
      "models": {
        "qwen3:14b": {
          "name": "qwen3-14b"
        }
      }
    }
  }
}
EOF
```

### containerized

put this in `Dockerfile`:
```
FROM debian:bookworm-slim

# Prevent interactive prompts during apt operations
ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies in a single layer and clear apt caches immediately
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    grep \
    ripgrep \
    jq \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install OpenCode binary
RUN curl -fsSL https://opencode.ai/install | bash

# Ensure OpenCode is available in system PATH
ENV PATH="/root/.opencode/bin:${PATH}"

WORKDIR /workspace

CMD ["opencode"]
```

build it:
```
docker build -t opencode-agent:debian .
```

run it:
```
docker run -it --rm   --network=host   -v ~/workspace:/workspace   -v ~/.config/opencode:/root/.config/opencode   opencode-agent:debian
```
