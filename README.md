# Aruba Onboard for UVG

Automatic enrollment and configuration for **UVG CAMPUS CENTRAL** WiFi network (802.1X EAP-TLS).

### Quick Start (Arch Linux)

#### Step 1: Install the Package

```bash
makepkg -si
```

#### Step 2: Configure NetworkManager

```bash
sudo systemctl enable --now NetworkManager
```

#### Step 3: Connect to Internet

**Important**: You need an active internet connection for enrollment (use your phone's hotspot, home WiFi, or any available network).

#### Step 4: Enroll Your Device

1. **Visit the enrollment portal** (while connected to internet):
   ```
   https://uswest4.cloudguest.central.arubanetworks.com/portal/scope.cust-52e107d4ed9b11ec8b7e52e2ed9a47ed/cda-user-portal/passpoint
   ```

2. **Click "Yes, I already have HPE Aruba Networking Onboard"**
   - Ignore any warnings about "unsupported operating system"
   - The package you installed makes Arch Linux supported!

3. **Click "Install using HPE Aruba Networking Onboard app"**
   - This should automatically launch the enrollment app
   - A window will appear asking you to accept certificates
   - Click "Accept" or "Continue" when prompted
   - Wait for enrollment to complete

4. **Enrollment complete!**
   - You should see a success message
   - The app will close automatically

#### Step 5: Connect to UVG Network

```bash
nmcli connection up "UVG CAMPUS CENTRAL"
```

You should now be connected to the university WiFi!

### For Ubuntu Students

The original `.deb` file works directly:
```bash
sudo dpkg -i Aruba_Onboard_Installer.deb
sudo apt-get install -f  # Fix dependencies
```

Then visit the enrollment portal and click the link.

### For Other Distros

Use gemini-cli or claude-code or open-code or codex and have them read the AGENTS.md file, they should manage to fix the connection. Maybe.

... Godspeed.

---

## What This Package Does

### 1. Extracts and installs HPE Aruba Onboard tools
- Binaries installed to `/opt/aruba-onboard/`
- Desktop integration for `arubadp://` protocol

### 2. Automatic enrollment workflow
When you click an enrollment link from the university portal:
- Wrapper script starts `onboard-srv` (background D-Bus service)
- Launches `onboard-ui` (enrollment GUI)
- Monitors D-Bus to capture the PKCS#12 password
- Downloads certificates from Aruba CloudGuest
- Auto-configures NetworkManager with complete credentials
- **You don't need to manually configure anything**

### 3. NetworkManager integration
Creates a working WiFi connection profile:
- SSID: `UVG CAMPUS CENTRAL`
- EAP-TLS with certificates
- PKCS#12 password automatically configured
- Ready to connect immediately

### 4. Hardware compatibility fixes
Includes fixes for specific hardware configurations (e.g. Framework laptops) where interfaces might not initialize correctly with standard settings.

---

## Requirements

- **NetworkManager**: Required for EAP-TLS and connection management.
- **wpa_supplicant**: The backend for NetworkManager (usually default).
- **Qt5 libraries**: For the enrollment GUI.

*Note: This package is not compatible with `iwd` as the backend.*

---

## Troubleshooting

### Enrollment completes but password capture fails
The helper script monitors D-Bus for the PKCS#12 password. If capture fails:

1. Check `~/.aruba-onboard/ui.log` for enrollment logs
2. The password might be visible in the D-Bus logs: `/tmp/aruba-dbus-capture-*.log`
3. Manually add it to NetworkManager:
   ```bash
   nmcli connection modify "UVG CAMPUS CENTRAL" \
       802-1x.private-key-password "YOUR-PASSWORD-HERE"
   ```

### Connection fails / I use `iwd`
The university network requires NetworkManager with wpa_supplicant. `iwd` is not supported.

**If you are using `iwd`, switch to NetworkManager:**
```bash
sudo systemctl stop iwd
sudo systemctl disable iwd
sudo systemctl restart NetworkManager
```

Verify backend:
```bash
systemctl status NetworkManager
# Should NOT show iwd backend
```

### "wlan0 not found" (Framework Laptops)
If your `wlan0` interface is missing (common on Framework 13 laptops without `iwd`), use the included fix:

```bash
sudo systemctl enable --now wifi-interface
sudo systemctl restart NetworkManager
```

### Check connection status
```bash
nmcli connection show "UVG CAMPUS CENTRAL"
journalctl -u NetworkManager -f  # Live logs
```

### App doesn't launch when clicking "Install using HPE Aruba Networking Onboard app"

If clicking the button on the enrollment page doesn't automatically launch the app (due to browser settings or xdg configuration):

**Manual launch method:**

1. **Before clicking the button**, open your browser's JavaScript console:
   - **Firefox**: Press `F12`, click "Console" tab
   - **Chrome/Chromium**: Press `F12`, click "Console" tab
   - **Other browsers**: Right-click → "Inspect Element" → "Console" tab

2. **Click the "Install using HPE Aruba Networking Onboard app" button**

3. **In the console**, you'll see the enrollment link. It looks like:
   ```
   arubadp://uswest4.cloudguest.central.arubanetworks.com/...
   ```

4. **Copy the entire link** from the console

5. **Launch the app manually** from terminal:
   ```bash
   aruba-onboard "arubadp://uswest4.cloudguest.central.arubanetworks.com/..."
   ```
   (Replace with your actual copied link)

6. **Continue with enrollment** as normal - the app should open and complete enrollment

**Tip**: The link contains your enrollment token and is only valid for a limited time, so use it immediately after copying. It is one use only, even if the process fails. 

---

## Technical Details

### Why NetworkManager instead of iwd?

**iwd fails with UVG's network** (tested extensively):
- `EAP completed with eapFail`
- `4-Way handshake failed, reason: 23`
- Root cause: iwd's built-in TLS stack incompatible with Aruba RADIUS server

**wpa_supplicant (via NetworkManager) works**:
- Uses OpenSSL for TLS (more mature, compatible)
- Successfully negotiates with Aruba ClearPass
- Handles TLS 1.2/1.3 properly

See `UVG-CAMPUS-CENTRAL-documentation.md` for full technical analysis.

### Network specifications
- **802.1X**: WPA2/WPA3-Enterprise
- **EAP Method**: TLS (certificate-based)
- **TLS Versions**: 1.2 or 1.3 (`phase1-auth-flags: 1`)
- **Certificates**: ECDSA P-384, issued by Aruba CloudGuest
- **Validity**: 1 year (re-enrollment required annually)

### Certificate storage
- Location: `~/.aruba-onboard/`
- Files:
  - `UVG CAMPUS CENTRAL-ca.pem` - CA certificate
  - `UVG CAMPUS CENTRAL-user.pem` - Your device certificate
  - `UVG CAMPUS CENTRAL-key.p12` - Encrypted private key
  - `profiles.adpdata` - Enrollment metadata

---

## Files Installed

```
/opt/aruba-onboard/bin/
├── onboard-cli        # Command-line tool
├── onboard-srv        # D-Bus service for NetworkManager
└── onboard-ui         # Enrollment GUI

/usr/bin/
└── aruba-onboard      # Wrapper script (manages srv + ui)

/usr/lib/aruba-onboard/
└── enrollment-helper  # Auto-configures NetworkManager

/usr/share/applications/
└── aruba-onboard.desktop  # Desktop integration

/usr/lib/systemd/system/
└── wifi-interface.service  # Optional: Framework laptop fix
```

---

## For Package Maintainers

### Building from source
```bash
makepkg -s    # Build with dependency checking
makepkg -si   # Build and install
```

### Checksums
Update SHA256 checksums in PKGBUILD after modifying scripts:
```bash
sha256sum Aruba_Onboard_Installer.deb aruba-onboard-wrapper \
          aruba-enrollment-helper wifi-interface.service \
          aruba-onboard.desktop
```

### Testing
1. Test enrollment in a VM or test machine
2. Verify D-Bus password capture works
3. Confirm NetworkManager connection profile is created
4. Test actual network connection

### Known Issues
- **D-Bus password capture**: Requires dbus-monitor access (should work for normal users)
- **Framework laptop wlan0 issue**: Documented and fixed with wifi-interface.service
- **iwd conflict**: Intentional - package will refuse to install if iwd is present

---

## FAQ

**Q: Do I need to re-enroll every year?**
A: Yes, certificates expire after 1 year. Re-enrollment is automatic through the same portal link.

**Q: Will this work on other Aruba networks?**
A: Yes, any university or organization using HPE Aruba ClearPass + CloudGuest should work with this package.

**Q: Can I share my certificates with another device?**
A: No. Certificates are device-specific (tied to your machine-id). Each device needs separate enrollment.

**Q: What if I reinstall my OS?**
A: Backup `~/.aruba-onboard/` directory before reinstalling. The certificates may still work if you keep the same `/etc/machine-id`.

---

## Credits

- **Package**: Created for UVG students by Dennis
- **HPE Aruba**: Original onboard tools
- **Testing & Documentation**: Collaborative debugging effort

## License

This package wraps proprietary HPE Aruba Onboard software. The wrapper scripts and integration are provided as-is for educational use.

---

## Support

For issues specific to this package:
- Contact: dmaldana@uvg.edu.gt

For network connectivity issues at UVG:
- IT Support: lol

For Aruba Onboard tool issues:
- HPE Aruba Networks support
