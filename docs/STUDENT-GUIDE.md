# UVG CAMPUS CENTRAL WiFi - Setup Guide for Students

**Quick guide to connect your Arch Linux laptop to UVG's enterprise WiFi.**

---

## Requirements

- **NetworkManager**: This tool requires NetworkManager to handle the enterprise connection.
- **Root access**: You will need `sudo` privileges to install the package.

---

## Installation Steps

### 1. Install the package

Open your terminal in this folder and run:

```bash
makepkg -si
```

### 2. Connect to temporary WiFi

Before enrolling, connect to the internet:
```bash
nmcli device wifi list
nmcli device wifi connect "UVG VISITAS"  # Or your home WiFi
```

Note: UVG VISITAS needs registration and a sponsor. Try to sponsor yourself or ask a professor. Follow the instructions on the screen and verify your email to get a password.

### 3. Enroll your device

1. Go to the university enrollment portal (ask your professor for the link or read the readme)
2. Log in with your UVG credentials
3. Click the enrollment button/link
4. A window will open automatically (Aruba Onboard)
5. Follow the prompts
6. Wait for "Enrollment successful!" notification

**The system auto-configures everything** - you don't need to manually enter certificates or passwords!

### 4. Connect!

```bash
nmcli connection up "UVG CAMPUS CENTRAL"
```

Or use the WiFi icon in your system tray and select "UVG CAMPUS CENTRAL".

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

---

## Troubleshooting & FAQ

### I use `iwd` / The package conflicts with `iwd`
The university's network (EAP-TLS) is currently incompatible with `iwd`. You must switch to NetworkManager with `wpa_supplicant`.

**To switch to NetworkManager:**

```bash
# Stop and disable iwd
sudo systemctl disable --now iwd

# Enable and start NetworkManager
sudo systemctl enable --now NetworkManager

# Wait a few seconds then check if wlan0 exists
sleep 5
nmcli device status
```

### "wlan0 not found" (Framework Laptops)
Some laptops, like the Framework 13, might not automatically create the `wlan0` interface without `iwd`. This package includes a fix.

Run this command if your WiFi interface is missing:
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

---

## Questions?

- **Package issues**: Contact your professor
- **Network problems at UVG**: Contact UVG IT support
- **How it works**: Read `README.md` in the parent folder

---

## Summary

1. Install package: `makepkg -si`
2. Enroll at university portal
3. Connect: `nmcli connection up "UVG CAMPUS CENTRAL"`

**The entire process takes < 5 minutes.**
