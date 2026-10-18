![Local Genie](../images/local.genie.png)

# Setting up a local agent

Prerequisites: a Windows machine with WSL.

1. A dedicated WSL instance `AgentHost`

```bash
wsl.exe --install -d Debian --name AgentHost
wsl.exe -d AgentHost
```

Remove its auto-mounts, make sure it has systemd in `/etc/wsl.conf`:

```bash
sudo tee /etc/wsl.conf <<EOF
[boot]
systemd=true

[automount]
enabled = false
mountFsTab = false
EOF

sudo passwd -d $USER # remove user password
sudo apt update
sudo apt install ca-certificates curl jq fzf

wsl.exe --terminate AgentHost # reboot
```

1. Share a directory

The personal WSL owns a workspace directory, which has this repository and the other repositories we want to work on.

Let's not use `/etc/fstab` because it prints this error when starting WSL:
> wsl: Processing /etc/fstab with mount -a failed.
Root cause: auto-mount fstab runs before `/wsl` is ready.

Let's not use systemd because its syntax is too verbose.

We will use `.bashrc` or `.bash_local` or `.profile`, choose what works for you.

## In the host: personal WSL

```bash
WORKSPACE=~/workspace/WorkspaceForAgents/
DEST=/wsl/WorkspaceForAgents
RCFILE=~/.bashrc # customize here

mkdir -p $WORKSPACE
mkdir -p $DEST

cat >> $RCFILE << EOF

 # mount bind for sharing Agent Workspace
if ! mountpoint -q $DEST ; then
  sudo mount --bind $WORKSPACE $DEST
fi
EOF

source $RCFILE
```

## In the AgentHost WSL


```bash
WORKSPACE=/wsl/WorkspaceForAgents
DEST=/workspace
RCFILE=~/.profile

sudo mkdir -p $DEST

cat >> $RCFILE << EOF

 # mount bind for sharing Agent Workspace
if ! mountpoint -q $DEST; then
  sudo mount --bind $WORKSPACE $DEST
fi
EOF

source $RCFILE
```



1. Docker

In the `AgentHost` WSL:

```bash
wsl.exe -d AgentHost

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

wsl.exe --terminate AgentHost # reboot
wsl.exe -d AgentHost
```

make sure it works:

```bash
systemctl status docker
docker run hello-world
docker images
docker image rm $ID
```

1. nvidia-container-toolkit

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
docker images
docker image rm $ID
```

1. Ollama

An inference server.

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

It is now setup to be auto-started with the wsl instance, using systemd.

1. Start, Stop, release VRAM

## Start

```bash
wsl.exe -d AgentHost true
```

## Stop

```bash
wsl.exe --terminate AgentHost
```

## Release VRAM only

For gaming, without stopping everything.

From personal WSL:
```bash
wsl.exe -d AgentHost docker restart ollama
```

From `AgentHost` WSL:
```bash
docker restart ollama
```

1. Models

## Hugging Face CLI (optional)

for browsing

```bash
curl -LsSf https://hf.co/cli/install.sh | bash

hf models ls --num-parameters "min:6B,max:16B" --search "code" --sort downloads

hf models list Qwen/Qwen2.5-Coder-14B-Instruct-GGUF
```

## Install some models

For example:

```bash
ollama pull hf.co/mradermacher/Qwen3.8-14B-Instruct-Turbo-GGUF:Q4_K_M
ollama pull hf.co/unsloth/Devstral-Small-2-24B-Instruct-2512-GGUF:IQ4_XS
ollama pull hf.co/unsloth/Qwen3.8-27B-GGUF:UD-IQ3_S
```

1. OpenCode

It is the agentic harness.

## Bare-Metal -> NO

Important security point: we are not installing the agentic harness in the personal WSL, nor in the Windows host. It belongs in the dedicated WSL instance, and then further isolated in a docker container.

The actual installation command is found in the `Dockerfile`.

Corollary: directly launching ollama through opencode or vice versa can not work when ollama and opencode are in different containers, they do not see each other.
```
ollama ollama launch opencode # can not work
```

## Containerized in a Docker container

And inside the WSL dedicated instance.

Put this in `Dockerfile`:
```
FROM debian:bookworm-slim

 # prevent interactive prompts for apt
ENV DEBIAN_FRONTEND=noninteractive

 # install dependencies and clear apt cache
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    grep \
    ripgrep \
    jq \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

 # install OpenCode
RUN curl -fsSL https://opencode.ai/install | bash

 # add to PATH
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
docker run -it --rm \
  --network=host \
  -v ~/.config/opencode:/root/.config/opencode \
  -v ~/.local/share/opencode:/root/.local/share/opencode \
  -v /workspace:/workspace \
  opencode-agent:debian \
  $@
```

It needs outside directories for the config, the state (to save sessions) and the repositories.

## Auto-discover models

I can not get OpenCode to auto discover the local Ollama models.

Here's how to update the model list from the API call:

## Test Ollama API
Hit the ollama API:
```
curl -s http://127.0.0.1:11434/api/tags  | jq  # api is OpenCode schema
curl -s http://127.0.0.1:11434/v1/models | jq  # v1 is OpenAI schema
```

## Convert `/api/tags` to `opencode.json`

```bash
#!/usr/bin/env bash

curl -s http://localhost:11434/api/tags | jq '{
  "$schema": "https://opencode.ai/config.json",
  "disabled_providers": [
    "opencode"
  ],
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama",
      "options": {
        "baseURL": "http://localhost:11434/v1"
      },
      "models": (
        [ .models[] | {
            key: .model,
            value: {
              name: (
                (.model | sub("^(?:[^/]+/){2}"; "") | sub("-GGUF"; "")) +
                " (" + ((.size / 1000000000 | floor) | tostring) + " GB)"
              )
            }
          }
        ] | from_entries
      )
    }
  }
}'
```

## Update the list like this:

```
curl ... |& tee ~/.config/opencode/opencode.json | jq
```

verify:

```
jq '.provider.ollama.models[].name' ~/.config/opencode/opencode.json
```

1. TODO

[*] hide external providers
[*] share an agentic-workspace dir between all
[ ] put all this content in dedicated files
[ ] after registering models, prompt for default with fzf
[ ] save favorite models, they are lost on container restart
[ ] set verbose
[ ] customize context size per model
[ ] find a few models that fit in VRAM
[ ] test the models with the prompt for the CatPi bug fix
