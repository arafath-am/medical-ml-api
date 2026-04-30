# API Usage Examples

This directory contains example code for integrating with the Medical ML API in various programming languages.

## Quick Start

All examples require:
1. The API running (locally or deployed)
2. A valid API key
3. Sample audio file (for transcription examples)

## Python Examples

### Basic Transcription
```python
import requests

API_URL = "http://localhost:8000"
API_KEY = "your-api-key-here"

def transcribe_audio(audio_path):
    """Transcribe an audio file to text."""
    with open(audio_path, "rb") as audio_file:
        response = requests.post(
            f"{API_URL}/transcribe",
            headers={"X-API-Key": API_KEY},
            files={"audio": audio_file}
        )
    
    if response.status_code == 200:
        data = response.json()
        print(f"Transcription: {data['text']}")
        print(f"Duration: {data['duration_seconds']}s")
        return data['text']
    else:
        print(f"Error: {response.status_code} - {response.json()}")
        return None

# Example usage
text = transcribe_audio("patient_question.wav")
```

### Medical Query
```python
def ask_medical_question(question, max_tokens=256):
    """Ask a medical question and get an educational response."""
    response = requests.post(
        f"{API_URL}/medical-query",
        headers={
            "X-API-Key": API_KEY,
            "Content-Type": "application/json"
        },
        json={
            "text": question,
            "max_tokens": max_tokens,
            "temperature": 0.7
        }
    )
    
    if response.status_code == 200:
        data = response.json()
        print(f"Question: {question}")
        print(f"Answer: {data['response']}")
        print(f"\n{data['disclaimer']}")
        return data['response']
    else:
        print(f"Error: {response.status_code}")
        return None

# Example usage
answer = ask_medical_question("What causes diabetes?")
```

### Complete Pipeline (Audio → Medical Analysis)
```python
def process_medical_audio(audio_path, max_tokens=256):
    """Process audio through full pipeline: transcribe → analyze."""
    with open(audio_path, "rb") as audio_file:
        response = requests.post(
            f"{API_URL}/audio-to-medical",
            headers={"X-API-Key": API_KEY},
            files={"audio": audio_file},
            data={
                "max_tokens": max_tokens,
                "temperature": 0.7
            }
        )
    
    if response.status_code == 200:
        data = response.json()
        print(f"Transcription: {data['transcription']}")
        print(f"\nMedical Analysis: {data['medical_response']}")
        print(f"\n{data['disclaimer']}")
        return data
    else:
        print(f"Error: {response.status_code}")
        return None

# Example usage
result = process_medical_audio("doctor_dictation.wav")
```

### Health Check
```python
def check_api_health():
    """Check API health and GPU status."""
    response = requests.get(f"{API_URL}/health")
    
    if response.status_code == 200:
        data = response.json()
        print(f"Status: {data['status']}")
        print(f"GPU Available: {data['gpu_available']}")
        if data['gpu_memory']:
            mem = data['gpu_memory']
            print(f"GPU Memory: {mem['allocated_gb']}/{mem['total_gb']} GB used")
        print(f"Loaded Models: {data['loaded_models']}")
        return data
    else:
        print(f"API not healthy: {response.status_code}")
        return None

# Example usage
health = check_api_health()
```

## JavaScript/Node.js Examples

### Basic Transcription (using fetch)
```javascript
const API_URL = "http://localhost:8000";
const API_KEY = "your-api-key-here";

async function transcribeAudio(audioPath) {
    const fs = require('fs');
    const FormData = require('form-data');
    
    const form = new FormData();
    form.append('audio', fs.createReadStream(audioPath));
    
    const response = await fetch(`${API_URL}/transcribe`, {
        method: 'POST',
        headers: {
            'X-API-Key': API_KEY,
            ...form.getHeaders()
        },
        body: form
    });
    
    if (response.ok) {
        const data = await response.json();
        console.log(`Transcription: ${data.text}`);
        return data.text;
    } else {
        console.error(`Error: ${response.status}`);
        return null;
    }
}

// Usage
transcribeAudio('patient_question.wav');
```

### Medical Query
```javascript
async function askMedicalQuestion(question) {
    const response = await fetch(`${API_URL}/medical-query`, {
        method: 'POST',
        headers: {
            'X-API-Key': API_KEY,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            text: question,
            max_tokens: 256,
            temperature: 0.7
        })
    });
    
    if (response.ok) {
        const data = await response.json();
        console.log(`Q: ${question}`);
        console.log(`A: ${data.response}`);
        return data.response;
    } else {
        console.error(`Error: ${response.status}`);
        return null;
    }
}

// Usage
askMedicalQuestion('What is hypertension?');
```

## cURL Examples

### Transcribe Audio
```bash
curl -X POST "http://localhost:8000/transcribe" \
  -H "X-API-Key: your-api-key-here" \
  -F "audio=@patient_question.wav"
```

### Medical Query
```bash
curl -X POST "http://localhost:8000/medical-query" \
  -H "X-API-Key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "What causes asthma?",
    "max_tokens": 256,
    "temperature": 0.7
  }'
```

### Complete Pipeline
```bash
curl -X POST "http://localhost:8000/audio-to-medical" \
  -H "X-API-Key: your-api-key-here" \
  -F "audio=@doctor_notes.wav" \
  -F "max_tokens=512" \
  -F "temperature=0.7"
```

### Health Check
```bash
curl "http://localhost:8000/health"
```

### Force Cleanup (Admin)
```bash
curl -X POST "http://localhost:8000/admin/cleanup" \
  -H "X-API-Key: your-api-key-here"
```

## Java Examples

### Using OkHttp
```java
import okhttp3.*;
import java.io.File;
import org.json.JSONObject;

public class MedicalMLClient {
    private static final String API_URL = "http://localhost:8000";
    private static final String API_KEY = "your-api-key-here";
    private static final OkHttpClient client = new OkHttpClient();

    public static String transcribeAudio(String audioPath) throws Exception {
        File audioFile = new File(audioPath);
        
        RequestBody requestBody = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("audio", audioFile.getName(),
                RequestBody.create(audioFile, MediaType.parse("audio/wav")))
            .build();

        Request request = new Request.Builder()
            .url(API_URL + "/transcribe")
            .header("X-API-Key", API_KEY)
            .post(requestBody)
            .build();

        try (Response response = client.newCall(request).execute()) {
            if (response.isSuccessful()) {
                JSONObject json = new JSONObject(response.body().string());
                return json.getString("text");
            } else {
                throw new Exception("API Error: " + response.code());
            }
        }
    }

    public static String askMedicalQuestion(String question) throws Exception {
        JSONObject json = new JSONObject();
        json.put("text", question);
        json.put("max_tokens", 256);
        json.put("temperature", 0.7);

        RequestBody requestBody = RequestBody.create(
            json.toString(),
            MediaType.parse("application/json")
        );

        Request request = new Request.Builder()
            .url(API_URL + "/medical-query")
            .header("X-API-Key", API_KEY)
            .post(requestBody)
            .build();

        try (Response response = client.newCall(request).execute()) {
            if (response.isSuccessful()) {
                JSONObject responseJson = new JSONObject(response.body().string());
                return responseJson.getString("response");
            } else {
                throw new Exception("API Error: " + response.code());
            }
        }
    }
}
```

## Error Handling Best Practices

### Python with Retry Logic
```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """Create requests session with automatic retries."""
    session = requests.Session()
    
    retry_strategy = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["POST", "GET"]
    )
    
    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    
    return session

# Usage
session = create_session_with_retries()
response = session.post(
    f"{API_URL}/transcribe",
    headers={"X-API-Key": API_KEY},
    files={"audio": open("audio.wav", "rb")}
)
```

### Handle API Errors Gracefully
```python
def safe_medical_query(question):
    """Query with comprehensive error handling."""
    try:
        response = requests.post(
            f"{API_URL}/medical-query",
            headers={"X-API-Key": API_KEY, "Content-Type": "application/json"},
            json={"text": question, "max_tokens": 256},
            timeout=30  # 30 second timeout
        )
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 401:
            print("Error: Invalid API key")
        elif response.status_code == 503:
            print("Error: GPU out of memory. Models may be loading.")
        else:
            print(f"Error: {response.status_code} - {response.text}")
        
        return None
        
    except requests.exceptions.Timeout:
        print("Error: Request timed out")
        return None
    except requests.exceptions.ConnectionError:
        print("Error: Cannot connect to API")
        return None
    except Exception as e:
        print(f"Unexpected error: {str(e)}")
        return None
```

## Integration Patterns

### Async Processing (for batch jobs)
```python
import asyncio
import aiohttp

async def process_multiple_audios(audio_files):
    """Process multiple audio files concurrently."""
    async with aiohttp.ClientSession() as session:
        tasks = []
        
        for audio_file in audio_files:
            task = transcribe_async(session, audio_file)
            tasks.append(task)
        
        results = await asyncio.gather(*tasks)
        return results

async def transcribe_async(session, audio_path):
    """Async transcription."""
    with open(audio_path, 'rb') as f:
        data = aiohttp.FormData()
        data.add_field('audio', f, filename=audio_path)
        
        async with session.post(
            f"{API_URL}/transcribe",
            headers={"X-API-Key": API_KEY},
            data=data
        ) as response:
            return await response.json()

# Usage
audio_files = ['audio1.wav', 'audio2.wav', 'audio3.wav']
results = asyncio.run(process_multiple_audios(audio_files))
```

---

For more examples and integration guidance, check the [main README](../README.md).
