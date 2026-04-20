# Smart Sprayer V2 — Autostart on Boot (Raspberry Pi)

> **Target hardware:** Raspberry Pi 4 (or 3B+) · LCD Touchscreen (DSI or HDMI) · WiFi  
> **OS:** Raspberry Pi OS with Desktop (Bullseye or Bookworm, 32-bit or 64-bit)

---

## Overview

This guide walks you through setting up `run_gui.py` to launch automatically on every boot, **after WiFi is connected**, using a **systemd service**. The app runs in fullscreen on the LCD touchscreen.

---

## Prerequisites Checklist

Before starting, confirm each item:

- [ ] Raspberry Pi OS **with Desktop** is installed and working
- [ ] The LCD touchscreen is connected and displays the desktop at boot
- [ ] **Autologin is enabled** (desktop boots directly without a password prompt)
- [ ] WiFi credentials are saved (`sudo raspi-config` → System → Wireless LAN)
- [ ] Python 3.9+ is installed (`python3 --version`)
- [ ] All Python packages in `requirements.txt` are installed (see Step 2)
- [ ] Firebase credentials are set up in `firebase_credentials.py`

---

## Step 1 — Enable Desktop Autologin

The GUI app requires the graphical desktop (X11/Wayland) to already be running, which only happens after autologin.

```bash
sudo raspi-config
```

Navigate to:

```
1 System Options  →  S5 Boot / Auto Login  →  B4 Desktop Autologin
```

Select **B4 Desktop Autologin** — this boots straight to the desktop as the `pi` user without a password prompt. Choose **Finish** and **reboot**.

> If your username is not `pi` (e.g. `sprayer`), replace every occurrence of `pi` in this guide with your actual username. Verify with `whoami`.

---

## Step 2 — Install Python Dependencies

SSH into the Pi (or open a terminal on the Pi) and run:

```bash
cd /home/pi/SmartSprayerV2
pip3 install -r requirements.txt
```

Verify key packages are present:

```bash
python3 -c "import customtkinter, PIL, firebase_admin; print('OK')"
```

If you see `OK`, all required libraries are installed.

---

## Step 3 — Confirm the App Runs Manually

Always test manually before setting up autostart. From a terminal **on the Pi desktop** (not SSH):

```bash
cd /home/pi/SmartSprayerV2
python3 run_gui.py
```

Confirm:
- The fullscreen splash screen appears
- The Welcome screen appears  
- Login works (`sprayer` / `1234`)
- The dashboard loads and displays sensor data

Press **Escape** or use the app's exit mechanism to close. If the app does not run correctly now, fix it before proceeding — the service will have the same problem.

---

## Step 4 — Find Your Python Interpreter Path

The service file needs the absolute path to Python:

```bash
which python3
```

Typical output: `/usr/bin/python3`

If you use a virtual environment (e.g. `venv`), activate it first and run `which python3` to get the venv path, e.g. `/home/pi/venv/bin/python3`.

---

## Step 5 — Determine Your Display Server (X11 or Wayland)

Run this command on the Pi desktop terminal:

```bash
echo $XDG_SESSION_TYPE
```

| Output    | Display server  | Notes                              |
|-----------|-----------------|------------------------------------|
| `x11`     | X11 (Xorg)      | Bullseye default, most Pi setups   |
| `wayland` | Wayland         | Bookworm default (Pi 4/5 only)     |

Keep this answer — you'll use it in Step 6.

---

## Step 6 — Create the systemd Service File

Create the service file using nano (or any editor):

```bash
sudo nano /etc/systemd/system/smart-sprayer.service
```

### Option A — X11 (most common)

Paste the following (edit paths if your username or app location differs):

```ini
[Unit]
Description=Smart Sprayer V2 GUI
# Wait for the desktop environment AND network before starting
After=graphical.target network-online.target
Wants=network-online.target graphical.target

[Service]
Type=simple
User=pi
Group=pi
# Point to the app directory
WorkingDirectory=/home/pi/SmartSprayerV2

# X11 display and auth — required for any GUI app
Environment=DISPLAY=:0
Environment=XAUTHORITY=/home/pi/.Xauthority

# Prevent the app from starting before the X server is accepting connections
ExecStartPre=/bin/sleep 5

# The main command — adjust python3 path if using a venv
ExecStart=/usr/bin/python3 /home/pi/SmartSprayerV2/run_gui.py

# Restart the app if it crashes
Restart=on-failure
RestartSec=10

# Log stdout/stderr to the systemd journal (view with: journalctl -u smart-sprayer)
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=graphical.target
```

### Option B — Wayland (Raspberry Pi OS Bookworm)

```ini
[Unit]
Description=Smart Sprayer V2 GUI
After=graphical.target network-online.target
Wants=network-online.target graphical.target

[Service]
Type=simple
User=pi
Group=pi
WorkingDirectory=/home/pi/SmartSprayerV2

# Wayland display variables
Environment=WAYLAND_DISPLAY=wayland-0
Environment=XDG_RUNTIME_DIR=/run/user/1000
Environment=DISPLAY=:0

ExecStartPre=/bin/sleep 5
ExecStart=/usr/bin/python3 /home/pi/SmartSprayerV2/run_gui.py

Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=graphical.target
```

Save and exit: **Ctrl+O → Enter → Ctrl+X**

> **App location:** The tutorial assumes the app is at `/home/pi/SmartSprayerV2/`. If you cloned the repository elsewhere (e.g., `/home/pi/Desktop/Smart-Sprayer/source/rpi/SmartSprayerV2/`), update `WorkingDirectory` and `ExecStart` accordingly.

---

## Step 7 — Copy App Files to the Pi

If you are developing on Windows and transferring files to the Pi, use `scp` or `rsync`. From your Windows machine:

```powershell
# Replace PI_IP with your Pi's IP address
scp -r "C:\Users\sajed\Desktop\PROJECTS\Smart-Sprayer\source\rpi\SmartSprayerV2\*" pi@PI_IP:/home/pi/SmartSprayerV2/
```

Or using `rsync` (preferred — only copies changed files):

```powershell
rsync -avz --exclude='__pycache__' --exclude='*.pyc' \
  "C:/Users/sajed/Desktop/PROJECTS/Smart-Sprayer/source/rpi/SmartSprayerV2/" \
  pi@PI_IP:/home/pi/SmartSprayerV2/
```

---

## Step 8 — Enable networkd-wait-online (WiFi Dependency)

The service waits for `network-online.target`. For this to work properly with WiFi, enable the network wait service:

```bash
sudo systemctl enable systemd-networkd-wait-online.service
```

Or, if your Pi uses `dhcpcd` (most common on Raspberry Pi OS):

```bash
sudo systemctl enable NetworkManager-wait-online.service
```

> To check which networking daemon is active: `systemctl is-active dhcpcd NetworkManager systemd-networkd 2>/dev/null`

---

## Step 9 — Enable and Start the Service

```bash
# Reload systemd to pick up the new file
sudo systemctl daemon-reload

# Enable the service (auto-start on every boot)
sudo systemctl enable smart-sprayer.service

# Start the service right now (without rebooting)
sudo systemctl start smart-sprayer.service
```

Check status immediately after starting:

```bash
sudo systemctl status smart-sprayer.service
```

You should see `Active: active (running)` and the GUI should appear on the touchscreen.

---

## Step 10 — Reboot and Verify

```bash
sudo reboot
```

Watch the touchscreen. Expected sequence:

1. Raspberry Pi bootloader (rainbow screen)
2. Raspberry Pi OS boot text
3. Desktop loads briefly
4. **Fullscreen splash screen** — "AUTOMATED SPRAYER SYSTEM" with loading bar
5. **Welcome screen** — "Mabuhay!" 
6. **Login screen** — sign in with `sprayer` / `1234`
7. **Main dashboard** — fullscreen control panel

---

## Step 11 — Touchscreen Calibration (if needed)

If touch input is inaccurate or inverted, calibrate the screen:

```bash
# Install calibration tool
sudo apt install xinput-calibrator -y

# Run the calibrator (run on the Pi desktop)
xinput_calibrator
```

Follow the on-screen prompts (tap the 4 crosshairs). Add the output to:

```bash
sudo nano /etc/X11/xorg.conf.d/99-calibration.conf
```

For screen rotation (e.g., portrait mode), add to `/boot/config.txt`:

```ini
# Rotate 90°
display_rotate=1

# Rotate 180°
display_rotate=2

# Rotate 270°
display_rotate=3
```

---

## Troubleshooting

### App does not appear on screen

```bash
# Check service logs
journalctl -u smart-sprayer.service -n 50 --no-pager

# Check if the service is active
systemctl status smart-sprayer.service
```

Common causes:
- **DISPLAY not set**: Make sure `Environment=DISPLAY=:0` is in the service file.
- **XAUTHORITY wrong**: Run `echo $XAUTHORITY` on the Pi desktop terminal to get the correct path.
- **Started too early**: Increase `ExecStartPre=/bin/sleep 5` to `10` or `15` seconds.

---

### App starts but crashes immediately

```bash
journalctl -u smart-sprayer.service -xe
```

Common causes:
- Missing Python packages → re-run `pip3 install -r requirements.txt`
- Firebase credentials missing or invalid → check `firebase_credentials.py`
- Wrong `WorkingDirectory` → relative imports fail if the working directory is wrong

---

### WiFi not connected when the app starts

The app needs WiFi for Firebase and weather APIs. If it starts before WiFi is up:

```bash
# Check which state network-online.target reaches
systemctl status network-online.target

# Increase the sleep delay in the service
sudo nano /etc/systemd/system/smart-sprayer.service
# Change:  ExecStartPre=/bin/sleep 5
# To:      ExecStartPre=/bin/sleep 15
```

Alternatively, add a WiFi wait loop to the ExecStartPre:

```ini
ExecStartPre=/bin/bash -c 'until ping -c1 8.8.8.8 &>/dev/null; do sleep 2; done'
```

---

### Touch does not work / wrong rotation

See **Step 11** above. Also check:

```bash
# List input devices
xinput list

# Test touch events
evtest /dev/input/event0   # replace event0 with your touchscreen device
```

---

### Disable or stop the service

```bash
# Stop immediately
sudo systemctl stop smart-sprayer.service

# Disable from autostart
sudo systemctl disable smart-sprayer.service
```

---

### View live logs while the app is running

```bash
journalctl -u smart-sprayer.service -f
```

---

## Quick Reference

| Task                      | Command                                               |
|---------------------------|-------------------------------------------------------|
| Start service             | `sudo systemctl start smart-sprayer.service`          |
| Stop service              | `sudo systemctl stop smart-sprayer.service`           |
| Restart service           | `sudo systemctl restart smart-sprayer.service`        |
| View status               | `sudo systemctl status smart-sprayer.service`         |
| View logs                 | `journalctl -u smart-sprayer.service -n 100`          |
| Follow logs live          | `journalctl -u smart-sprayer.service -f`              |
| Enable on boot            | `sudo systemctl enable smart-sprayer.service`         |
| Disable on boot           | `sudo systemctl disable smart-sprayer.service`        |
| Reload after editing file | `sudo systemctl daemon-reload`                        |
