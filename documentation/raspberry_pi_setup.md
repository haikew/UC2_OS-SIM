# Raspberry Pi Hardware Configuration Guide

Complete setup instructions for configuring Raspberry Pi Zero 2W with DPI interface for DMD control.

## Prerequisites

- Raspberry Pi Zero 2W or compatible model
- Micro SD card (16GB recommended)
- Power supply (5V, 2A minimum)
- DPI display connector and cable
- Raspberry Pi Imager or compatible tool

## 1. SD Card Preparation

### Step 1: Format SD Card
Use Raspberry Pi Imager or your system's disk utility:

- **Windows**: Disk Manager
- **macOS**: Disk Utility  
- **Linux**: `gparted` or similar

### Step 2: Install Raspberry Pi OS
1. Download Raspberry Pi Imager
2. Select **Raspberry Pi OS (Legacy) Lite** (Bullseye)
3. Insert SD card and write image
4. Wait for verification to complete

### Step 3: Initial Boot
Insert SD card into Pi and power on. First boot may take 2-3 minutes.

> **Note**: Default credentials are `pi:raspberry` (if security is enabled)

---

## 2. System Configuration

### Update System Packages
```bash
sudo apt update
sudo apt upgrade -y
```

### Install Required Packages
```bash
sudo apt install -y \
    git \
    python3-pip \
    python3-dev \
    python3-smbus \
    i2c-tools
```

### Configure Python Virtual Environment (Optional)
```bash
python3 -m venv ~/uc2dmd_env
source ~/uc2dmd_env/bin/activate
```

---

## 3. DPI Interface Configuration

### Edit `/boot/config.txt`

Access the boot configuration:
```bash
sudo nano /boot/config.txt
```

**Find and modify the KMS line:**
```ini
# Original line (disable if present):
# dtoverlay=vc4-kms-v3d

# Add DPI configuration:
dtoverlay=vc4-kms-dpi-generic,hactive=1280,hfp=110,hsync=40,hbp=220
dtparam=vactive=720,vfp=5,vsync=5,vbp=20
dtparam=clock-frequency=74250000,rgb888,pixclk-invert
```

**Configuration Parameters:**
- `hactive=1280`: Horizontal active pixels
- `hfp=110`: Horizontal front porch
- `hsync=40`: Horizontal sync pulse width
- `hbp=220`: Horizontal back porch
- `vactive=720`: Vertical active pixels
- `vfp=5`: Vertical front porch
- `vsync=5`: Vertical sync pulse width
- `vbp=20`: Vertical back porch
- `clock-frequency=74250000`: Pixel clock frequency (74.25 MHz for 720p@60)

**Save and exit:** Press Ctrl+X, then Y, then Enter

### Reboot System
```bash
sudo reboot
```

---

## 4. Kernel Command Line Configuration

### Edit `/boot/cmdline.txt`

Access the kernel command line:
```bash
sudo nano /boot/cmdline.txt
```

**Append to the END of the line (do not create new line):**
```text
video=DPI-1:-32
```

**Example full line:**
```text
console=serial0,115200 console=tty1 root=PARTUUID=... video=DPI-1:-32
```

**Save and exit:** Press Ctrl+X, then Y, then Enter

### Reboot Again
```bash
sudo reboot
```

---

## 5. GPIO Initialization

### Edit `/etc/rc.local`

Access the system startup script:
```bash
sudo nano /etc/rc.local
```

**Add before `exit 0`:**
```bash
# Configure GPIO pins 4-19 as inputs
raspi-gpio set 4-19 ip
```

### Enable rc-local Service
```bash
sudo chmod +x /etc/rc.local
sudo systemctl enable rc-local
sudo systemctl status rc-local
```

---

## 6. I2C Configuration (for DMD Controller)

### Enable I2C
```bash
sudo raspi-config
```

Navigate to: **Interfacing Options** → **I2C** → **Yes**

Or manually enable in `/boot/config.txt`:
```ini
dtparam=i2c_arm=on
```

### Configure I2C Bus
If using custom I2C bus:
```bash
# Set custom I2C bus frequency (optional)
dtparam=i2c_arm_baudrate=100000
```

### Verify I2C Detection
```bash
i2cdetect -y 1  # Bus 1
i2cdetect -y 8  # Custom bus (if configured)
```

---

## 7. Verification Steps

### Check Framebuffer Configuration
```bash
fbset -fb /dev/fb0
```

**Expected output:**
```text
mode "1280x720"
    geometry 1280 720 1280 720 32
    timings 0 0 0 0 0 0 0
    accel true
    rgba 8/16,8/8,8/0,0/0
endmode
```

### Verify KMS/DPI Status
```bash
kmsprint
```

**Expected output should show:**
```text
Connector 1 (42) DPI-1 (connected)
  Encoder 1 (41) DPI
    Crtc 1 (71) 1280x720@...
```

### Check GPIO Configuration
```bash
raspi-gpio get
```

**Expected pins:**
- GPIO 4-19: `func=INPUT`
- GPIO 20-27: `alt=2 func=DPI_D*` (data lines)

**Example excerpt:**
```text
GPIO 4:  func=INPUT
GPIO 5:  func=INPUT
...
GPIO 19: func=INPUT
GPIO 20: alt=2 func=DPI_D16
GPIO 21: alt=2 func=DPI_D17
...
GPIO 27: alt=2 func=DPI_D23
```

### Test I2C Communication
```bash
i2cdetect -y 1
```

---

## 8. Clone and Setup Repository

### Clone UC2 DMD Repository
```bash
cd ~
git clone https://github.com/yourusername/uc2dmd.git
cd uc2dmd
```

### Install Python Dependencies
```bash
pip install -r requirements.txt
```

### Configure Auto-Start (Optional)

Create systemd service file:
```bash
sudo nano /etc/systemd/system/dmdfastapi.service
```

Add the following content:
```ini
[Unit]
Description=UC2 DMD FastAPI Server
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/uc2dmd
ExecStart=/usr/bin/python3 -m uvicorn src.dmdfastapi:app --host 0.0.0.0 --port 8000
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start service:
```bash
sudo systemctl enable dmdfastapi.service
sudo systemctl start dmdfastapi.service
sudo systemctl status dmdfastapi.service
```

---

## 9. Test FastAPI Server

### Start the Server
```bash
python3 src/dmdfastapi.py
```

Expected output:
```text
INFO:     Uvicorn running on http://0.0.0.0:8000
```

### Test Connection
From another terminal or device:
```bash
curl http://localhost:8000/health
```

Expected response:
```json
{"status": "healthy"}
```

### Stop the Server
Press `Ctrl+C`

---

## 10. Troubleshooting Initial Setup

### Issue: DPI not appearing in `kmsprint`
- Verify `/boot/config.txt` edits are correct
- Check for typos in configuration parameters
- Reboot system: `sudo reboot`

### Issue: GPIO pins not recognized
- Ensure `/etc/rc.local` is executable: `sudo chmod +x /etc/rc.local`
- Check if `raspi-gpio` is installed: `which raspi-gpio`
- Run manually to test: `raspi-gpio set 4-19 ip`

### Issue: I2C communication fails
- Enable I2C using `raspi-config`
- Check I2C device file exists: `ls -la /dev/i2c*`
- Verify device address: `i2cdetect -y 1`
- Check pull-up resistors on I2C lines

### Issue: Framebuffer configuration incorrect
- Verify DPI overlay is properly configured
- Check pixel clock frequency: should be 74250000 Hz
- Reboot after each configuration change

---

## References

- [Raspberry Pi DPI Documentation](https://www.raspberrypi.org/)
- [Texas Instruments DLPC1438 DMD Controller](https://www.ti.com/)
- Linux framebuffer documentation
- Kernel DPI display driver

---

## Backup Files

Configuration backups are available in this directory:
- `config_backup.txt` - Original `/boot/config.txt`
- `cmdline_backup.txt` - Original `/boot/cmdline.txt`

