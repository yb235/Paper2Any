# Paper2Any API Reference

## Overview

This document provides a complete reference for the Paper2Any REST API. All endpoints are prefixed with `/api/`.

**Base URL**: `http://localhost:8000` (development)

---

## Authentication

### API Key Middleware

Most endpoints require an API key for LLM access. The API key should be provided in request body parameters.

```bash
# Example request with API key
curl -X POST http://localhost:8000/api/paper2figure \
  -F "pdf_file=@paper.pdf" \
  -F "api_key=your-llm-api-key" \
  -F "model=gpt-4o"
```

---

## Endpoints

### 1. Verify LLM Connection

**POST** `/api/verify-llm`

Verify that your LLM API credentials are valid.

**Request Body**:
```json
{
  "api_url": "https://api.openai.com/v1",
  "api_key": "sk-...",
  "model": "gpt-4o"
}
```

**Response** (Success):
```json
{
  "success": true
}
```

**Response** (Failure):
```json
{
  "success": false,
  "error": "API Error 401: Invalid API key"
}
```

**Example**:
```bash
curl -X POST http://localhost:8000/api/verify-llm \
  -H "Content-Type: application/json" \
  -d '{
    "api_url": "https://api.openai.com/v1",
    "api_key": "sk-...",
    "model": "gpt-4o"
  }'
```

---

### 2. Paper2Figure - Generate Diagram from Paper

**POST** `/api/paper2figure`

Generate an editable model architecture diagram from a research paper or description.

**Request Parameters** (multipart/form-data):

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `pdf_file` | File | Conditional* | - | PDF file to process |
| `image_file` | File | Conditional* | - | Image file to process |
| `text_input` | String | Conditional* | - | Text description |
| `input_type` | String | Yes | - | One of: `PDF`, `TEXT`, `FIGURE` |
| `mask_detail_level` | Integer | No | 2 | Layout detection detail (1-3) |
| `api_key` | String | Yes | - | LLM API key |
| `api_url` | String | Yes | - | LLM API endpoint |
| `model` | String | Yes | - | LLM model name |
| `gen_fig_model` | String | No | "dall-e-3" | Image generation model |
| `aspect_ratio` | String | No | "16:9" | Output aspect ratio |
| `mineru_port` | Integer | No | 8010 | MinerU service port |
| `prev_image` | String | No | "" | Path to previous image (for editing) |
| `edit_prompt` | String | No | "" | Edit instructions |

*One of `pdf_file`, `image_file`, or `text_input` is required based on `input_type`.

**Response**:
```json
{
  "status": "success",
  "ppt_url": "http://localhost:8000/outputs/paper2figure/20260108_abc123/output/presentation.pptx",
  "draft_image_url": "http://localhost:8000/outputs/paper2figure/20260108_abc123/output/figure_draft.jpg",
  "task_id": "20260108_abc123",
  "processing_time": 142.5
}
```

**Example - From PDF**:
```bash
curl -X POST http://localhost:8000/api/paper2figure \
  -F "pdf_file=@paper.pdf" \
  -F "input_type=PDF" \
  -F "mask_detail_level=2" \
  -F "api_key=sk-..." \
  -F "api_url=https://api.openai.com/v1" \
  -F "model=gpt-4o" \
  -F "aspect_ratio=16:9"
```

**Example - From Text**:
```bash
curl -X POST http://localhost:8000/api/paper2figure \
  -F "text_input=A transformer model with encoder-decoder architecture..." \
  -F "input_type=TEXT" \
  -F "api_key=sk-..." \
  -F "api_url=https://api.openai.com/v1" \
  -F "model=gpt-4o"
```

**Example - Edit Existing Figure**:
```bash
curl -X POST http://localhost:8000/api/paper2figure \
  -F "image_file=@existing_diagram.png" \
  -F "input_type=FIGURE" \
  -F "edit_prompt=Make the attention mechanism larger" \
  -F "api_key=sk-..." \
  -F "api_url=https://api.openai.com/v1" \
  -F "model=gpt-4o"
```

---

### 3. Paper2PPT - Generate Presentation from Paper

**POST** `/api/paper2ppt/generate`

Generate a complete PowerPoint presentation from a research paper.

**Request Parameters** (multipart/form-data):

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `pdf_file` | File | Yes | - | PDF file to process |
| `num_pages` | Integer | No | 10 | Target number of slides |
| `style` | String | No | "beamer" | Presentation style |
| `api_key` | String | Yes | - | LLM API key |
| `api_url` | String | Yes | - | LLM API endpoint |
| `model` | String | Yes | - | LLM model name |
| `include_figures` | Boolean | No | true | Extract and include figures |
| `include_tables` | Boolean | No | true | Extract and include tables |

**Response**:
```json
{
  "status": "success",
  "ppt_url": "http://localhost:8000/outputs/paper2ppt/20260108_xyz789/output/presentation.pptx",
  "num_slides": 12,
  "task_id": "20260108_xyz789",
  "processing_time": 285.3
}
```

**Example**:
```bash
curl -X POST http://localhost:8000/api/paper2ppt/generate \
  -F "pdf_file=@paper.pdf" \
  -F "num_pages=10" \
  -F "style=beamer" \
  -F "api_key=sk-..." \
  -F "api_url=https://api.openai.com/v1" \
  -F "model=gpt-4o" \
  -F "include_figures=true" \
  -F "include_tables=true"
```

---

### 4. PDF2PPT - Convert PDF to Editable PPT

**POST** `/api/pdf2ppt`

Convert a PDF presentation to editable PowerPoint format while preserving layout.

**Request Parameters** (multipart/form-data):

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `pdf_file` | File | Yes | - | PDF file to convert |
| `preserve_layout` | Boolean | No | true | Preserve original layout |
| `ocr_enabled` | Boolean | No | true | Enable OCR for text extraction |
| `remove_backgrounds` | Boolean | No | false | Remove image backgrounds |
| `mineru_port` | Integer | No | 8010 | MinerU service port |
| `sam_port` | Integer | No | 8020 | SAM service port |

**Response**:
```json
{
  "status": "success",
  "ppt_url": "http://localhost:8000/outputs/pdf2ppt/20260108_def456/output/converted.pptx",
  "num_pages": 24,
  "task_id": "20260108_def456",
  "processing_time": 95.7
}
```

**Example**:
```bash
curl -X POST http://localhost:8000/api/pdf2ppt \
  -F "pdf_file=@slides.pdf" \
  -F "preserve_layout=true" \
  -F "ocr_enabled=true" \
  -F "remove_backgrounds=false"
```

---

### 5. Paper2Video - Generate Video Script

**POST** `/api/paper2video/generate`

Generate a video script with subtitles and cursor positions from a paper.

**Request Parameters** (multipart/form-data):

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `pdf_file` | File | Yes | - | PDF file to process |
| `duration` | Integer | No | 300 | Target video duration (seconds) |
| `api_key` | String | Yes | - | LLM API key |
| `api_url` | String | Yes | - | LLM API endpoint |
| `model` | String | Yes | - | LLM model name |

**Response**:
```json
{
  "status": "success",
  "script_url": "http://localhost:8000/outputs/paper2video/20260108_ghi012/output/script.json",
  "ppt_url": "http://localhost:8000/outputs/paper2video/20260108_ghi012/output/slides.pptx",
  "task_id": "20260108_ghi012",
  "estimated_duration": 312,
  "processing_time": 178.2
}
```

---

### 6. List History Files

**GET** `/api/paper2figure/history_files`

List all previously generated files for a user (identified by invite code).

**Query Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `invite_code` | String | Yes | User's invite code |

**Response**:
```json
{
  "success": true,
  "files": [
    "http://localhost:8000/outputs/abc123/paper2figure/20260108_123456/presentation.pptx",
    "http://localhost:8000/outputs/abc123/paper2figure/20260107_789012/presentation.pptx",
    "http://localhost:8000/outputs/abc123/paper2ppt/20260106_345678/slides.pptx"
  ]
}
```

**Example**:
```bash
curl "http://localhost:8000/api/paper2figure/history_files?invite_code=abc123"
```

---

## Response Codes

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | Success | Request completed successfully |
| 400 | Bad Request | Invalid parameters or missing required fields |
| 401 | Unauthorized | Invalid API key |
| 413 | Payload Too Large | File size exceeds limit (default: 10MB) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error during processing |
| 503 | Service Unavailable | Model service (MinerU/SAM) is down |

---

## Error Response Format

All errors return a consistent format:

```json
{
  "error": "Error description",
  "detail": "Detailed error message with technical info",
  "status_code": 400
}
```

**Examples**:

```json
{
  "error": "Invalid input type",
  "detail": "input_type must be one of: PDF, TEXT, FIGURE",
  "status_code": 400
}
```

```json
{
  "error": "LLM API error",
  "detail": "API Error 401: Invalid API key",
  "status_code": 401
}
```

```json
{
  "error": "Model service unavailable",
  "detail": "MinerU service at port 8010 is not responding",
  "status_code": 503
}
```

---

## Rate Limiting

**Default Limits**:
- 10 requests per minute per user
- 100 requests per day per user
- Concurrent request limit: 1 per user

**Rate Limit Headers**:
```
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 7
X-RateLimit-Reset: 1704729600
```

**Rate Limit Response**:
```json
{
  "error": "Rate limit exceeded",
  "detail": "Maximum 10 requests per minute. Try again in 45 seconds.",
  "retry_after": 45,
  "status_code": 429
}
```

---

## File Size Limits

| File Type | Max Size | Notes |
|-----------|----------|-------|
| PDF | 10 MB | Configurable via `MAX_FILE_SIZE` env var |
| Image | 5 MB | PNG, JPG, JPEG formats |
| Text | 100 KB | Plain text only |

---

## Webhook Support (Optional)

For long-running tasks, configure webhooks to receive completion notifications.

**Setup**:
1. Provide `webhook_url` in request
2. Server POSTs to this URL when complete

**Request**:
```bash
curl -X POST http://localhost:8000/api/paper2figure \
  -F "pdf_file=@paper.pdf" \
  -F "input_type=PDF" \
  -F "webhook_url=https://your-server.com/webhook" \
  ...
```

**Webhook Payload**:
```json
{
  "task_id": "20260108_abc123",
  "status": "success",
  "ppt_url": "http://localhost:8000/outputs/.../presentation.pptx",
  "processing_time": 142.5,
  "timestamp": "2026-01-08T12:34:56Z"
}
```

---

## Python Client Example

```python
import requests
from pathlib import Path

class Paper2AnyClient:
    def __init__(self, base_url="http://localhost:8000", api_key=None):
        self.base_url = base_url
        self.api_key = api_key
    
    def verify_llm(self, api_url, model="gpt-4o"):
        """Verify LLM connection"""
        response = requests.post(
            f"{self.base_url}/api/verify-llm",
            json={
                "api_url": api_url,
                "api_key": self.api_key,
                "model": model
            }
        )
        return response.json()
    
    def paper2figure(
        self,
        pdf_path=None,
        text=None,
        image_path=None,
        input_type="PDF",
        api_url="https://api.openai.com/v1",
        model="gpt-4o",
        mask_detail_level=2,
        aspect_ratio="16:9"
    ):
        """Generate figure from paper"""
        files = {}
        data = {
            "input_type": input_type,
            "api_key": self.api_key,
            "api_url": api_url,
            "model": model,
            "mask_detail_level": mask_detail_level,
            "aspect_ratio": aspect_ratio
        }
        
        if pdf_path:
            files["pdf_file"] = open(pdf_path, "rb")
        elif image_path:
            files["image_file"] = open(image_path, "rb")
        elif text:
            data["text_input"] = text
        
        response = requests.post(
            f"{self.base_url}/api/paper2figure",
            files=files,
            data=data
        )
        
        # Close file handles
        for f in files.values():
            f.close()
        
        return response.json()
    
    def paper2ppt(
        self,
        pdf_path,
        api_url="https://api.openai.com/v1",
        model="gpt-4o",
        num_pages=10,
        style="beamer"
    ):
        """Generate PPT from paper"""
        with open(pdf_path, "rb") as f:
            response = requests.post(
                f"{self.base_url}/api/paper2ppt/generate",
                files={"pdf_file": f},
                data={
                    "api_key": self.api_key,
                    "api_url": api_url,
                    "model": model,
                    "num_pages": num_pages,
                    "style": style
                }
            )
        return response.json()
    
    def pdf2ppt(self, pdf_path, preserve_layout=True, ocr_enabled=True):
        """Convert PDF to editable PPT"""
        with open(pdf_path, "rb") as f:
            response = requests.post(
                f"{self.base_url}/api/pdf2ppt",
                files={"pdf_file": f},
                data={
                    "preserve_layout": preserve_layout,
                    "ocr_enabled": ocr_enabled
                }
            )
        return response.json()
    
    def download_file(self, url, save_path):
        """Download result file"""
        response = requests.get(url)
        Path(save_path).write_bytes(response.content)
        return save_path


# Usage example
client = Paper2AnyClient(api_key="sk-...")

# Verify connection
result = client.verify_llm("https://api.openai.com/v1")
print(f"LLM verified: {result['success']}")

# Generate figure
result = client.paper2figure(
    pdf_path="paper.pdf",
    input_type="PDF",
    mask_detail_level=2
)
print(f"PPT URL: {result['ppt_url']}")

# Download result
client.download_file(result['ppt_url'], "output.pptx")
```

---

## JavaScript Client Example

```javascript
class Paper2AnyClient {
  constructor(baseUrl = 'http://localhost:8000', apiKey = null) {
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
  }

  async verifyLLM(apiUrl, model = 'gpt-4o') {
    const response = await fetch(`${this.baseUrl}/api/verify-llm`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        api_url: apiUrl,
        api_key: this.apiKey,
        model: model
      })
    });
    return await response.json();
  }

  async paper2figure({
    pdfFile = null,
    textInput = null,
    imageFile = null,
    inputType = 'PDF',
    apiUrl = 'https://api.openai.com/v1',
    model = 'gpt-4o',
    maskDetailLevel = 2,
    aspectRatio = '16:9'
  }) {
    const formData = new FormData();
    
    if (pdfFile) formData.append('pdf_file', pdfFile);
    if (imageFile) formData.append('image_file', imageFile);
    if (textInput) formData.append('text_input', textInput);
    
    formData.append('input_type', inputType);
    formData.append('api_key', this.apiKey);
    formData.append('api_url', apiUrl);
    formData.append('model', model);
    formData.append('mask_detail_level', maskDetailLevel);
    formData.append('aspect_ratio', aspectRatio);

    const response = await fetch(`${this.baseUrl}/api/paper2figure`, {
      method: 'POST',
      body: formData
    });
    
    return await response.json();
  }

  async paper2ppt({
    pdfFile,
    apiUrl = 'https://api.openai.com/v1',
    model = 'gpt-4o',
    numPages = 10,
    style = 'beamer'
  }) {
    const formData = new FormData();
    formData.append('pdf_file', pdfFile);
    formData.append('api_key', this.apiKey);
    formData.append('api_url', apiUrl);
    formData.append('model', model);
    formData.append('num_pages', numPages);
    formData.append('style', style);

    const response = await fetch(`${this.baseUrl}/api/paper2ppt/generate`, {
      method: 'POST',
      body: formData
    });
    
    return await response.json();
  }
}

// Usage
const client = new Paper2AnyClient(apiKey: 'sk-...');

// From file input
const fileInput = document.querySelector('#pdf-upload');
const file = fileInput.files[0];

const result = await client.paper2figure({
  pdfFile: file,
  inputType: 'PDF',
  maskDetailLevel: 2
});

console.log('PPT URL:', result.ppt_url);
```

---

## Testing

### Health Check

```bash
curl http://localhost:8000/health
# Response: {"status": "ok"}
```

### Test Workflow

1. **Verify LLM**
2. **Upload test file**
3. **Check processing status**
4. **Download result**

```bash
# 1. Verify
curl -X POST http://localhost:8000/api/verify-llm \
  -H "Content-Type: application/json" \
  -d '{"api_url": "...", "api_key": "...", "model": "gpt-4o"}'

# 2. Generate
curl -X POST http://localhost:8000/api/paper2figure \
  -F "pdf_file=@test.pdf" \
  -F "input_type=PDF" \
  -F "api_key=..." \
  -F "api_url=..." \
  -F "model=gpt-4o"

# 3. Download
wget http://localhost:8000/outputs/.../presentation.pptx
```

---

## Support

For issues or questions:
- 📖 See [User Guide](./user-guide.md) for usage help
- 🏗️ See [Architecture Guide](./architecture.md) for technical details
- 🐛 Report bugs on GitHub Issues
- 💬 Ask questions in GitHub Discussions

