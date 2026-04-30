# Setup and Deployment Guide

This guide walks through setting up the Medical ML API from scratch on a fresh Ubuntu instance with an NVIDIA GPU.

## Table of Contents
1. [System Requirements](#system-requirements)
2. [Initial Setup](#initial-setup)
3. [Model Download](#model-download)
4. [Configuration](#configuration)
5. [Running the API](#running-the-api)
6. [Production Deployment](#production-deployment)
7. [Troubleshooting](#troubleshooting)

## System Requirements

### Hardware
- **GPU**: NVIDIA GPU with 15GB+ VRAM (tested on Tesla T4)
- **RAM**: 16GB+ system RAM recommended
- **Storage**: 20GB+ free space (for models and cache)

### Software
- **OS**: Ubuntu 20.04 or 22.04 (other Linux distros should work)
- **CUDA**: 11.8 or later
- **Python**: 3.10 or 3.11
- **NVIDIA Drivers**: 525+ (comes with CUDA toolkit)

## Initial Setup

### 1. Install CUDA and NVIDIA Drivers

Check if NVIDIA drivers are installed:
```bash
nvidia-smi
```

If not installed, follow the [NVIDIA CUDA Installation Guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/).

Quick install (Ubuntu):
```bash
# Add NVIDIA package repositories
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update

# Install CUDA toolkit
sudo apt-get install cuda-toolkit-11-8

# Add to PATH
echo 'export PATH=/usr/local/cuda-11.8/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-11.8/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### 2. Install Python Dependencies

```bash
# Update system packages
sudo apt-get update
sudo apt-get install -y python3.10 python3.10-venv python3-pip git

# Install audio libraries (required for librosa)
sudo apt-get install -y libsndfile1 ffmpeg
```

### 3. Clone and Setup Project

```bash
# Clone repository
git clone https://github.com/arafath-am/medical-ml-api.git
cd medical-ml-api

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip

# Install PyTorch with CUDA support
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

# Install other dependencies
pip install -r requirements.txt
```

### 4. Verify GPU Access

```python
python3 -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0) if torch.cuda.is_available() else None}')"
```

Expected output:
```
CUDA available: True
GPU: Tesla T4
```

## Model Download

The models need to be downloaded once before first use. They'll be cached locally.

### Option 1: Manual Download (Recommended)

Create a Python script `download_models.py`:

```python
import os
from transformers import AutoTokenizer, AutoModelForCausalLM, pipeline

# Create cache directory
cache_dir = os.path.expanduser("~/ml-platform/cache")
os.makedirs(cache_dir, exist_ok=True)

print("Downloading MedGemma 1.5 4B IT...")
medgemma_cache = f"{cache_dir}/medgemma"
tokenizer = AutoTokenizer.from_pretrained(
    "google/medgemma-1.5-4b-it",
    cache_dir=medgemma_cache
)
model = AutoModelForCausalLM.from_pretrained(
    "google/medgemma-1.5-4b-it",
    cache_dir=medgemma_cache,
    torch_dtype="auto"
)
print(f"✓ MedGemma downloaded to {medgemma_cache}")

print("\nDownloading Google MedASR...")
medasr_cache = f"{cache_dir}/medasr"
pipe = pipeline(
    "automatic-speech-recognition",
    model="google/MedASR",
    cache_dir=medasr_cache
)
print(f"✓ MedASR downloaded to {medasr_cache}")

print("\nAll models downloaded successfully!")
```

Run it:
```bash
python download_models.py
```

This will download ~8GB of models. It may take 10-30 minutes depending on internet speed.

### Option 2: Auto-Download on First Request

Skip manual download and let the API download models automatically on the first request. The first API call will take longer (~10-15 minutes) while models download.

## Configuration

### 1. Update Model Paths

After downloading models, find the exact paths:

```bash
ls -la ~/ml-platform/cache/medgemma/models--google--*/snapshots/
ls -la ~/ml-platform/cache/medasr/models--google--*/snapshots/
```

Update `model_manager.py` with the correct snapshot paths:

```python
# In _load_medgemma()
model_path = "/path/from/ls/command/above"

# In _load_medasr()  
model_path = "/path/from/ls/command/above"
```

### 2. Configure API Keys

Edit `auth.py` and update the `API_KEYS` dictionary:

```python
API_KEYS: Dict[str, dict] = {
    "your-app": {
        "key": "your-secure-key-here",  # Generate with: python -c "import secrets; print(secrets.token_urlsafe(32))"
        "description": "Your Application",
        "created": "2025-04-30",
        "active": True,
    },
}
```

Generate secure keys:
```bash
python -c "import secrets; print('API Key:', secrets.token_urlsafe(32))"
```

### 3. Create Log Directory

```bash
mkdir -p ~/ml-platform/logs/api
```

## Running the API

### Development Mode

```bash
# Activate virtual environment
source venv/bin/activate

# Run the API
python main.py
```

The API will start on `http://0.0.0.0:8000`

Visit `http://localhost:8000/docs` for interactive API documentation.

### Test the API

```bash
# Check health
curl http://localhost:8000/health

# Test transcription (replace with your API key)
curl -X POST "http://localhost:8000/transcribe" \
  -H "X-API-Key: your-api-key-here" \
  -F "audio=@test_audio.wav"
```

## Production Deployment

### 1. Create Systemd Service

Create `/etc/systemd/system/medical-ml-api.service`:

```ini
[Unit]
Description=Medical ML API Service
After=network.target

[Service]
Type=simple
User=your-username
Group=your-username
WorkingDirectory=/home/your-username/medical-ml-api
Environment="PATH=/home/your-username/medical-ml-api/venv/bin"
ExecStart=/home/your-username/medical-ml-api/venv/bin/python main.py
Restart=always
RestartSec=10
StandardOutput=append:/home/your-username/ml-platform/logs/api/service.log
StandardError=append:/home/your-username/ml-platform/logs/api/service.log

[Install]
WantedBy=multi-user.target
```

Replace `your-username` with your actual username.

### 2. Enable and Start Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service (start on boot)
sudo systemctl enable medical-ml-api

# Start service
sudo systemctl start medical-ml-api

# Check status
sudo systemctl status medical-ml-api

# View logs
journalctl -u medical-ml-api -f
```

### 3. Setup Reverse Proxy (Optional)

For production, use nginx or Caddy as a reverse proxy for HTTPS:

**Nginx example** (`/etc/nginx/sites-available/medical-ml-api`):
```nginx
server {
    listen 443 ssl;
    server_name api.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Troubleshooting

### GPU Out of Memory

**Symptoms**: API returns 503 with "GPU out of memory" error

**Solutions**:
1. Reduce `ttl_minutes` in `model_manager.py` to unload models more aggressively
2. Call `/admin/cleanup` endpoint to manually unload models
3. Restart the service: `sudo systemctl restart medical-ml-api`

### Models Not Loading

**Symptoms**: "Model not found" errors

**Solutions**:
1. Verify model paths in `model_manager.py` match actual downloaded paths
2. Check cache directory permissions: `ls -la ~/ml-platform/cache/`
3. Re-download models: `python download_models.py`

### CUDA Not Available

**Symptoms**: `torch.cuda.is_available()` returns `False`

**Solutions**:
1. Verify NVIDIA drivers: `nvidia-smi`
2. Check CUDA installation: `nvcc --version`
3. Reinstall PyTorch with CUDA: `pip install torch --index-url https://download.pytorch.org/whl/cu118 --force-reinstall`

### Port Already in Use

**Symptoms**: "Address already in use" error

**Solutions**:
1. Check what's using port 8000: `sudo lsof -i :8000`
2. Kill the process: `sudo kill -9 <PID>`
3. Change port in `main.py`: Update `uvicorn.run(... port=8001)`

### Audio File Format Issues

**Symptoms**: "Unsupported audio format" errors

**Solutions**:
1. Ensure `ffmpeg` is installed: `sudo apt-get install ffmpeg`
2. Convert audio to WAV: `ffmpeg -i input.mp3 -ar 16000 output.wav`
3. Use standard formats: WAV, MP3, M4A, FLAC

## Performance Tuning

### Optimize Batch Size
For transcription jobs, processing in batches can improve throughput:
- Small files (<1 min): Process individually
- Large files (>5 min): Use chunking (already implemented in `transcribe` endpoint)

### Adjust TTL
Balance memory usage vs. response time by tuning the TTL:
- **Low traffic**: Increase TTL (keep models loaded longer)
- **High memory pressure**: Decrease TTL (unload sooner)

### Pre-warm Models
For production, pre-load models on startup to avoid cold start delays:

Add to `main.py` startup event:
```python
@app.on_event("startup")
async def startup_event():
    logger.info("Pre-warming models...")
    model_manager.get_medgemma()
    model_manager.get_medasr()
    logger.info("Models loaded and ready")
```

---

For additional help, open an issue on GitHub or check the main README.
