# UVG CAMPUS CENTRAL WiFi - Setup Guide for Students

**Quick guide to connect your Arch Linux laptop to UVG's enterprise WiFi.**

---

## Before You Start

### Check what you're using now
```bash
systemctl is-active iwd
systemctl is-active NetworkManager
```

- If **iwd** is active → You need to switch (instructions below)
- If **NetworkManager** is active → You're ready!

---

## Installation Steps

### 1. Get the package

Your professor will provide:
- `arch-package/` folder with all files
- Or a direct `.pkg.tar.zst` package file

### 2. Install

**From the arch-package folder:**
```bash
cd arch-package
makepkg -si
```

**From a pre-built package:**
```bash
sudo pacman -U aruba-onboard-uvg-1.0.0-1-x86_64.pkg.tar.zst
```

### 3. Switch from iwd to NetworkManager (if needed)

**IMPORTANT**: iwd doesn't work with UVG's network. You must use NetworkManager.

```bash
# Stop and disable iwd
sudo systemctl disable --now iwd

# Enable and start NetworkManager
sudo systemctl enable --now NetworkManager

# Wait a few seconds
sleep 5

# Check wlan0 exists
nmcli device status
```

**If `wlan0` is missing**:
```bash
sudo systemctl enable --now wifi-interface
sudo systemctl restart NetworkManager
nmcli device status  # Should show wlan0 now
```

### 4. Connect to temporary WiFi

Before enrolling, connect to internet:
```bash
nmcli device wifi list
nmcli device wifi connect "UVG VISITAS"  # Or your home WiFi
```

Note: UVG VISITAS needs a regristration and a sponsor. Try to sponsor yourselves or ask a professor. Follow the instructions in the screen and verify your email to get a password.

### 5. Enroll your device

1. Go to the university enrollment portal (ask your professor for the link or read the readme)
2. Log in with your UVG credentials
3. Click the enrollment button/link
4. A window will open automatically (Aruba Onboard)
5. Follow the prompts
6. Wait for "Enrollment successful!" notification

**The system auto-configures everything** - you don't need to manually enter certificates or passwords!

### 6. Connect!

```bash
nmcli connection up "UVG CAMPUS CENTRAL"
```

Or use the WiFi icon in your system tray and select "UVG CAMPUS CENTRAL".

---

## Troubleshooting

### "wlan0 not found" after switching to NetworkManager

**Framework 13 laptop users:**
```bash
sudo systemctl enable --now wifi-interface
sudo systemctl restart NetworkManager
```

### Enrollment window doesn't open

Check the protocol handler is registered:
```bash
xdg-mime query default x-scheme-handler/arubadp
# Should show: aruba-onboard.desktop
```

If not:
```bash
xdg-mime default aruba-onboard.desktop x-scheme-handler/arubadp
```

### "Password capture failed" notification

The auto-configuration might have missed the password. Check manually:

1. Your certificates are in `~/.aruba-onboard/`
2. Look for the password in: `~/.aruba-onboard/ui.log`
3. Add it manually:
   ```bash
   nmcli connection modify "UVG CAMPUS CENTRAL" \
       802-1x.private-key-password "PASSWORD-FROM-LOG"
   ```

### Connection fails with authentication errors

Verify NetworkManager is using wpa_supplicant (not iwd):
```bash
ps aux | grep wpa_supplicant
# Should see wpa_supplicant running
```

If not:
```bash
sudo systemctl restart NetworkManager
```

### Still can't connect

Check logs:
```bash
journalctl -u NetworkManager -f
```

Look for EAP-TLS or authentication errors. Share these with your professor.

---

## Daily Use

### Auto-connect at university
The connection is saved. When you're on campus, just turn on WiFi:
```bash
nmcli radio wifi on
```

Or use your system tray WiFi icon.

### At home
Your home WiFi still works normally. NetworkManager manages all connections.

### Switching between networks
```bash
# List available networks
nmcli device wifi list

# Connect to different network
nmcli device wifi connect "NETWORK-NAME" password "PASSWORD"

# Back to UVG
nmcli connection up "UVG CAMPUS CENTRAL"
```

---

## Certificate Expiration

Your certificate expires in **1 year**. Before expiration:

1. Go back to the enrollment portal
2. Click the enrollment link again
3. Re-enroll (same process as initial setup)
4. Your connection will update automatically

---

## Going Back to iwd (if needed)

If you really want to switch back to iwd:

```bash
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl enable --now iwd

# Reconnect to WiFi with iwd
iwctl station wlan0 connect "YOUR-NETWORK"
```

**Note**: UVG CAMPUS CENTRAL won't work with iwd!

---

## Questions?

- **Package issues**: Contact your professor
- **Network problems at UVG**: Contact UVG IT support
- **How it works**: Read `UVG-CAMPUS-CENTRAL-documentation.md` in the arch-package folder

---

## Summary

1. Install package: `makepkg -si`
2. Switch to NetworkManager if using iwd
3. Enroll at university portal
4. Connect: `nmcli connection up "UVG CAMPUS CENTRAL"`
5. Done!

**The entire process takes < 5 minutes.**
