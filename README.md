# Raspberry Pi 5 Print Server Setup

## Project Overview

This project documents the setup of a Raspberry Pi 5 as a network print server using CUPS (Common Unix Printing System), allowing wireless printing from a MacBook to a Brother HL-L2320D laser printer and a Dymo LabelWriter 4XL label printer. This project also covers diagnosing a failing SD card, setting up Raspberry Pi Connect for remote access, and configuring CUPS as a shared print server.

**Hardware Used:**
- Raspberry Pi 5
- Brother HL-L2320D Laser Printer (USB)
- Dymo LabelWriter 4XL Label Printer (USB)
- MacBook Pro (client machine)

---

## Table of Contents

1. [Initial SD Card Failure & Recovery](#1-initial-sd-card-failure--recovery)
2. [Fresh OS Installation](#2-fresh-os-installation)
3. [Setting Up Raspberry Pi Connect](#3-setting-up-raspberry-pi-connect)
4. [Installing and Configuring CUPS](#4-installing-and-configuring-cups)
5. [Adding Printers to CUPS](#5-adding-printers-to-cups)
6. [Connecting Printers from MacBook](#6-connecting-printers-from-macbook)
7. [Troubleshooting](#7-troubleshooting)
8. [Key Lessons Learned](#8-key-lessons-learned)

---

## 1. Initial SD Card Failure & Recovery

### Problem
The Raspberry Pi was exhibiting several unusual symptoms:
- Booting to terminal instead of desktop despite configuration changes
- Settings not saving after reboot
- Raspberry Pi Connect not working
- DNS resolution failing
- Filesystem mounted as read-only

All of these issues turned out to share a single root cause — a failing SD card.

### Diagnosis
Running `dmesg | grep -i "error"` revealed the root cause:

```
mmc0: Card stuck being busy!
mmcblk0: recovery failed!
I/O error, dev mmcblk0, sector 1056768
EXT4-fs (mmcblk0p2): I/O error while writing superblock
```

The SD card was physically failing, causing the OS to mount the filesystem as read-only to prevent further corruption. Because nothing could be written to disk, no settings or configuration changes would persist across reboots.

### Solution
- Backed up any needed data from the SD card via another computer
- Replaced the SD card with a quality branded card (Samsung or SanDisk Endurance series recommended for Pi use)
- Flashed a fresh Raspberry Pi OS image using Raspberry Pi Imager

### How to Check SD Card Health on macOS
```bash
# Check for filesystem errors using Disk Utility
# Open Disk Utility → Select SD Card → Click First Aid

# Or install F3 to test for bad sectors
brew install f3
diskutil list  # find your SD card disk number
sudo f3probe --destructive --time-ops /dev/disk4  # replace disk4 with your disk
```

---

## 2. Fresh OS Installation

### Tools Used
- [Raspberry Pi Imager](https://www.raspberrypi.com/software/) on macOS

### Steps
1. Downloaded and installed Raspberry Pi Imager on MacBook
2. Selected **Raspberry Pi OS (64-bit)** as the OS
3. In the Imager settings (gear icon), pre-configured:
   - Hostname
   - SSH enabled
   - WiFi SSID and password
   - Username and password
4. Flashed to new SD card and inserted into Pi

> **Note:** To find your WiFi SSID on macOS, click the WiFi icon in the menu bar — the network with a checkmark is your current SSID. Enter it exactly as shown, it is case-sensitive.

---

## 3. Setting Up Raspberry Pi Connect

Raspberry Pi Connect allows remote access to the Pi (screen sharing and remote shell) from any browser on the local network. This was chosen over using a 3.5" display, as the display drivers conflicted with the Wayland compositor that Raspberry Pi Connect requires, making remote access impossible while the screen was in use.

### Problem
After accidentally deleting the Raspberry Pi Connect account, the device showed:
```
Signed in: yes
Subscribed to events: no
```

### Solution
Sign out and back in to refresh credentials:
```bash
rpi-connect signout
rpi-connect on
rpi-connect signin
```

### Verifying Setup
```bash
rpi-connect doctor
```

All checks should pass once:
- A fresh SD card is installed (DNS and filesystem working)
- No conflicting display drivers are present
- Internet connectivity is confirmed

---

## 4. Installing and Configuring CUPS

CUPS (Common Unix Printing System) turns the Raspberry Pi into a network print server accessible to any device on the local network.

### Installation
```bash
sudo apt update && sudo apt install cups -y
```

### Add User to CUPS Admin Group
```bash
sudo usermod -a -G lpadmin pi
```

### Enable Network Access
```bash
sudo cupsctl --remote-admin --remote-any --share-printers
sudo systemctl restart cups
```

### Install Printer Drivers
```bash
# Brother driver
sudo apt install printer-driver-brlaser -y

# Dymo driver
sudo apt install printer-driver-dymo -y

sudo systemctl restart cups
```

### Install Bonjour/AirPrint Broadcasting
This makes the printers automatically discoverable on the network by Macs and other devices:
```bash
sudo apt install avahi-daemon cups-browsed -y
sudo systemctl enable avahi-daemon
sudo systemctl start avahi-daemon
sudo systemctl restart cups
```

### Access CUPS Web Interface
From any browser on the local network, navigate to:
```
http://<your-pi-ip-address>:631
```

To find your Pi's IP address:
```bash
hostname -I
```

---

## 5. Adding Printers to CUPS

1. Open a browser and go to `http://<your-pi-ip-address>:631`
2. Go to **Administration** → **Add Printer**
3. Log in with your Pi username and password when prompted
4. Select the printer from the detected USB devices list
5. Select the correct driver:
   - **Brother HL-L2320D** → Manufacturer: Brother → Model: HL-L2320D (or HL-L2300D)
   - **Dymo 4XL** → Manufacturer: Dymo → Model: LabelWriter 4XL
6. Check **"Share This Printer"** for both
7. Click **Add Printer**

### Verify Printers Are Active
```bash
lpstat -p
```
Both printers should show as **enabled** and **accepting**. If not:
```bash
sudo cupsenable PrinterName
sudo cupsaccept PrinterName
```
<img width="1278" height="1183" alt="Screenshot 2026-03-05 at 6 59 20 PM" src="https://github.com/user-attachments/assets/28ca8939-3155-4acc-83e7-65ff25b6e987" />

---

## 6. Connecting Printers from MacBook

### Problem Encountered
Manually adding printers through **System Settings → Printers & Scanners** resulted in printers being assigned "Generic PostScript Printer" drivers, which caused print jobs to fail or print incorrectly.

### Solution — Use Print Center
Instead of adding through System Settings, use the macOS **Print Center** app:

1. Open **Spotlight** (Cmd + Space) and search for **Print Center**
2. The printers shared from the Raspberry Pi will appear automatically via Bonjour
3. **Right-click** each printer and select **"Remember this Printer"**
4. macOS automatically selects the correct driver and settings

This method correctly identifies the printer drivers and avoids the generic driver issue entirely.

### Install Mac Drivers (if needed)
- **Brother HL-L2320D:** Download from [support.brother.com](https://support.brother.com)
- **Dymo LabelWriter 4XL:** Download DYMO Connect for Desktop from [dymo.com/support](https://www.dymo.com/support)

---

## 7. Troubleshooting

### Print Jobs Stuck in Queue
```bash
# Check printer status
lpstat -p

# View active jobs
lpq -a

# Check error log in real time
sudo tail -f /var/log/cups/error_log
```

### "Unable to open raster file" Error
```bash
sudo apt install cups-filters -y
sudo systemctl restart cups
```

### Only One Printer Detected
**Problem:** When both printers were connected to the same type of USB port, only one was detected.

**Solution:** Connect one printer to the **USB 3.0 port** (blue) and the other to the **USB 2.0 port** (black) on the Raspberry Pi 5. This ensures each printer gets its own USB controller and is detected independently.

### Printers Not Showing on Mac
```bash
# Verify avahi is running
sudo systemctl status avahi-daemon

# Restart if needed
sudo systemctl restart avahi-daemon
sudo systemctl restart cups
```

### RPi Connect Not Working
- Ensure no conflicting display drivers are installed (3.5" SPI screen drivers are known to break Wayland)
- Verify internet and DNS are working: `ping google.com`
- Run `rpi-connect doctor` to identify specific failures

---

## 8. Key Lessons Learned

| Problem | Root Cause | Solution |
|---|---|---|
| Settings not saving / DNS failing / RPi Connect failing | Failing SD card | Replace SD card — this fixed everything at once |
| RPi Connect incompatible with 3.5" screen | Screen driver disabled Wayland compositor | Removed screen, used headless + RPi Connect instead |
| Only one printer detected | Both USBs on same controller | Use USB 3.0 and USB 2.0 ports separately |
| Wrong Mac driver assigned | Added via System Settings | Use Print Center → Remember Printer instead |
| Print jobs failing | Missing CUPS filters | Install cups-filters package |

---

## Skills Demonstrated

- Linux system administration and troubleshooting
- Hardware diagnostics (SD card failure detection via `dmesg`)
- Print server setup and management (CUPS/IPP)
- Driver installation and configuration
- Network discovery configuration (Bonjour/mDNS/AirPrint)
- Cross-platform networking (Linux → macOS)
- Remote access setup (Raspberry Pi Connect)
- Reading and interpreting system logs (`dmesg`, CUPS error log)

---

![IMG_0687](https://github.com/user-attachments/assets/6c4711c7-cd17-432b-aff4-91417e924e15)

