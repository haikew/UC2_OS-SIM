# UC2 DMD Controller - FastAPI-Based Pattern Control System

A FastAPI-based DMD (Digital Micromirror Device) controller for structured illumination microscopy and projection applications running on Raspberry Pi with DPI interface.

## � Project Origin

This project is based on the hardware and methodology from [openMLA/photon-ultra-controller](https://github.com/openMLA/photon-ultra-controller/tree/main), adapted and enhanced for Structured Illumination Microscopy (SIM) applications. The controller has been customized to provide advanced pattern control and reconstruction capabilities specifically optimized for microscopy workflows.

## �📚 Documentation

**New users should start here:**
- **[Getting Started Guide](documentation/getting_started.md)** - Step-by-step tutorial (recommended for first-time setup)
- **[Documentation Index](documentation/index.md)** - Complete guide to all documentation

**Key References:**
- [Raspberry Pi Hardware Setup](documentation/raspberry_pi_setup.md) - DPI and GPIO configuration
- [Get Pi Zero 2W IP via USB](documentation/get_raspberry_pi_zero_2w_ip_through_usb.md) - USB Ethernet setup and IP discovery
- [Repair Windows ICS Service](documentation/repair_ics.md) - Fix Internet Connection Sharing issues
- [API Documentation](documentation/api_documentation.md) - Complete endpoint reference
- [Troubleshooting Guide](documentation/troubleshooting.md) - Error diagnosis and solutions

---

## Quick Start

### Prerequisites
- Raspberry Pi Zero 2W (Bullseye)
- TI DLPC1438 DMD Controller
- Board from photon-ultra-controller
- Python 3.9+

### Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/haikew/UC2_OS-SIM.git
   cd uc2dmd
   ```

2. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

3. **Run the FastAPI server:**
   ```bash
   python src/dmdfastapi.py
   ```

   The API will be available at `http://localhost:8000`
   
   > **First time?** See [Getting Started Guide](documentation/getting_started.md) for detailed setup instructions

## Project Structure

```
uc2dmd/
├── README.md                          # This file
├── requirements.txt                   # Python dependencies
├── documentation/                     # Configuration and troubleshooting guides
│   ├── getting_started.md             # Quick start tutorial
│   ├── raspberry_pi_setup.md          # Raspberry Pi configuration guide
│   ├── troubleshooting.md             # Error diagnosis and solutions
│   ├── api_documentation.md           # FastAPI endpoint reference
│   ├── get_raspberry_pi_zero_2w_ip_through_usb.md  # USB network setup
│   ├── repair_ics.md                  # Windows ICS troubleshooting
│   ├── cmdline_backup.txt             # Kernel command line backup
│   └── config_backup.txt              # Config.txt backup
├── src/                               # Main source code
│   ├── dmdfastapi.py                 # Auto-launching FastAPI server (MAIN ENTRY POINT)
│   ├── main.py                       # OpenMLA test code 
│   ├── dmd_fastapi_image/            # API image serving directory
│   └── UV_projector/
│       ├── controller.py             # DLPC1438 DMD controller interface
│       └── __init__.py
├── dmd_reconstruction_tools/          # Image reconstruction utilities
│   ├── OSSIM_simple.py               # Simple IOS reconstruction
│   ├── OSSIM_complex.py              # Advanced IOS reconstruction
│   ├── ossim_reconstruction_gui.py   # **Recommended starting tool**
│   ├── OSSIM_simple.py
│   └── fouriertransform.ipynb        # Fourier analysis notebook
└── patterns/                          # Pre-generated pattern files
    ├── binary_p12_s4/
    ├── binary_p24_s8/
    ├── sine_p6_s1/
    └── ...
```

## Core Components

### 1. DMD FastAPI Server (`dmdfastapi.py`)
**Automatically launches FastAPI application for pattern control**

- REST API for pattern image serving
- Health monitoring and status endpoints
- Graceful error recovery
- Compatible with DLPC1438 controller

Features:
- Image display control
- Pattern sequence management
- Real-time parameter adjustment
- Automatic startup at boot

### 2. Image Reconstruction Tools (`dmd_reconstruction_tools/`)

#### Getting Started (Recommended)
Start with the **GUI-based reconstruction tool**:
```bash
python dmd_reconstruction_tools/ossim_reconstruction_gui.py
```

This tool provides:
- Interactive image selection (drag & drop support)
- Real-time parameter adjustment
- Graphical preview of results
- Batch processing capability

#### Available Tools
- **OSSIM_simple.py** - Basic IOS (In-focus/Out-of-focus Sectioning) algorithm
- **OSSIM_complex.py** - Advanced reconstruction with preprocessing
- **fouriertransform.ipynb** - Fourier frequency analysis for validation

### 3. Pattern Generator (`grating_generator.ipynb`)

Interactive notebook for generating phase-shifted grating patterns:
- Binary and sinusoidal gratings
- Configurable period, shift, and phase offset
- Real-time preview
- Batch export to PNG format

## API Usage

### Display a Pattern
```bash
curl -X POST http://localhost:8000/pattern/display \
  -F "file=@pattern.png"
```

### Get Status
```bash
curl http://localhost:8000/health
```

## Hardware Configuration

### Raspberry Pi Setup
The system requires specific GPIO and DPI configurations. See [documentation/raspberry_pi_setup.md](documentation/raspberry_pi_setup.md) for:
- DPI interface configuration (720p@60 timing)
- GPIO initialization
- Kernel command line setup
- Framebuffer verification

### Pin Mapping
- GPIO 4-19: Input/control pins
- GPIO 20-27: DPI data lines (DPI_D16-DPI_D23)
- GPIO 5: PROJ_ON (DMD power control)
- GPIO 6: HOST_IRQ (DMD interrupt signal)

## Troubleshooting

For common issues and solutions, refer to [documentation/troubleshooting.md](documentation/troubleshooting.md):
- Connection problems
- Image display issues
- Performance optimization
- Debug logging

### DMD Refresh Stability Issues

If you experience unstable DMD refresh rates or display flickering during configuration:
1. Check the **reference configuration backups** in the `documentation/` folder:
   - [cmdline_backup.txt](documentation/cmdline_backup.txt) - Working kernel command line parameters
   - [config_backup.txt](documentation/config_backup.txt) - Verified Raspberry Pi config.txt settings
2. Compare your current `/boot/cmdline.txt` and `/boot/config.txt` with these known-good configurations
3. Pay special attention to the DPI timing parameters (720p@60Hz, 74.25MHz pixel clock)
4. See [raspberry_pi_setup.md](documentation/raspberry_pi_setup.md) for detailed configuration steps

## Development

### Project Structure Details
- **src/dmdfastapi.py** - Main FastAPI application (auto-started)
- **src/main.py** - Direct hardware control example
- **src/UV_projector/controller.py** - DLPC1438 hardware abstraction
- **dmd_reconstruction_tools/** - Scientific image processing utilities
- **documentation/** - Configuration and reference guides

### Testing the Reconstruction Pipeline
1. Run `ossim_reconstruction_gui.py`
2. Select three phase-shifted test images
3. Export results for analysis

