# OpenAvatarChat Runpod Deployment Guide

This guide provides step-by-step instructions for deploying OpenAvatarChat to Runpod using Docker, with PowerShell commands for Windows.

## Prerequisites

- Windows with PowerShell
- Docker Desktop installed with GPU support
- Git installed
- Docker Hub account (username: `lukecarniege`)
- Runpod account

## Table of Contents

1. [Initialize Git Submodules](#1-initialize-git-submodules)
2. [Build Docker Image](#2-build-docker-image)
3. [Local Testing](#3-local-testing)
4. [Push to Docker Hub](#4-push-to-docker-hub)
5. [Runpod Configuration](#5-runpod-configuration)

---

## 1. Initialize Git Submodules

Initialize all git submodules with depth 1 for faster cloning:

```powershell
# Navigate to the repository root
cd OpenAvatarChat1

# Initialize and update submodules with depth 1
git submodule update --init --recursive --depth 1
```

---

## 2. Build Docker Image

Build the Docker image using the `chat_with_openai_compatible_bailian_cosyvoice.yaml` configuration:

```powershell
# Build the Docker image with the specified config
docker build `
    --build-arg CONFIG_FILE=config/chat_with_openai_compatible_bailian_cosyvoice.yaml `
    -t open-avatar-chat:0.0.1 `
    -f Dockerfile.cuda12.8 `
    .
```

**Alternative:** Use the standard Dockerfile (CUDA 12.2):

```powershell
docker build `
    --build-arg CONFIG_FILE=config/chat_with_openai_compatible_bailian_cosyvoice.yaml `
    -t open-avatar-chat:0.0.1 `
    .
```

---

## 3. Local Testing

### 3.1 Create Environment File

Create a `.env` file in your project root with your API keys:

```powershell
# Create .env file (edit with your actual keys)
@"
DASHSCOPE_API_KEY=your_dashscope_api_key_here
"@ | Out-File -FilePath .env -Encoding UTF8
```

### 3.2 Run Container Locally

Test the Docker container locally on port 8888 with GPU support:

```powershell
# Run with GPU support, mounting required directories and loading .env
docker run --rm --gpus all -it --name open-avatar-chat `
    -p 8888:8282 `
    -v "${PWD}/models:/root/open-avatar-chat/models" `
    -v "${PWD}/config:/root/open-avatar-chat/config" `
    -v "${PWD}/ssl_certs:/root/open-avatar-chat/ssl_certs" `
    -v "${PWD}/build:/root/open-avatar-chat/build" `
    --env-file .env `
    open-avatar-chat:0.0.1 `
    --config config/chat_with_openai_compatible_bailian_cosyvoice.yaml
```

**With network host mode (alternative):**

```powershell
docker run --rm --gpus all -it --name open-avatar-chat `
    --network=host `
    -v "${PWD}/models:/root/open-avatar-chat/models" `
    -v "${PWD}/config:/root/open-avatar-chat/config" `
    -v "${PWD}/ssl_certs:/root/open-avatar-chat/ssl_certs" `
    -v "${PWD}/build:/root/open-avatar-chat/build" `
    --env-file .env `
    open-avatar-chat:0.0.1 `
    --config config/chat_with_openai_compatible_bailian_cosyvoice.yaml
```

### 3.3 Verify Local Deployment

Open your browser and navigate to:
- **Local:** `https://localhost:8888`

---

## 4. Push to Docker Hub

### 4.1 Login to Docker Hub

```powershell
docker login
# Enter username: lukecarniege
# Enter password when prompted
```

### 4.2 Tag the Image

```powershell
# Tag for Docker Hub
docker tag open-avatar-chat:0.0.1 lukecarniege/open-avatar-chat:0.0.1
docker tag open-avatar-chat:0.0.1 lukecarniege/open-avatar-chat:latest
```

### 4.3 Push to Docker Hub

```powershell
# Push both tags
docker push lukecarniege/open-avatar-chat:0.0.1
docker push lukecarniege/open-avatar-chat:latest
```

---

## 5. Runpod Configuration

### 5.1 Pod Configuration

| Setting | Value |
|---------|-------|
| **Container Image** | `lukecarniege/open-avatar-chat:0.0.1` |
| **GPU Type** | NVIDIA GPU (RTX 3090, RTX 4090, A100, etc.) |
| **Container Disk** | 50GB minimum |
| **Volume Disk** | 100GB+ (for models) |
| **Volume Mount Path** | `/root/open-avatar-chat/models` |

### 5.2 Environment Variables

Set the following environment variables in your Runpod pod:

| Variable | Description | Example |
|----------|-------------|---------|
| `DASHSCOPE_API_KEY` | Alibaba DashScope API key for CosyVoice TTS | `sk-xxxxxxxxxxxxxxxx` |
| `PYTHONUNBUFFERED` | Disable Python output buffering | `1` |

**Optional environment variables:**

| Variable | Description | Default |
|----------|-------------|---------|
| `PYTORCH_JIT` | Disable PyTorch JIT (required for MuseTalk) | `0` |
| `PYTHONUTF8` | Force UTF-8 encoding | `1` |

### 5.3 Exposed Ports

Configure the following HTTP port in Runpod:

| Internal Port | External Port | Protocol |
|---------------|---------------|----------|
| 8282 | 8888 | HTTPS |

### 5.4 Startup Command

Use this as the Docker command override in Runpod:

```bash
uv run src/demo.py --config config/chat_with_openai_compatible_bailian_cosyvoice.yaml
```

**Or with custom port:**

```bash
uv run src/demo.py --config config/chat_with_openai_compatible_bailian_cosyvoice.yaml --port 8282
```

### 5.5 Access Your Deployment

Once the pod is running, access your OpenAvatarChat instance at:

```
https://2nlt0ao42gkivj-8888.proxy.runpod.net/
```

---

## Troubleshooting

### Common Issues

1. **GPU not detected:**
   - Ensure NVIDIA drivers are installed on the host
   - Check that `--gpus all` flag is set

2. **Connection timeout:**
   - Verify the correct port mapping (8282 internal → 8888 external)
   - Check if SSL certificates are properly mounted

3. **API key errors:**
   - Ensure `DASHSCOPE_API_KEY` is set correctly
   - Verify the API key is valid and has sufficient credits

4. **Model loading failures:**
   - Ensure models are properly downloaded to the volume
   - Check volume mount path matches `/root/open-avatar-chat/models`

### View Container Logs

```powershell
# Local Docker
docker logs open-avatar-chat

# Follow logs
docker logs -f open-avatar-chat
```

### Enter Container Shell

```powershell
docker exec -it open-avatar-chat /bin/bash
```

---

## Quick Reference: Complete Deployment Script

```powershell
# 1. Clone and setup
git clone https://github.com/lukecarniege/OpenAvatarChat1.git
cd OpenAvatarChat1
git submodule update --init --recursive --depth 1

# 2. Build image
docker build `
    --build-arg CONFIG_FILE=config/chat_with_openai_compatible_bailian_cosyvoice.yaml `
    -t open-avatar-chat:0.0.1 `
    -f Dockerfile.cuda12.8 `
    .

# 3. Test locally
docker run --rm --gpus all -it --name open-avatar-chat `
    -p 8888:8282 `
    -v "${PWD}/models:/root/open-avatar-chat/models" `
    -v "${PWD}/config:/root/open-avatar-chat/config" `
    -v "${PWD}/ssl_certs:/root/open-avatar-chat/ssl_certs" `
    -v "${PWD}/build:/root/open-avatar-chat/build" `
    --env-file .env `
    open-avatar-chat:0.0.1 `
    --config config/chat_with_openai_compatible_bailian_cosyvoice.yaml

# 4. Push to Docker Hub
docker login
docker tag open-avatar-chat:0.0.1 lukecarniege/open-avatar-chat:0.0.1
docker push lukecarniege/open-avatar-chat:0.0.1
```

---

## Additional Resources

- [OpenAvatarChat README](../README.md)
- [FAQ](./FAQ.md)
- [Runpod Documentation](https://docs.runpod.io/)
- [Docker GPU Support](https://docs.docker.com/config/containers/resource_constraints/#gpu)
