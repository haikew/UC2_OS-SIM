# Troubleshooting Guide

Comprehensive error diagnosis and resolution guide for UC2 DMD system.

## Table of Contents

1. [Display Issues](#display-issues)
2. [Connection Problems](#connection-problems)
3. [Network and USB Setup](#network-and-usb-setup)
4. [Image Processing Errors](#image-processing-errors)
5. [Performance Issues](#performance-issues)
6. [Hardware Problems](#hardware-problems)
7. [Python/Dependency Errors](#pythondependency-errors)
8. [FastAPI Server Issues](#fastapi-server-issues)

---

## Display Issues

### Problem: No Image Output on DPI Display

**Symptoms:**
- Blank or unresponsive display connected to DPI interface
- No framebuffer activity detected

**Diagnosis:**
```bash
# Check DPI connector status
kmsprint

# Verify framebuffer
fbset -fb /dev/fb0

# Check display dimensions
cat /sys/class/graphics/fb0/virtual_size
```

**Solutions:**

1. **Verify DPI Configuration:**
   ```bash
   grep dtoverlay /boot/config.txt | grep dpi
   ```
   Expected output should contain `vc4-kms-dpi-generic`

2. **Check Kernel Command Line:**
   ```bash
   cat /boot/cmdline.txt
   ```
   Should contain `video=DPI-1:-32` at the end

3. **Reboot and Test:**
   ```bash
   sudo reboot
   # After boot, run:
   kmsprint
   ```

4. **Verify Display Cable:**
   - Ensure DPI ribbon cable is properly seated
   - Check pin alignment (look for red stripe indicating pin 1)
   - Test with different cable if available

### Problem: Distorted or Flickering Display

**Symptoms:**
- Image appears scrambled or has visual artifacts
- Display flickers or goes in and out

**Diagnosis:**
```bash
# Check pixel clock configuration
grep clock-frequency /boot/config.txt

# Monitor system temperature (may cause throttling)
vcgencmd measure_temp
```

**Solutions:**

1. **Verify Timing Parameters:**
   ```bash
   # In /boot/config.txt, confirm:
   # clock-frequency=74250000  (should be 74.25 MHz)
   ```

2. **Check for Timing Conflicts:**
   ```bash
   # Disable overlaps that might cause DPI issues
   # Comment out other overlays if present:
   # dtoverlay=vc4-kms-v3d
   ```

3. **Adjust Sync Parameters:**
   If flickering persists, try slightly different timing:
   ```ini
   dtparam=vfp=10,vsync=10
   ```

4. **Check System Temperature:**
   ```bash
   vcgencmd measure_temp
   ```
   If above 80°C, improve cooling before proceeding

### Problem: Resolution Mismatch

**Symptoms:**
- Framebuffer shows different resolution than expected (e.g., 1024x768 instead of 1280x720)

**Diagnosis:**
```bash
fbset -fb /dev/fb0
# Should show: geometry 1280 720 1280 720 32
```

**Solutions:**

1. **Force Resolution in Kernel Command Line:**
   ```bash
   # Edit /boot/cmdline.txt, add or modify:
   video=DPI-1:1280x720@60
   ```

2. **Set Framebuffer Size Explicitly:**
   ```bash
   # In /boot/config.txt, add:
   framebuffer_width=1280
   framebuffer_height=720
   framebuffer_depth=32
   ```

3. **Reboot and Verify:**
   ```bash
   sudo reboot
   fbset -fb /dev/fb0
   ```

---

## Connection Problems

### Problem: API Server Not Responding

**Symptoms:**
- Cannot reach `http://localhost:8000`
- Connection refused or timeout

**Diagnosis:**
```bash
# Check if FastAPI process is running
ps aux | grep dmdfastapi

# Check if port is listening
netstat -tlnp | grep 8000
# or
ss -tlnp | grep 8000

# Try connecting locally
curl http://localhost:8000/health
```

**Solutions:**

1. **Start the Server:**
   ```bash
   python3 src/dmdfastapi.py
   ```

2. **Check for Port Conflicts:**
   ```bash
   # If port 8000 is in use, find the process:
   lsof -i :8000
   # Kill the process if needed:
   kill -9 <PID>
   ```

3. **Verify Network Configuration:**
   ```bash
   # Check IP address
   ip addr show

   # Test from another device
   curl http://<raspberry_pi_ip>:8000/health
   ```

4. **Enable UFW if Blocked:**
   ```bash
   # If firewall is active, allow port 8000
   sudo ufw allow 8000/tcp
   ```

### Problem: I2C Communication Failure

**Symptoms:**
- Cannot communicate with DMD controller via I2C
- `i2cdetect` shows no devices

**Diagnosis:**
```bash
# Check I2C bus
i2cdetect -y 1
# or for custom bus:
i2cdetect -y 8

# Verify I2C kernel module is loaded
lsmod | grep i2c
```

**Solutions:**

1. **Enable I2C Interface:**
   ```bash
   sudo raspi-config
   # Interfacing Options → I2C → Yes
   ```

2. **Or Enable Manually:**
   ```bash
   # Edit /boot/config.txt
   sudo nano /boot/config.txt
   # Add or uncomment:
   dtparam=i2c_arm=on
   
   sudo reboot
   ```

3. **Check I2C Hardware:**
   ```bash
   # Verify I2C device file exists
   ls -la /dev/i2c*
   
   # Check GPIO pins for I2C (SDA, SCL)
   raspi-gpio get | grep -E "GPIO.*func"
   ```

4. **Verify Pull-up Resistors:**
   - I2C lines typically require 4.7k ohm pull-ups
   - Check DMD controller board schematic
   - Verify connections are secure

### Problem: Timeout Connecting to DMD Controller

**Symptoms:**
- `dmdfastapi.py` times out during initialization
- Error: "DLPC1438 initialization timeout"

**Diagnosis:**
```bash
# Check I2C device address
i2cdetect -y 1

# Verify I2C clock frequency
grep i2c_baudrate /boot/config.txt
```

**Solutions:**

1. **Verify Device Address:**
   ```bash
   # Check which I2C bus in code
   grep -r "SMBus" src/
   # Default is bus 1 (GPIO 2/3)
   ```

2. **Adjust I2C Baudrate:**
   ```bash
   # In /boot/config.txt:
   dtparam=i2c_arm_baudrate=100000
   # Reduce if communication is unstable
   ```

3. **Check Physical Connection:**
   - Verify all DMD control pins are connected
   - Check GND connection
   - Test with multi-meter if available

4. **Increase Timeout in Code:**
   ```python
   # Edit src/dmdfastapi.py
   # Find DLPC1438 initialization
   # Increase timeout parameter if needed
   ```

---

## Network and USB Setup

### Problem: Cannot Connect to Raspberry Pi via USB

**Symptoms:**
- USB Ethernet adapter not recognized on Windows
- Cannot SSH to Pi through USB
- No network adapter appears in Device Manager

**Diagnosis:**
```powershell
# Windows: Check network adapters
ipconfig /all

# Windows: Check device manager for unknown devices
devmgmt.msc
```

**Solutions:**

1. **Install or Update RNDIS Driver:**
   - Open Device Manager (`devmgmt.msc`)
   - Find unknown USB device (often under "Other devices")
   - Right-click → Update driver
   - Choose "Browse my computer"
   - Select "Let me pick from a list"
   - Choose "Network adapters"
   - Find "Microsoft" → "Remote NDIS Compatible Device"
   - Click Next to install

2. **Configure Pi for USB Ethernet:**
   ```bash
   # Edit /boot/config.txt, add:
   dtoverlay=dwc2
   
   # Edit /boot/cmdline.txt, append at end:
   modules_load=dwc2,g_ether
   
   # Create SSH file
   sudo touch /boot/ssh
   ```

3. **Set Static IP on Pi:**
   ```bash
   sudo nano /etc/dhcpcd.conf
   # Append:
   # interface usb0
   #   static ip_address=192.168.137.2/24
   #   static routers=192.168.137.1
   
   sudo systemctl restart dhcpcd
   ```

4. **Full Setup Guide:**
   - See [get_raspberry_pi_zero_2w_ip_through_usb.md](get_raspberry_pi_zero_2w_ip_through_usb.md)

### Problem: Windows ICS (Internet Connection Sharing) Issues

**Symptoms:**
- "Connection refused" when trying to SSH to Pi
- ICS service not running or not available
- Cannot share internet to USB Ethernet adapter

**Diagnosis:**
```powershell
# Check ICS service status
sc.exe qc sharedaccess
sc.exe queryex sharedaccess

# Check network adapters
ipconfig /all

# Check Windows components
DISM /Online /Cleanup-Image /CheckHealth
```

**Solutions:**

1. **Repair Windows Components:**
   Run PowerShell as Administrator:
   ```powershell
   DISM /Online /Cleanup-Image /RestoreHealth
   sfc /scannow
   ```

2. **Enable ICS on WiFi Adapter:**
   - Open Network Connections (`ncpa.cpl`)
   - Right-click Wi-Fi adapter
   - Select Properties
   - Go to Sharing tab
   - Check "Allow other network users…"
   - Select USB Ethernet/RNDIS Gadget from dropdown
   - Click OK

3. **Verify ICS Configuration:**
   ```powershell
   # USB adapter should be 192.168.137.1
   ipconfig
   
   # Pi should be 192.168.137.2
   ping 192.168.137.2
   ```

4. **Complete ICS Fix Guide:**
   - See [repair_ics.md](repair_ics.md) for detailed Windows troubleshooting

### Problem: SSH Won't Connect Over USB

**Symptoms:**
- `ssh: connect to host 192.168.137.2 port 22: Connection refused`
- Can see USB network adapter but can't reach Pi
- Ping fails to Pi

**Diagnosis:**
```bash
# From Pi, check USB interface
ifconfig usb0

# Check if SSH is running
sudo systemctl status ssh

# Verify SSH port is listening
sudo netstat -tulnp | grep ssh
```

**Solutions:**

1. **Verify SSH is Enabled:**
   ```bash
   # Create /boot/ssh file if missing
   sudo touch /boot/ssh
   
   # Start SSH service
   sudo systemctl start ssh
   sudo systemctl enable ssh
   ```

2. **Check Network Connectivity:**
   ```bash
   # From Pi, ping the host
   ping 192.168.137.1
   
   # Check route
   ip route
   ```

3. **Verify Static IP is Set:**
   ```bash
   # Check current IP on usb0
   ip addr show usb0
   # Should show: 192.168.137.2/24
   ```

4. **Test Connectivity from Windows:**
   ```powershell
   # Ping Pi
   ping 192.168.137.2
   
   # Try SSH with verbose output
   ssh -v pi@192.168.137.2
   ```

---

## Image Processing Errors

### Problem: OSSIM Reconstruction Fails to Load Images

**Symptoms:**
- `ossim_reconstruction_gui.py` cannot find or load image files
- Error: "File not found" or "Cannot open image"

**Diagnosis:**
```bash
# Check image directory exists
ls -la dmd_fastapi_image/

# Verify image format
file dmd_fastapi_image/*.png
```

**Solutions:**

1. **Verify Image Path:**
   ```bash
   # Ensure images are in correct directory:
   # dmd_fastapi_image/RGB/ for input images
   # dmd_fastapi_image/ for output images
   
   ls -la src/dmd_fastapi_image/
   ```

2. **Check Image Format:**
   ```bash
   # Images must be PNG or TIFF
   # Verify with:
   identify dmd_fastapi_image/*.png
   # or
   file *.png
   ```

3. **Convert Image Format if Needed:**
   ```bash
   # Use ImageMagick to convert
   convert input.jpg -depth 8 output.png
   ```

4. **Verify File Permissions:**
   ```bash
   chmod 644 dmd_fastapi_image/*
   ```

### Problem: Memory Error During Reconstruction

**Symptoms:**
- `MemoryError` or `OSError: cannot allocate memory`
- Process killed due to out-of-memory

**Diagnosis:**
```bash
# Check available memory
free -h

# Check image dimensions
identify dmd_fastapi_image/*.png

# Monitor memory usage during process
watch -n 1 free -h
```

**Solutions:**

1. **Reduce Image Size:**
   ```bash
   # Downscale images before processing
   convert large_image.png -scale 50% reduced_image.png
   ```

2. **Process Images in Batches:**
   ```bash
   # Modify reconstruction tool to process one image at a time
   # Instead of loading all into memory
   ```

3. **Increase Swap Memory:**
   ```bash
   # Create swap file if needed (slow but effective)
   sudo fallocate -l 2G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

4. **Free Up System Memory:**
   ```bash
   # Stop unnecessary services
   sudo systemctl stop bluetooth
   sudo systemctl disable bluetooth
   ```

### Problem: OpenCV (cv2) Import Error

**Symptoms:**
- `ImportError: No module named 'cv2'`
- `ModuleNotFoundError: No module named 'opencv'`

**Diagnosis:**
```bash
python3 -c "import cv2; print(cv2.__version__)"
```

**Solutions:**

1. **Install OpenCV:**
   ```bash
   pip install opencv-python==4.8.0.74
   ```

2. **Install from System Packages (Slower):**
   ```bash
   sudo apt install python3-opencv
   ```

3. **Verify Installation:**
   ```bash
   python3 -c "import cv2; print(cv2.__file__)"
   ```

---

## Performance Issues

### Problem: Slow Image Processing

**Symptoms:**
- Reconstruction takes unusually long time
- CPU usage at 100%

**Diagnosis:**
```bash
# Check system resources
top
# or
htop

# Monitor CPU temperature
vcgencmd measure_temp

# Check disk I/O
iostat -x 1
```

**Solutions:**

1. **Optimize Image Size:**
   ```python
   # Process smaller images or use ROI (Region of Interest)
   image = image[y1:y2, x1:x2]  # Crop to relevant area
   ```

2. **Enable Multi-threading:**
   ```python
   # Use parallel processing libraries
   from multiprocessing import Pool
   # Implement parallel reconstruction
   ```

3. **Use Compiled Extensions:**
   ```bash
   # Install NumPy with BLAS support for faster math
   pip install numpy --upgrade
   ```

4. **Monitor System Temperature:**
   ```bash
   vcgencmd measure_temp
   # Reduce clock speed if throttling:
   # arm_freq=900  # in /boot/config.txt
   ```

### Problem: FastAPI Server Responds Slowly

**Symptoms:**
- HTTP requests take long time to complete
- Intermittent timeouts

**Diagnosis:**
```bash
# Monitor system load
uptime

# Check if disk is bottleneck
iostat -x 1

# Monitor memory
free -h

# Check network
iftop  # if installed
```

**Solutions:**

1. **Optimize Uvicorn Settings:**
   ```bash
   python3 -m uvicorn src.dmdfastapi:app \
     --host 0.0.0.0 \
     --port 8000 \
     --workers 1 \
     --loop uvloop
   ```

2. **Cache Image Loading:**
   ```python
   # Implement caching for frequently accessed images
   from functools import lru_cache
   
   @lru_cache(maxsize=10)
   def load_image(filename):
       return Image.open(filename)
   ```

3. **Reduce Image Quality for Preview:**
   ```python
   # Load lower resolution versions for preview
   # Full resolution for final output
   ```

---

## Hardware Problems

### Problem: GPIO Pins Not Responding

**Symptoms:**
- Cannot set GPIO pin values
- Error: "gpio not recognized" or "gpio: Permission denied"

**Diagnosis:**
```bash
# Check GPIO status
raspi-gpio get

# Verify GPIO permissions
ls -la /dev/gpiomem

# Check if raspi-gpio is installed
which raspi-gpio
```

**Solutions:**

1. **Install/Update raspi-gpio:**
   ```bash
   sudo apt update
   sudo apt install -y raspi-gpio
   ```

2. **Fix Permissions:**
   ```bash
   # Ensure GPIO is accessible by user
   sudo usermod -aG gpio $USER
   # Log out and back in for changes to take effect
   ```

3. **Run Commands with sudo if Needed:**
   ```bash
   sudo raspi-gpio get
   sudo raspi-gpio set 5 op ov pd
   ```

4. **Test GPIO Pin Directly:**
   ```bash
   # Set GPIO 5 as output, value high (3.3V)
   sudo raspi-gpio set 5 op ov ph
   # Set GPIO 5 low (0V)
   sudo raspi-gpio set 5 op ov pd
   ```

### Problem: DMD Controller Not Responding

**Symptoms:**
- No LED activity on DMD controller
- Cannot communicate via I2C
- Projector does not turn on

**Diagnosis:**
```bash
# Check power supply voltage
vcgencmd measure_volts

# Monitor I2C with logic analyzer (if available)
# Check power to DMD board with multi-meter
```

**Solutions:**

1. **Verify Power Supply:**
   ```bash
   # Check Pi voltage
   vcgencmd measure_volts
   # Should be close to 5.0V
   
   # Check current draw
   # Use multi-meter or power monitor
   ```

2. **Check DMD Board Connections:**
   - Verify all power connections (GND, VCC)
   - Check I2C lines are connected correctly
   - Verify any pull-up resistors

3. **Reset DMD Controller:**
   ```bash
   # Power cycle the board
   # Or through GPIO reset pin if available
   ```

4. **Test with Logic Analyzer:**
   - Monitor I2C bus during communication
   - Check for proper START, STOP, ACK signals

---

## Python/Dependency Errors

### Problem: Module Import Errors

**Symptoms:**
- `ModuleNotFoundError: No module named 'X'`
- Various missing package errors

**Diagnosis:**
```bash
# List installed packages
pip list

# Check specific package
pip show numpy
```

**Solutions:**

1. **Install Missing Package:**
   ```bash
   pip install <package_name>
   ```

2. **Install from requirements.txt:**
   ```bash
   pip install -r requirements.txt
   ```

3. **Check Python Version:**
   ```bash
   python3 --version
   # Should be 3.9 or newer
   ```

4. **Use Correct Python Interpreter:**
   ```bash
   # If multiple Python versions installed
   python3.9 -m pip install <package>
   ```

### Problem: Version Conflicts

**Symptoms:**
- `ImportError: cannot import name 'X'`
- Version mismatch errors
- Incompatible API usage

**Diagnosis:**
```bash
# Check package versions
pip show numpy pillow opencv-python

# Check for conflicting versions
pip check
```

**Solutions:**

1. **Update All Packages:**
   ```bash
   pip install --upgrade pip setuptools wheel
   pip install -r requirements.txt --upgrade
   ```

2. **Use Specific Versions:**
   ```bash
   # If issues occur, use pinned versions from requirements.txt
   # Each package listed with ==version
   ```

3. **Create Fresh Virtual Environment:**
   ```bash
   deactivate
   rm -rf venv
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

---

## FastAPI Server Issues

### Problem: Server Crashes on Startup

**Symptoms:**
- Error on `python src/dmdfastapi.py`
- Traceback shows initialization failure

**Diagnosis:**
```bash
# Run with verbose output
python3 -u src/dmdfastapi.py

# Check error logs
tail -f /tmp/dmdfastapi.log  # if logging configured
```

**Solutions:**

1. **Check for Port Already in Use:**
   ```bash
   lsof -i :8000
   kill -9 <PID>
   ```

2. **Verify Dependencies:**
   ```bash
   python3 -c "import fastapi; import uvicorn"
   ```

3. **Check Configuration Files:**
   ```bash
   # Ensure dmd_fastapi_image/ directory exists
   mkdir -p src/dmd_fastapi_image
   ```

4. **Run with Debug Mode:**
   ```python
   # In dmdfastapi.py, add:
   import logging
   logging.basicConfig(level=logging.DEBUG)
   ```

### Problem: High Memory Usage by Server

**Symptoms:**
- Server process consumes increasing memory
- System becomes sluggish
- OutOfMemory errors

**Diagnosis:**
```bash
# Monitor memory usage
watch -n 1 'ps aux | grep dmdfastapi'

# Check for memory leaks
# Use memory_profiler if installed
```

**Solutions:**

1. **Implement Image Caching Limits:**
   ```python
   # Limit cache size in dmdfastapi.py
   from functools import lru_cache
   
   @lru_cache(maxsize=5)  # Keep only 5 images cached
   def load_image(filename):
       pass
   ```

2. **Clear Cache Periodically:**
   ```python
   # Add periodic cleanup
   import asyncio
   
   async def cleanup_old_files():
       # Remove old cached files
       pass
   ```

3. **Use Streaming for Large Images:**
   ```python
   # Stream large files instead of loading fully
   from fastapi.responses import FileResponse
   return FileResponse(filepath)
   ```

### Problem: Endpoint Returns 404 Not Found

**Symptoms:**
- Accessing `/pattern/display` returns 404
- Routes not defined properly

**Diagnosis:**
```bash
# Test health endpoint
curl http://localhost:8000/health

# List available routes
curl http://localhost:8000/openapi.json
```

**Solutions:**

1. **Verify Endpoint is Defined:**
   ```bash
   grep -n "@app.post\|@app.get" src/dmdfastapi.py
   ```

2. **Check Path Spelling:**
   ```bash
   # Ensure path in curl matches exactly
   # /pattern/display (case-sensitive)
   ```

3. **Check Request Method:**
   ```bash
   # POST vs GET - must match definition
   curl -X POST http://localhost:8000/pattern/display
   ```

---

## Contacting Support

If issues persist:

1. **Collect Diagnostic Information:**
   ```bash
   # System info
   uname -a
   
   # Python version
   python3 --version
   
   # Installed packages
   pip list > installed_packages.txt
   
   # Error logs
   dmesg | tail -50 > system_log.txt
   ```

2. **Check Documentation:**
   - Review Raspberry Pi Setup Guide: `RASPBERRY_PI_SETUP.md`
   - Check API Documentation: `API_DOCUMENTATION.md`

3. **Provide Clear Error Report:**
   - Exact error message
   - Steps to reproduce
   - Diagnostic output from above

