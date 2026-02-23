# Arch-CachyOS-Laptop-Performance-Power-Tuning-Guide
This is a Comprehensive Guide to Tuning your Arch Linux/CachyOS gaming laptop for optimal performance and usability. This was made with the Alienware M17R4 in mind, featuring an i7-10870h, 32GB DDR4, RTX 3080 16GB.
# ⚡ Arch / CachyOS Laptop Performance & Power Tuning Guide

![Arch Linux](https://img.shields.io/badge/Arch-Linux-blue?logo=arch-linux)
![CachyOS](https://img.shields.io/badge/CachyOS-Optimized-brightgreen)
![Intel](https://img.shields.io/badge/CPU-Intel-blue)
![NVIDIA](https://img.shields.io/badge/GPU-NVIDIA-76B900?logo=nvidia)
![License](https://img.shields.io/badge/License-MIT-yellow)

Advanced power & thermal optimization guide for Intel + NVIDIA laptops.

Designed for:
- Arch Linux
- CachyOS
- Hybrid GPU gaming laptops

---

# 📑 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Intel Power Limits by Profile](#intel-power-limits-by-profile)
- [CPU Undervolting](#cpu-undervolting)
- [NVIDIA Runtime Power Management](#nvidia-runtime-power-management)
- [GPU Idle Monitor](#gpu-idle-monitor)
- [Verification & Testing](#verification--testing)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)
- [License](#license)

---

# 🧠 Overview

This guide configures:

✔ Dynamic Intel PL1 / PL2 limits  
✔ Persistent CPU undervolting  
✔ NVIDIA runtime power management  
✔ Safe GPU idle monitoring  

Goal:
- Lower idle power draw  
- Better thermals  
- Reduced fan noise  
- Longer battery life  
- Controlled performance scaling  

⚠️ **Advanced users only. Improper values can cause instability.**

---

# 🚀 Features

| Feature | Benefit |
|----------|----------|
| Dynamic PL limits | Smart performance scaling |
| CPU Undervolt | Lower temps & power draw |
| NVIDIA Runtime PM | GPU sleeps when idle |
| Idle Monitor | Verify GPU suspend state |

---

# 🔋 Intel Power Limits by Profile

Automatically changes PL1/PL2 when switching power profiles.

## Create Script

```bash
sudo nano /usr/local/bin/set-pl-by-profile.sh
```

## Paste script (see full script below) Be sure to EDIT Power Limits according to YOUR HARDWARE
```bash
#!/bin/bash

RAPL="/sys/class/powercap/intel-rapl:0"

PROFILE=$(powerprofilesctl get)

set_limits() {
  echo "$1" > $RAPL/constraint_0_power_limit_uw
  echo "$2" > $RAPL/constraint_1_power_limit_uw
  echo "$3" > $RAPL/constraint_0_time_window_us
  echo "$4" > $RAPL/constraint_1_time_window_us
}

case "$PROFILE" in
  performance)
    # PL1 70W, PL2 110W, Tau 56s / 8s
    set_limits 70000000 110000000 56000000 8000000
    ;;
  balanced)
    # PL1 35W, PL2 60W, Tau 20s / 3s
    set_limits 35000000 60000000 20000000 3000000
    ;;
  power-saver)
    # PL1 28W, PL2 30W, Tau 16s / 2s
    set_limits 28000000 30000000 16000000 2000000
    ;;
esac
```
Make executable:

```bash
sudo chmod +x /usr/local/bin/set-pl-by-profile.sh
```

Create service:

```bash
sudo nano /etc/systemd/system/pl-profile.service
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now pl-profile.service
sudo systemctl enable --now pl-profile.path
```

---

# ❄️ CPU Undervolting

Install dependencies:

```bash
sudo pacman -S git python python-setuptools
```

Install undervolt:

```bash
git clone https://github.com/georgewhewell/undervolt.git
cd undervolt
sudo python setup.py install
```

Test:

```bash
sudo undervolt --read
```

Create persistent service:

```bash
sudo nano /etc/systemd/system/undervolt.service
```
## Paste script (see full script below) Be sure to EDIT Power Limits according to YOUR HARDWARE

Start with -50 for core/cache/gpu and move up in increments of 5-10mv and test stability by stressing the system
```ini
[Unit]
Description=Apply Intel CPU undervolt & power limits
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/undervolt --core -50 --cache -50 --gpu -50
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now undervolt.service
```

Reboot and verify.

---

# 🎮 NVIDIA Runtime Power Management - Fix GPU Suspend

## Create udev Rule File

```bash
sudo nano /etc/udev/rules.d/99-nvidia-runtimepm.rules
```

---

## Add This To The File

```bash
# Enable runtime PM for RTX 3080 Mobile
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{device}=="0x24dc", ATTR{power/control}="auto"
```

---

## Find Your Correct NVIDIA PCI Device ID

Run:

```bash
lspci -nn | grep -i nvidia
```

### Example Output

```text
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104M [GeForce RTX 3080 Mobile / Max-Q 8GB/16GB] [10de:24dc] (rev a1)

01:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
```

### Understanding The IDs

- `10de` → NVIDIA’s vendor ID  
- `24dc` → Your GPU’s device ID  

So your rule should contain:

```bash
ATTR{vendor}=="0x10de"
ATTR{device}=="0x24dc"
```

Replace `24dc` with your actual GPU device ID if different.

---

## What This Does

When this exact NVIDIA GPU appears on the PCI bus, it enables:

```bash
ATTR{power/control}="auto"
```

This tells the system to use **runtime power management** for the device.

---

## Reload udev Rules

```bash
sudo udevadm control --reload
sudo udevadm trigger
```

Reboot recommended after applying changes.

## 🖥 GPU Idle Monitor
This script monitors NVIDIA runtime power management **without waking the GPU**. This is useful (yet completely optional) as using programs like BTOP will call on the GPU and wake it up while it is running. you can also check without this by simply typing "sensors" in your terminal while nothing is running (close steam or any other GPU processes first) and the GPU temperature should in theory read 0 or single digits when suspended.

It safely reads:
- `runtime_status`
- `runtime_active_time`
- `runtime_suspended_time`

---

## Create Idle Monitor Script

```bash
nano ~/gpu_idle_monitor.sh
```

---

## Add This Script

```bash
#!/bin/bash

GPU="/sys/bus/pci/devices/0000:01:00.0/power"

echo "Monitoring NVIDIA RTX 3080 Mobile runtime PM (safe, does NOT wake GPU)"
echo "Press Ctrl+C to stop"
echo

while true; do
    STATUS=$(cat $GPU/runtime_status)
    ACTIVE_US=$(cat $GPU/runtime_active_time)
    SUSPENDED_US=$(cat $GPU/runtime_suspended_time)

    ACTIVE_SEC=$(echo "scale=2; $ACTIVE_US / 1000000" | bc)
    SUSPENDED_SEC=$(echo "scale=2; $SUSPENDED_US / 1000000" | bc)

    printf "Status: %-10s | Active: %6.2fs | Suspended: %6.2fs\r" \
        "$STATUS" "$ACTIVE_SEC" "$SUSPENDED_SEC"

    sleep 1
done
```

---

## Make Script Executable

```bash
chmod +x ~/gpu_idle_monitor.sh
```

---

## Run Monitor

```bash
~/gpu_idle_monitor.sh
```

Press **Ctrl+C** to stop monitoring.

---

## ⚠️ Important

If your GPU PCI address is different, check with:

```bash
lspci | grep -i nvidia
```

Replace:

```bash
0000:01:00.0
```

with your actual PCI device path if needed.


Reboot recommended.


---

# 🛠 Troubleshooting

### System Freezes After Undervolt
- Reduce undervolt value (e.g. -40 instead of -50)

### GPU Never Suspends
- Ensure no background apps use NVIDIA
- Check with:
  ```bash
  nvidia-smi
  ```

### PL Limits Not Applying
- Verify:
  ```bash
  powerprofilesctl get
  ```

---

# 📜 License

MIT License — Free to modify and distribute.

---

# ⚠️ Disclaimer

This project modifies low-level power management settings.  
You assume all risk for instability or hardware issues.
