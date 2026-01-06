# FastAPI Documentation

Complete API reference for UC2 DMD FastAPI Server.

## Overview

The DMD FastAPI server provides a RESTful interface for pattern control and system management. It automatically starts as the main entry point and handles image display requests, health monitoring, and status reporting.

**Base URL:** `http://localhost:8000`

**Auto-generated API docs:** `http://localhost:8000/docs` (Swagger UI)

---

## Core Endpoints

### Health Check

#### GET `/health`

Check if the server is running and responsive.

**Request:**
```bash
curl http://localhost:8000/health
```

**Response (200 OK):**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-06T10:30:45.123456Z"
}
```

**Status Codes:**
- `200 OK` - Server is healthy
- `503 Service Unavailable` - Server has errors

---

### Pattern Display

#### POST `/pattern/display`

Display a pattern image on the DMD.

**Request:**
```bash
curl -X POST http://localhost:8000/pattern/display \
  -F "file=@pattern.png"
```

**Parameters:**
- `file` (multipart/form-data, required) - PNG or image file to display
- `duration` (optional) - Display duration in milliseconds (-1 for infinite)

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Pattern displayed successfully",
  "filename": "pattern.png",
  "display_duration": -1,
  "resolution": {
    "width": 1280,
    "height": 720
  }
}
```

**Error Response (400 Bad Request):**
```json
{
  "detail": "Invalid image format or corrupted file"
}
```

**Status Codes:**
- `200 OK` - Pattern displayed
- `400 Bad Request` - Invalid image format
- `413 Payload Too Large` - File size exceeds limit
- `422 Unprocessable Entity` - Missing required parameters
- `500 Internal Server Error` - Server error during display

---

### Get Current Pattern

#### GET `/pattern/current`

Retrieve information about the currently displayed pattern.

**Request:**
```bash
curl http://localhost:8000/pattern/current
```

**Response (200 OK):**
```json
{
  "filename": "pattern.png",
  "resolution": {
    "width": 1280,
    "height": 720
  },
  "color_mode": "grayscale",
  "display_time": "2024-01-06T10:30:45.123456Z",
  "status": "active"
}
```

**If no pattern is displayed:**
```json
{
  "filename": null,
  "status": "idle"
}
```

---

### List Available Patterns

#### GET `/patterns`

List all available pattern files in the image directory.

**Request:**
```bash
curl http://localhost:8000/patterns
```

**Response (200 OK):**
```json
{
  "patterns": [
    {
      "filename": "grating_p12_s4_idx00.png",
      "path": "dmd_fastapi_image/grating_p12_s4_idx00.png",
      "size_bytes": 245678,
      "resolution": {
        "width": 1280,
        "height": 720
      }
    },
    {
      "filename": "grating_p12_s4_idx01.png",
      "path": "dmd_fastapi_image/grating_p12_s4_idx01.png",
      "size_bytes": 242134,
      "resolution": {
        "width": 1280,
        "height": 720
      }
    }
  ],
  "total_count": 2
}
```

**Query Parameters:**
- `filter` (optional) - Filter by filename pattern (e.g., "grating")
- `limit` (optional) - Maximum number of results (default: 100)

---

### Display Pattern by Filename

#### POST `/pattern/display/{filename}`

Display a specific pattern from the available patterns directory.

**Request:**
```bash
curl -X POST http://localhost:8000/pattern/display/grating_p12_s4_idx00.png
```

**Path Parameters:**
- `filename` (required) - Exact filename of pattern to display

**Query Parameters:**
- `duration` (optional) - Display duration in milliseconds

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Pattern displayed successfully",
  "filename": "grating_p12_s4_idx00.png"
}
```

**Error Response (404 Not Found):**
```json
{
  "detail": "Pattern file not found: grating_p12_s4_idx00.png"
}
```

---

### Stop Display

#### POST `/pattern/stop`

Stop displaying the current pattern and return to idle state.

**Request:**
```bash
curl -X POST http://localhost:8000/pattern/stop
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Display stopped successfully"
}
```

---

## DMD Controller Endpoints

### Get DMD Status

#### GET `/dmd/status`

Get current status of the DMD controller.

**Request:**
```bash
curl http://localhost:8000/dmd/status
```

**Response (200 OK):**
```json
{
  "connected": true,
  "mode": "EXTERNALPRINT",
  "led_pwm": 1023,
  "temperature_celsius": 45.2,
  "error_count": 0,
  "last_command_timestamp": "2024-01-06T10:30:45.123456Z"
}
```

**Field Descriptions:**
- `connected` (bool) - I2C connection status with DLPC1438
- `mode` (string) - Current DMD operating mode (STANDBY, EXTERNALPRINT, etc.)
- `led_pwm` (int) - LED PWM intensity (0-1023)
- `temperature_celsius` (float) - DMD internal temperature
- `error_count` (int) - Number of errors since startup
- `last_command_timestamp` (string) - ISO 8601 timestamp of last command

---

### Configure DMD LED

#### POST `/dmd/led/configure`

Adjust DMD LED brightness.

**Request:**
```bash
curl -X POST http://localhost:8000/dmd/led/configure \
  -H "Content-Type: application/json" \
  -d '{"pwm_value": 800}'
```

**Request Body:**
```json
{
  "pwm_value": 800
}
```

**Parameters:**
- `pwm_value` (int, required) - LED intensity (0-1023, where 1023 is maximum)

**Response (200 OK):**
```json
{
  "success": true,
  "message": "LED configured successfully",
  "pwm_value": 800
}
```

**Error Response (400 Bad Request):**
```json
{
  "detail": "PWM value must be between 0 and 1023"
}
```

---

### Reset DMD

#### POST `/dmd/reset`

Perform a soft reset of the DMD controller.

**Request:**
```bash
curl -X POST http://localhost:8000/dmd/reset
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "DMD reset successfully",
  "timestamp": "2024-01-06T10:30:45.123456Z"
}
```

---

## Sequence/Batch Operations

### Display Sequence

#### POST `/sequence/display`

Display a sequence of patterns in order.

**Request:**
```bash
curl -X POST http://localhost:8000/sequence/display \
  -H "Content-Type: application/json" \
  -d '{
    "patterns": [
      "grating_p12_s4_idx00.png",
      "grating_p12_s4_idx01.png",
      "grating_p12_s4_idx02.png"
    ],
    "duration_per_pattern": 100,
    "loop_count": 1,
    "loop_delay": 500
  }'
```

**Request Body:**
```json
{
  "patterns": ["pattern1.png", "pattern2.png", "pattern3.png"],
  "duration_per_pattern": 100,
  "loop_count": 1,
  "loop_delay": 500
}
```

**Parameters:**
- `patterns` (array, required) - List of pattern filenames
- `duration_per_pattern` (int, required) - Milliseconds per pattern
- `loop_count` (int, optional) - Number of loops (default: 1)
- `loop_delay` (int, optional) - Delay between loops in milliseconds (default: 0)

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Sequence started successfully",
  "total_patterns": 3,
  "total_duration": 800
}
```

---

### Stop Sequence

#### POST `/sequence/stop`

Stop a running sequence and return to idle.

**Request:**
```bash
curl -X POST http://localhost:8000/sequence/stop
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Sequence stopped successfully"
}
```

---

## Image Processing Endpoints

### Convert to Grayscale

#### POST `/image/convert/grayscale`

Convert an uploaded RGB image to grayscale.

**Request:**
```bash
curl -X POST http://localhost:8000/image/convert/grayscale \
  -F "file=@image.png"
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Image converted successfully",
  "original_size": {
    "width": 1920,
    "height": 1080
  },
  "output_filename": "converted_20240106_103045.png"
}
```

---

### Resize Image

#### POST `/image/resize`

Resize an image to target dimensions.

**Request:**
```bash
curl -X POST http://localhost:8000/image/resize \
  -F "file=@image.png" \
  -F "width=1280" \
  -F "height=720"
```

**Request Parameters:**
- `file` (multipart/form-data, required) - Image file
- `width` (int, required) - Target width in pixels
- `height` (int, required) - Target height in pixels

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Image resized successfully",
  "original_size": {"width": 1920, "height": 1080},
  "new_size": {"width": 1280, "height": 720},
  "output_filename": "resized_20240106_103045.png"
}
```

---

## System Information Endpoints

### Get System Info

#### GET `/system/info`

Get comprehensive system information.

**Request:**
```bash
curl http://localhost:8000/system/info
```

**Response (200 OK):**
```json
{
  "hostname": "raspberrypi",
  "platform": "Linux",
  "python_version": "3.9.2",
  "api_version": "1.0.0",
  "uptime_seconds": 3645,
  "memory": {
    "total_mb": 512,
    "available_mb": 256,
    "percent_used": 50
  },
  "disk": {
    "total_gb": 16.0,
    "available_gb": 8.5,
    "percent_used": 47
  }
}
```

---

### Get Service Statistics

#### GET `/system/stats`

Get API server statistics and performance metrics.

**Request:**
```bash
curl http://localhost:8000/system/stats
```

**Response (200 OK):**
```json
{
  "total_requests": 234,
  "requests_per_minute": 12.5,
  "average_response_time_ms": 45.3,
  "uptime_seconds": 3645,
  "patterns_served": 45,
  "errors": 2
}
```

---

## Error Responses

### Error Response Format

All error responses follow this format:

```json
{
  "detail": "Error description",
  "error_code": "ERROR_CODE",
  "timestamp": "2024-01-06T10:30:45.123456Z"
}
```

### Common HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | OK - Request successful |
| 400 | Bad Request - Invalid parameters |
| 404 | Not Found - Resource not found |
| 413 | Payload Too Large - File too large |
| 422 | Unprocessable Entity - Validation error |
| 500 | Internal Server Error - Server error |
| 503 | Service Unavailable - Server unavailable |

---

## Authentication (if configured)

Some deployments may require API key authentication:

```bash
curl -H "X-API-Key: your-api-key" http://localhost:8000/pattern/current
```

Check server configuration for authentication requirements.

---

## WebSocket Connections (Real-time Updates)

### Connect to Status Stream

Real-time status updates via WebSocket:

```javascript
const ws = new WebSocket('ws://localhost:8000/ws/status');

ws.onmessage = function(event) {
    const status = JSON.parse(event.data);
    console.log('Pattern status:', status.current_pattern);
    console.log('DMD temperature:', status.dmd_temperature);
};

ws.onerror = function(error) {
    console.error('WebSocket error:', error);
};
```

**Message Format:**
```json
{
  "timestamp": "2024-01-06T10:30:45.123456Z",
  "current_pattern": "grating_p12_s4_idx00.png",
  "dmd_temperature": 45.2,
  "display_status": "active"
}
```

---

## Rate Limiting

The API implements rate limiting to prevent abuse:

- **Default limit:** 100 requests per minute per IP address
- **Burst limit:** 10 requests per second

Rate limit headers in responses:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704516645
```

---

## Pagination

List endpoints support pagination:

```bash
curl "http://localhost:8000/patterns?limit=20&offset=40"
```

**Query Parameters:**
- `limit` (int, optional) - Number of items per page (default: 20, max: 100)
- `offset` (int, optional) - Number of items to skip (default: 0)

**Response includes:**
```json
{
  "items": [...],
  "total": 245,
  "limit": 20,
  "offset": 40,
  "has_more": true
}
```

---

## Code Examples

### Python Client

```python
import requests
from pathlib import Path

BASE_URL = "http://localhost:8000"

# Display a pattern
with open("pattern.png", "rb") as f:
    files = {"file": f}
    response = requests.post(f"{BASE_URL}/pattern/display", files=files)
    print(response.json())

# Get current pattern
response = requests.get(f"{BASE_URL}/pattern/current")
print(f"Current pattern: {response.json()['filename']}")

# List available patterns
response = requests.get(f"{BASE_URL}/patterns")
patterns = response.json()["patterns"]
print(f"Available patterns: {len(patterns)}")

# Display sequence
sequence_data = {
    "patterns": ["pattern1.png", "pattern2.png"],
    "duration_per_pattern": 100,
    "loop_count": 5
}
response = requests.post(f"{BASE_URL}/sequence/display", json=sequence_data)
print(response.json())
```

### Bash/cURL Client

```bash
#!/bin/bash
API_URL="http://localhost:8000"

# Check server health
curl "${API_URL}/health"

# Display image
curl -X POST "${API_URL}/pattern/display" \
  -F "file=@pattern.png"

# Get current pattern info
curl "${API_URL}/pattern/current"

# List all patterns
curl "${API_URL}/patterns?limit=10"

# Display sequence
curl -X POST "${API_URL}/sequence/display" \
  -H "Content-Type: application/json" \
  -d '{
    "patterns": ["p1.png", "p2.png"],
    "duration_per_pattern": 100,
    "loop_count": 1
  }'
```

### JavaScript/Fetch Client

```javascript
const API_URL = "http://localhost:8000";

// Check server health
fetch(`${API_URL}/health`)
  .then(r => r.json())
  .then(data => console.log('Server status:', data.status));

// Display pattern
const formData = new FormData();
const fileInput = document.getElementById('pattern-file');
formData.append('file', fileInput.files[0]);

fetch(`${API_URL}/pattern/display`, {
  method: 'POST',
  body: formData
})
  .then(r => r.json())
  .then(data => console.log('Pattern displayed:', data.filename));

// Get current pattern
fetch(`${API_URL}/pattern/current`)
  .then(r => r.json())
  .then(data => console.log('Current:', data.filename));
```

---

## References

- FastAPI documentation: https://fastapi.tiangolo.com/
- OpenAPI/Swagger specification: https://swagger.io/
- Uvicorn server: https://www.uvicorn.org/

