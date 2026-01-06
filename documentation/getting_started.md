# Getting Started Guide

Quick start instructions for UC2 DMD system setup and first use.

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Installation](#installation)
3. [Starting the FastAPI Server](#starting-the-fastapi-server)
4. [First Test](#first-test)
5. [Image Reconstruction Tutorial](#image-reconstruction-tutorial)
6. [Next Steps](#next-steps)

---

## System Requirements

### Hardware
- **Raspberry Pi Zero 2W** or compatible (Raspberry Pi 4/5 also supported)
- **TI DLPC1438 DMD Controller** board
- **Micro SD card** (16GB minimum recommended)
- **Power supply** (5V/2A for Pi Zero, 5V/3A for Pi 4)
- **DPI cable** assembly

### Software
- **Python 3.9** or newer
- **pip** (Python package manager)
- **git** (for cloning repository)

### Network
- Ethernet or WiFi connection to Raspberry Pi (optional, for headless setup)
- Direct USB cable (for initial setup)

---

## Installation

### Step 1: Prepare Raspberry Pi

Refer to [raspberry_pi_setup.md](raspberry_pi_setup.md) for complete hardware configuration:

1. Install Raspberry Pi OS (Bullseye Lite recommended)
2. Configure DPI interface
3. Set up GPIO pins
4. Enable I2C communication

**Time estimate:** 30-45 minutes

> **Headless Setup Tip:** If you don't have a monitor, see [get_raspberry_pi_zero_2w_ip_through_usb.md](get_raspberry_pi_zero_2w_ip_through_usb.md) for USB-based network access. Also check [repair_ics.md](repair_ics.md) if Windows ICS issues occur.

### Step 2: Clone Repository

```bash
# SSH into Raspberry Pi or use terminal
cd ~
git clone https://github.com/yourusername/uc2dmd.git
cd uc2dmd
```

### Step 3: Install Python Dependencies

**Option A: System-wide installation**
```bash
pip install -r requirements.txt
```

**Option B: Virtual environment (recommended)**
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Step 4: Verify Installation

```bash
# Test Python imports
python3 -c "import fastapi; import numpy; import cv2; print('All dependencies OK')"

# Check DMD connection
python3 -c "from src.UV_projector.controller import DLPC1438; print('DMD controller importable')"
```

---

## Starting the FastAPI Server

### Automatic Startup

The FastAPI server is the main application and auto-starts:

```bash
# Navigate to repository directory
cd ~/uc2dmd

# Start the server
python3 src/dmdfastapi.py
```

**Expected output:**
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Application startup complete
```

### Configure Auto-start on Boot (Optional)

Create a systemd service for automatic startup:

```bash
# Create service file
sudo nano /etc/systemd/system/dmdfastapi.service
```

**Add content:**
```ini
[Unit]
Description=UC2 DMD FastAPI Server
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/uc2dmd
ExecStart=/usr/bin/python3 /home/pi/uc2dmd/src/dmdfastapi.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Enable service:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable dmdfastapi.service
sudo systemctl start dmdfastapi.service
sudo systemctl status dmdfastapi.service
```

**Check service logs:**
```bash
sudo journalctl -u dmdfastapi.service -f
```

---

## First Test

### 1. Check Server Health

```bash
curl http://localhost:8000/health
```

**Expected response:**
```json
{"status": "healthy", "timestamp": "2024-01-06T10:30:45.123456Z"}
```

### 2. View API Documentation

Open browser and navigate to:
```
http://localhost:8000/docs
```

This opens interactive Swagger UI with all available endpoints.

### 3. Display Test Pattern

**Using curl:**
```bash
# First, check available patterns
curl http://localhost:8000/patterns

# Display a pattern
curl -X POST http://localhost:8000/pattern/display \
  -F "file=@test_pattern.png"
```

**Using Python:**
```python
import requests

response = requests.get("http://localhost:8000/patterns")
patterns = response.json()["patterns"]
print(f"Available patterns: {len(patterns)}")

# If patterns exist, display the first one
if patterns:
    filename = patterns[0]["filename"]
    response = requests.post(
        f"http://localhost:8000/pattern/display/{filename}"
    )
    print(response.json())
```

### 4. Verify DMD Output

Check if pattern appears on connected DPI display:

```bash
# Get current pattern info
curl http://localhost:8000/pattern/current

# Get DMD status
curl http://localhost:8000/dmd/status
```

---

## Image Reconstruction Tutorial

### Overview

The UC2 DMD system includes image reconstruction tools for structured illumination microscopy. The GUI-based tool is recommended for first-time users.

### Step 1: Prepare Test Images

Create or obtain three phase-shifted test images:

```bash
# Create sample test images
mkdir -p test_images
cd test_images

# Copy example images or create with grating_generator.ipynb
# Required: three PNG files with same resolution
# Example naming: phase_0.png, phase_1.png, phase_2.png
```

### Step 2: Run Reconstruction GUI

**Start the interactive reconstruction tool:**
```bash
python3 dmd_reconstruction_tools/ossim_reconstruction_gui.py
```

**First run:**
- A GUI window will open
- Click "Select File 1" button
- Navigate to and select first phase image (phase_0.png)
- Repeat for phases 2 and 3

**Expected window layout:**
```
┌─────────────────────────────────────┐
│     OSSIM Reconstruction Tool       │
├─────────────────────────────────────┤
│ [Select Image 1] [Select Image 2]   │
│ [Select Image 3]                    │
│                                     │
│ ┌──────────────────────────────┐   │
│ │   Preview                    │   │
│ │   (Image display)            │   │
│ └──────────────────────────────┘   │
│                                     │
│ Intensity Scale: [===========]      │
│ Smoothing: [====]                  │
│                                     │
│ [Reconstruct] [Export] [Exit]      │
└─────────────────────────────────────┘
```

### Step 3: Adjust Parameters

In the GUI, you can adjust:

- **Intensity Scale** - Brightness of reconstruction
- **Smoothing Factor** - Noise reduction amount
- **Threshold** - Minimum intensity to consider

The preview updates in real-time as you adjust parameters.

### Step 4: Export Results

1. Click "Export" button
2. Choose output location and filename
3. Select output format (PNG, TIFF, or HDF5)
4. Click "Save"

**Output file naming:**
- Automatic timestamp: `reconstruction_20240106_103045.png`
- Custom name: `my_reconstruction.png`

### Step 5: Advanced Options (Optional)

**For batch processing:**
```bash
# Process all images in directory
python3 dmd_reconstruction_tools/OSSIM_complex.py \
  --input_dir ./test_images \
  --output_dir ./reconstructed \
  --pattern_intensity 500
```

**For Fourier analysis validation:**
```bash
# Open analysis notebook
jupyter notebook dmd_reconstruction_tools/fouriertransform.ipynb
```

---

## Pattern Generation

### Using the Grating Generator

Generate custom phase-shifted grating patterns:

**Interactive notebook (recommended):**
```bash
jupyter notebook src/grating_generator.ipynb
```

**Features:**
- Binary and sinusoidal gratings
- Adjustable period and phase shift
- Real-time preview
- Batch PNG export

**Key Parameters:**
- `period` - Grating period in pixels (2-1024)
- `shift_pixels` - Pixel shift per image (1+)
- `num_images` - Number of phase steps to generate
- `gamma` - Tone curve adjustment (sine mode only)

**Quick example:**
```python
# In notebook cells
period = 12
shift_pixels = 4
num_images = 3
mode = 'sine'

imgs = run_generate_and_preview(period=period, num_images=num_images, 
                                shift_pixels=shift_pixels, mode=mode)
run_save(imgs)
```

Generated patterns saved to: `new_pattern/`

---

## Converting RGB Images to Grayscale

### Method 1: Using Conversion Notebook

```bash
jupyter notebook convert_images_to_grayscale.ipynb
```

This interactive notebook:
- Loads RGB images from `src/dmd_fastapi_image/RGB/`
- Converts to 8-bit grayscale
- Saves to `src/dmd_fastapi_image/`
- Verifies output

### Method 2: Using API

```bash
# Upload and convert RGB image
curl -X POST http://localhost:8000/image/convert/grayscale \
  -F "file=@color_image.jpg"
```

### Method 3: Python Script

```python
from PIL import Image

# Load RGB image
rgb_img = Image.open('color_image.jpg')

# Convert to grayscale
gray_img = rgb_img.convert('L')

# Save as PNG
gray_img.save('grayscale_image.png', 'PNG')

print(f"Saved grayscale image: {gray_img.size}")
```

---

## Troubleshooting First Setup

### Server Won't Start

**Error:** `Address already in use`
```bash
# Port 8000 is in use, find and kill the process
lsof -i :8000
kill -9 <PID>

# Or use different port
python3 -m uvicorn src.dmdfastapi:app --port 8001
```

**Error:** `ModuleNotFoundError`
```bash
# Reinstall dependencies
pip install -r requirements.txt --force-reinstall

# Or check Python version
python3 --version  # Should be 3.9+
```

### No Display Output

**Check steps:**
1. Verify DPI configuration: `cat /boot/config.txt | grep dpi`
2. Check GPIO status: `raspi-gpio get`
3. Test framebuffer: `fbset -fb /dev/fb0`

See [troubleshooting.md](troubleshooting.md) for detailed solutions.

### Reconstruction Fails

**Common issues:**
- Wrong image format (must be PNG or TIFF)
- Mismatched image dimensions
- Insufficient memory for large images

**Solution:**
```bash
# Convert images to correct format
convert input.jpg -depth 8 output.png

# Check image properties
identify image.png
# Should show: image.png PNG 1280x720 8-bit Gray

# Free up memory
sudo systemctl stop bluetooth
sudo systemctl disable bluetooth
```

---

## Directory Structure for First Use

```
uc2dmd/
├── src/
│   ├── dmdfastapi.py              # Main server (run this!)
│   ├── dmd_fastapi_image/         # Pattern directory
│   │   ├── RGB/                   # RGB input images
│   │   └── *.png                  # Grayscale patterns
│   └── UV_projector/
│       └── controller.py          # Hardware interface
│
├── dmd_reconstruction_tools/
│   ├── ossim_reconstruction_gui.py  # ← START HERE for reconstruction
│   ├── OSSIM_simple.py
│   └── OSSIM_complex.py
│
├── src/
│   ├── grating_generator.ipynb    # Generate patterns
│   └── convert_images_to_grayscale.ipynb
│
├── documentation/
│   ├── raspberry_pi_setup.md      # Hardware config
│   ├── troubleshooting.md         # Error solutions
│   ├── api_documentation.md       # API reference
│   └── config_backup.txt          # Config backups
│
└── README.md                      # Overview (you are here)
```

---

## Next Steps

### 1. **Verify Hardware** (if not done)
   - Follow [raspberry_pi_setup.md](raspberry_pi_setup.md)
   - Test GPIO and I2C communication
   - Verify DPI display works

### 2. **Test FastAPI Server**
   - Start `dmdfastapi.py`
   - Access `http://localhost:8000/docs`
   - Try displaying a test pattern

### 3. **Generate Test Patterns**
   - Use `grating_generator.ipynb`
   - Create binary and sine gratings
   - Save to `src/dmd_fastapi_image/`

### 4. **Test Image Reconstruction**
   - Run `ossim_reconstruction_gui.py`
   - Load three phase-shifted images
   - Export reconstructed result

### 5. **Integrate with Your Experiment**
   - Use FastAPI endpoints in your code
   - Or call server directly via HTTP
   - See api_documentation.md for examples

---

## Recommended Reading Order

1. **README.md** (main overview)
2. **getting_started.md** (this file)
3. **raspberry_pi_setup.md** (if you need hardware config)
4. **api_documentation.md** (for API usage)
5. **troubleshooting.md** (if issues occur)

---

## Quick Reference Commands

```bash
# Clone and setup
git clone https://github.com/yourusername/uc2dmd.git && cd uc2dmd
pip install -r requirements.txt

# Start FastAPI server
python3 src/dmdfastapi.py

# Test server
curl http://localhost:8000/health

# Open API docs
firefox http://localhost:8000/docs &  # or your browser

# Run reconstruction tool
python3 dmd_reconstruction_tools/ossim_reconstruction_gui.py

# Generate patterns
jupyter notebook src/grating_generator.ipynb

# Convert RGB to grayscale
jupyter notebook convert_images_to_grayscale.ipynb

# Check logs (if running as service)
sudo journalctl -u dmdfastapi.service -f
```

---

## Support and Resources

- **API Documentation**: http://localhost:8000/docs (when server running)
- **Hardware Setup**: See raspberry_pi_setup.md
- **Error Solutions**: See troubleshooting.md
- **Code Examples**: In api_documentation.md

---

## Success Indicators

You'll know everything is working when:

✅ FastAPI server starts without errors  
✅ `curl http://localhost:8000/health` returns healthy status  
✅ API documentation page loads at `/docs`  
✅ Pattern displays on DPI screen  
✅ Reconstruction GUI loads image files  
✅ Reconstructed images export successfully  

---

## Feedback and Improvements

Found an issue with this guide? Please report:
- Unclear instructions
- Missing steps
- Typos or errors
- Suggestions for improvement

