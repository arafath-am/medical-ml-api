# Medical ML API

A production-ready FastAPI service that provides medical AI capabilities through a REST API. Built for healthcare applications requiring medical speech-to-text transcription and medical question answering.

## Overview

This API combines two specialized medical AI models:
- **Google MedASR**: Medical speech recognition optimized for clinical conversations
- **MedGemma 1.5 4B**: Google's instruction-tuned medical language model

The service handles the complexity of GPU memory management, model loading, and provides a clean REST interface for integration with existing healthcare systems.

## Key Features

- **Medical Speech Transcription**: Convert doctor-patient conversations, medical dictations, and clinical audio to text
- **Medical Q&A**: Answer medical questions and explain medical concepts (educational purposes only)
- **Combined Pipeline**: Process audio directly into medical analysis in one API call
- **Smart Memory Management**: Lazy loading and automatic GPU cleanup prevent out-of-memory issues
- **API Key Authentication**: Secure access control for production deployments
- **Comprehensive Logging**: Full request/response logging for debugging and audit trails

## Tech Stack

- **Framework**: FastAPI (async Python web framework)
- **ML/AI**: PyTorch, Transformers (Hugging Face)
- **Hardware**: NVIDIA GPU (tested on Tesla T4, 15GB VRAM)
- **Authentication**: API key header-based auth
- **Deployment**: Systemd service on Ubuntu/GCP

## Architecture

The service uses a custom model lifecycle manager that:
1. **Lazy loads** models only when first requested (saves GPU memory)
2. **Caches** loaded models in VRAM for fast subsequent requests
3. **Auto-unloads** idle models after 10 minutes (configurable TTL)
4. **Prevents OOM** by managing GPU memory proactively

This design allows both models to coexist on a 15GB GPU without manual intervention.

## API Endpoints

### Health Check
```bash
GET /health
```
Returns GPU status and loaded model information.

### Transcribe Audio
```bash
POST /transcribe
Content-Type: multipart/form-data
X-API-Key: your-api-key

# Body: audio file (wav, mp3, m4a, flac, ogg)
```

**Response:**
```json
{
  "text": "The patient presents with persistent cough...",
  "language": "en",
  "model": "google/medasr",
  "duration_seconds": 45.2
}
```

### Medical Query
```bash
POST /medical-query
Content-Type: application/json
X-API-Key: your-api-key

{
  "text": "What causes hypertension?",
  "max_tokens": 256,
  "temperature": 0.7
}
```

**Response:**
```json
{
  "response": "Hypertension, or high blood pressure, can be caused by...",
  "disclaimer": "⚠️ This is for educational purposes only...",
  "model": "medgemma-1.5-4b",
  "tokens_generated": 187
}
```

### Audio-to-Medical Pipeline
```bash
POST /audio-to-medical
Content-Type: multipart/form-data
X-API-Key: your-api-key

# Body: audio file + optional max_tokens, temperature
```

Combines transcription + medical analysis in one call.

## Installation

### Prerequisites
- Ubuntu 20.04+ (or similar Linux)
- NVIDIA GPU with 15GB+ VRAM
- CUDA 11.8+ installed
- Python 3.10+

### Setup

1. **Clone the repository**
```bash
git clone https://github.com/arafath-am/medical-ml-api.git
cd medical-ml-api
```

2. **Create virtual environment**
```bash
python3 -m venv venv
source venv/bin/activate
```

3. **Install dependencies**
```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
pip install transformers accelerate sentencepiece librosa soundfile fastapi uvicorn python-multipart
```

4. **Download models** (one-time setup)
```python
from transformers import AutoTokenizer, AutoModelForCausalLM, pipeline

# Download MedGemma
model = AutoModelForCausalLM.from_pretrained(
    "google/medgemma-1.5-4b-it",
    cache_dir="~/ml-platform/cache/medgemma"
)

# Download MedASR
pipe = pipeline(
    "automatic-speech-recognition",
    model="google/MedASR",
    cache_dir="~/ml-platform/cache/medasr"
)
```

5. **Configure paths** (in `model_manager.py`)
Update the `cache_dir` paths to match where you downloaded models.

6. **Generate API keys**
```bash
python auth.py
# Copy one of the generated keys for testing
```

## Running the API

### Development
```bash
python main.py
# API runs on http://0.0.0.0:8000
# Docs available at http://0.0.0.0:8000/docs
```

### Production (systemd service)

Create `/etc/systemd/system/medical-ml-api.service`:
```ini
[Unit]
Description=Medical ML API
After=network.target

[Service]
Type=simple
User=your-username
WorkingDirectory=/path/to/medical-ml-api
Environment="PATH=/path/to/venv/bin"
ExecStart=/path/to/venv/bin/python main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable medical-ml-api
sudo systemctl start medical-ml-api
sudo systemctl status medical-ml-api
```

## Configuration

### Model Settings (model_manager.py)
- `ttl_minutes`: How long to keep models in memory (default: 10)
- `cache_dir`: Where models are stored

### API Settings (main.py)
- `host`: API bind address (default: 0.0.0.0)
- `port`: API port (default: 8000)

### Authentication (auth.py)
- Add/remove API keys in the `API_KEYS` dictionary
- Configure public endpoints that don't require auth

## Usage Examples

### cURL
```bash
# Transcribe audio
curl -X POST "http://localhost:8000/transcribe" \
  -H "X-API-Key: your-api-key" \
  -F "audio=@patient_question.wav"

# Medical query
curl -X POST "http://localhost:8000/medical-query" \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{"text": "What is diabetes?"}'
```

### Python
```python
import requests

API_URL = "http://localhost:8000"
API_KEY = "your-api-key"

# Transcribe
with open("audio.wav", "rb") as f:
    response = requests.post(
        f"{API_URL}/transcribe",
        headers={"X-API-Key": API_KEY},
        files={"audio": f}
    )
    print(response.json()["text"])

# Medical query
response = requests.post(
    f"{API_URL}/medical-query",
    headers={"X-API-Key": API_KEY},
    json={"text": "Explain hypertension"}
)
print(response.json()["response"])
```

## Project Structure

```
medical-ml-api/
├── main.py              # FastAPI app, endpoints, request handling
├── model_manager.py     # GPU memory management, lazy loading
├── auth.py              # API key authentication
├── requirements.txt     # Python dependencies (create this)
└── README.md           # This file
```

## Performance

Tested on **GCP VM with NVIDIA Tesla T4 (15GB)**:
- **First request** (cold start): ~8-12s (model loading time)
- **Subsequent requests**: 2-4s (models cached in GPU)
- **Transcription**: ~0.5x realtime (1 min audio = 30s processing)
- **Medical query**: 1-2s per response (256 tokens)

## Security Considerations

⚠️ **Important**: This implementation uses static API keys for development. For production:
- Move API keys to environment variables
- Use a proper secrets management system (AWS Secrets Manager, HashiCorp Vault)
- Implement rate limiting
- Add HTTPS/TLS termination (use nginx/Caddy as reverse proxy)
- Consider OAuth2 for user authentication

## Medical Disclaimer

This API is designed for **educational and informational purposes only**. It should NOT be used for:
- Medical diagnosis
- Treatment recommendations
- Prescribing medications
- Making clinical decisions

All outputs include disclaimers reminding users to consult qualified healthcare professionals.

## Future Enhancements

- [ ] Add request rate limiting
- [ ] Implement API usage analytics
- [ ] Support batch processing for multiple audio files
- [ ] Add WebSocket support for streaming responses
- [ ] Containerize with Docker
- [ ] Add Prometheus metrics for monitoring
- [ ] Implement HIPAA-compliant audit logging

## Contributing

This is a personal project, but suggestions and improvements are welcome. Feel free to open issues or submit pull requests.

## License

MIT License - see LICENSE file for details

## Author

**Arafath Amer**  
Senior Platform Engineer | AI/ML Engineer  
[GitHub](https://github.com/arafath-am) | [LinkedIn](https://www.linkedin.com/in/arafath-am)

---

*Built with ❤️ for advancing medical AI accessibility*
