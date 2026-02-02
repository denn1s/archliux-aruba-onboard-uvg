# Context for AI Agents - UVG Aruba Onboard Package Debugging

## Overview

You are helping debug the `aruba-onboard-uvg` package for connecting Arch Linux laptops to **UVG CAMPUS CENTRAL** WiFi (802.1X EAP-TLS).

**Critical Context**: This WiFi network **only works with NetworkManager + wpa_supplicant**. It does NOT work with iwd (iwd's TLS stack fails with `eapFail` errors). The package enforces this.

---

## Quick Facts

- **Network**: UVG CAMPUS CENTRAL (802.1X WPA-Enterprise)
- **Auth**: EAP-TLS (certificate-based)
- **Enrollment**: HPE Aruba CloudGuest via `arubadp://` protocol links
- **Certificate Lifetime**: 1 year
- **Working Config**: NetworkManager + wpa_supplicant + OpenSSL TLS
- **Failing Config**: iwd (built-in TLS, consistently fails with reason 23)

---

## Testing Environment

**IMPORTANT**: The user is likely connected to a **phone hotspot** for internet access while debugging. Testing the UVG network will **disconnect them from you**.

### Safe Testing Pattern

```bash
# 1. Prepare everything while on hotspot
# (install package, configure, etc.)

# 2. Create a test script that:
#    - Attempts connection to UVG CAMPUS CENTRAL
#    - Logs results
#    - Auto-reconnects to hotspot after timeout

# 3. Run script, wait, then read logs

# Example test script:
cat > /tmp/uvg_test.sh << 'EOF'
#!/bin/bash
LOG="/tmp/uvg_test_$(date +%s).log"
echo "=== UVG Connection Test $(date) ===" > "$LOG"

# Try to connect
echo "Attempting connection..." | tee -a "$LOG"
timeout 30 nmcli connection up "UVG CAMPUS CENTRAL" >> "$LOG" 2>&1
RESULT=$?

echo "Exit code: $RESULT" | tee -a "$LOG"

# Test internet
if [ $RESULT -eq 0 ]; then
    echo "Testing internet access..." | tee -a "$LOG"
    timeout 10 ping -c 3 8.8.8.8 >> "$LOG" 2>&1
    PING_RESULT=$?
    echo "Ping exit code: $PING_RESULT" | tee -a "$LOG"
fi

# Capture NM logs
echo "=== NetworkManager logs ===" >> "$LOG"
journalctl -u NetworkManager --since "1 minute ago" -n 100 >> "$LOG" 2>&1

# Disconnect and return to hotspot
echo "Disconnecting from UVG..." | tee -a "$LOG"
nmcli connection down "UVG CAMPUS CENTRAL" >> "$LOG" 2>&1

# Reconnect to phone hotspot
echo "Reconnecting to hotspot..." | tee -a "$LOG"
nmcli device wifi connect "PHONE_HOTSPOT_NAME" password "HOTSPOT_PASSWORD" >> "$LOG" 2>&1

echo "=== Test complete ===" | tee -a "$LOG"
echo "Log saved to: $LOG"
EOF

chmod +x /tmp/uvg_test.sh

# 4. Run it
sudo /tmp/uvg_test.sh

# 5. Wait 45 seconds, then read the log
sleep 45
cat /tmp/uvg_test_*.log | tail -50
```

**Ask the user for their hotspot SSID/password** before creating test scripts.

---

## Installation Flow

### Expected Steps

1. **Install package**: `cd arch-package && makepkg -si`
2. **Switch to NetworkManager** (if using iwd):
   ```bash
   sudo systemctl disable --now iwd
   sudo systemctl enable --now NetworkManager
   ```
3. **Enable wifi-interface service** (Framework laptops):
   ```bash
   sudo systemctl enable --now wifi-interface
   sudo systemctl restart NetworkManager
   ```
4. **Enroll**: User clicks `arubadp://` link from university portal
5. **Connect**: `nmcli connection up "UVG CAMPUS CENTRAL"`

---

## Common Issues & Debugging

### Issue 1: wlan0 Interface Doesn't Exist

**Symptom**: `nmcli device status` doesn't show wlan0

**Cause**: Framework 13 laptop or similar Intel WiFi that requires manual interface creation

**Check**:
```bash
# Check if phy0 exists but wlan0 doesn't
ls /sys/class/ieee80211/  # Should show phy0
ip link show wlan0         # Should fail
```

**Fix**:
```bash
# Enable the wifi-interface service
sudo systemctl enable --now wifi-interface

# Wait a moment
sleep 2

# Restart NetworkManager
sudo systemctl restart NetworkManager

# Verify
nmcli device status | grep wlan0
```

**Verify the service**:
```bash
systemctl status wifi-interface
journalctl -u wifi-interface -n 20
```

---

### Issue 2: Password Capture Failed

**Symptom**: Enrollment completes but connection prompts for PKCS#12 password

**Cause**: `aruba-enrollment-helper` failed to capture password from D-Bus

**Debug**:
```bash
# Check if helper ran
ps aux | grep enrollment-helper

# Check for D-Bus capture log
ls -lh /tmp/aruba-dbus-capture-*.log

# Read the log
cat /tmp/aruba-dbus-capture-*.log | grep -A 5 -B 5 "private-key-password"

# Check enrollment logs
cat ~/.aruba-onboard/ui.log | tail -50
```

**Password might be visible in**:
- `/tmp/aruba-dbus-capture-*.log`
- `~/.aruba-onboard/ui.log`
- Not easily extractable (random 32-char alphanumeric)

**Workaround**:
```bash
# If you find the password, add it manually:
nmcli connection modify "UVG CAMPUS CENTRAL" \
    802-1x.private-key-password "THE-PASSWORD-HERE"
```

**Alternative - extract from PKCS12 directly**:
The password is randomly generated and NOT derivable. It must be captured during enrollment.

---

### Issue 3: Connection Profile Doesn't Exist

**Symptom**: `nmcli connection show "UVG CAMPUS CENTRAL"` fails

**Cause**: enrollment-helper didn't create the connection

**Check**:
```bash
# List connections
nmcli connection show

# Check if certificates exist
ls -lh ~/.aruba-onboard/

# Should have:
# - *-ca.pem
# - *-user.pem
# - *-key.p12
```

**Manual creation** (if helper failed):
```bash
# Extract info from certificate
CERT_FILE=$(ls ~/.aruba-onboard/*-user.pem)
CA_FILE=$(ls ~/.aruba-onboard/*-ca.pem)
KEY_FILE=$(ls ~/.aruba-onboard/*-key.p12)

SCOPE_ID=$(openssl x509 -in "$CERT_FILE" -noout -subject | grep -oP 'O\s*=\s*\K[^,]+')
IDENTITY="anonymous@${SCOPE_ID}.cloudauth.net"

echo "Identity: $IDENTITY"
echo "Certificates: $CA_FILE, $CERT_FILE, $KEY_FILE"

# Create connection (without password - will prompt)
nmcli connection add \
    type wifi \
    con-name "UVG CAMPUS CENTRAL" \
    ifname wlan0 \
    ssid "UVG CAMPUS CENTRAL" \
    wifi-sec.key-mgmt wpa-eap \
    802-1x.eap tls \
    802-1x.identity "$IDENTITY" \
    802-1x.ca-cert "$CA_FILE" \
    802-1x.client-cert "$CERT_FILE" \
    802-1x.private-key "$KEY_FILE" \
    802-1x.phase1-auth-flags 1 \
    connection.autoconnect no
```

---

### Issue 4: Enrollment UI Doesn't Open

**Symptom**: Clicking `arubadp://` link does nothing

**Check**:
```bash
# Check protocol handler registration
xdg-mime query default x-scheme-handler/arubadp
# Should show: aruba-onboard.desktop

# Check desktop file exists
ls -lh /usr/share/applications/aruba-onboard.desktop

# Check wrapper script exists
ls -lh /usr/bin/aruba-onboard
```

**Manual registration**:
```bash
xdg-mime default aruba-onboard.desktop x-scheme-handler/arubadp
update-desktop-database
```

**Manual enrollment** (bypass protocol handler):
```bash
# Start wrapper directly with a test URL (ask user for their enrollment URL)
/usr/bin/aruba-onboard "arubadp:/provision?data=ENROLLMENT-DATA-HERE"
```

---

### Issue 5: EAP-TLS Authentication Fails

**Symptom**: Connection attempt fails with authentication errors

**Critical Check - Verify NOT using iwd**:
```bash
# iwd must NOT be running
systemctl is-active iwd
# Should be: inactive

# NetworkManager MUST be running
systemctl is-active NetworkManager
# Should be: active

# wpa_supplicant should be running under NetworkManager
ps aux | grep wpa_supplicant
# Should show wpa_supplicant process
```

**If iwd is running**: This is the problem. iwd's TLS fails with UVG.
```bash
sudo systemctl stop iwd
sudo systemctl disable iwd
sudo systemctl restart NetworkManager
```

**Check connection parameters**:
```bash
nmcli connection show "UVG CAMPUS CENTRAL" | grep 802-1x
```

**Expected**:
```
802-1x.eap:                             tls
802-1x.identity:                        anonymous@52e107d4ed9b11ec8b7e52e2ed9a47ed.cloudauth.net
802-1x.ca-cert:                         /home/USER/.aruba-onboard/UVG CAMPUS CENTRAL-ca.pem
802-1x.client-cert:                     /home/USER/.aruba-onboard/UVG CAMPUS CENTRAL-user.pem
802-1x.private-key:                     /home/USER/.aruba-onboard/UVG CAMPUS CENTRAL-key.p12
802-1x.private-key-password:            <hidden>
802-1x.phase1-auth-flags:               1
```

**Check certificate validity**:
```bash
CERT=$(ls ~/.aruba-onboard/*-user.pem)
openssl x509 -in "$CERT" -noout -dates -subject

# Should show:
# notBefore: Jan 30 00:06:26 2026 GMT (or recent date)
# notAfter: Jan 30 00:06:26 2027 GMT (1 year later)
# subject: ... CN=username@uvg.edu.gt
```

**Live logs during connection**:
```bash
# In one terminal, tail logs
journalctl -u NetworkManager -f

# In another, try connecting
nmcli connection up "UVG CAMPUS CENTRAL"
```

**Look for in logs**:
- `EAP-TLS` or `EAP authentication` messages
- `Certificate` or `TLS` errors
- `successfully activated` (success)
- `activation failed` (failure)

---

### Issue 6: makepkg Fails

**Symptom**: Package build errors during `makepkg -si`

**Common causes**:

**Missing dependencies**:
```bash
# Install base-devel if not present
sudo pacman -S --needed base-devel

# Check specific dependencies
pacman -Q networkmanager wpa_supplicant qt5-base qt5-svg openssl iw
```

**Checksum mismatch** (if scripts were edited):
```bash
# Regenerate checksums
cd arch-package
sha256sum Aruba_Onboard_Installer.deb aruba-onboard-wrapper \
          aruba-enrollment-helper wifi-interface.service \
          aruba-onboard.desktop

# Update PKGBUILD sha256sums=() array
```

**Conflicts with iwd**:
```bash
# Remove iwd first if package conflicts
sudo pacman -R iwd
```

---

## Diagnostic Commands

### Full System Check
```bash
cat > /tmp/diagnostic.sh << 'EOF'
#!/bin/bash
echo "=== System Diagnostic for UVG WiFi ==="
echo ""

echo "1. Network Daemons:"
echo -n "  iwd: "; systemctl is-active iwd 2>&1
echo -n "  NetworkManager: "; systemctl is-active NetworkManager 2>&1
echo -n "  wpa_supplicant: "; pgrep -a wpa_supplicant > /dev/null && echo "running" || echo "not running"
echo ""

echo "2. WiFi Hardware:"
echo -n "  phy0 exists: "; [ -d /sys/class/ieee80211/phy0 ] && echo "yes" || echo "no"
echo -n "  wlan0 exists: "; ip link show wlan0 >/dev/null 2>&1 && echo "yes" || echo "no"
echo ""

echo "3. Aruba Package:"
echo -n "  Installed: "; pacman -Q aruba-onboard-uvg 2>&1 || echo "not installed"
echo -n "  Wrapper: "; [ -f /usr/bin/aruba-onboard ] && echo "present" || echo "missing"
echo -n "  Helper: "; [ -f /usr/lib/aruba-onboard/enrollment-helper ] && echo "present" || echo "missing"
echo ""

echo "4. Protocol Handler:"
xdg-mime query default x-scheme-handler/arubadp 2>&1
echo ""

echo "5. Certificates:"
if [ -d ~/.aruba-onboard/ ]; then
    ls -lh ~/.aruba-onboard/*.pem ~/.aruba-onboard/*.p12 2>/dev/null | awk '{print "  "$9" ("$5")"}'
else
    echo "  ~/.aruba-onboard/ not found (not enrolled)"
fi
echo ""

echo "6. NetworkManager Connections:"
nmcli connection show | grep -E "NAME|UVG|uvg"
echo ""

echo "7. Network Status:"
nmcli device status
echo ""

echo "=== End Diagnostic ==="
EOF

chmod +x /tmp/diagnostic.sh
bash /tmp/diagnostic.sh
```

Run this script and share output with the user to identify issues.

---

## Testing Scripts

### Test 1: Basic Connectivity (Safe - No Hotspot Disconnect)

```bash
# Check if UVG network is visible
nmcli device wifi list | grep -i "UVG CAMPUS CENTRAL"
```

If visible, proceed to connection test.

### Test 2: Connection Attempt (Disconnects from Hotspot)

**Ask user first**: "I need to test the UVG connection. This will disconnect you from the hotspot for ~30 seconds. Ready? Also, what's your hotspot name and password so I can reconnect?"

```bash
# Create reconnect script with user's hotspot info
cat > /tmp/reconnect_hotspot.sh << 'EOF'
#!/bin/bash
nmcli device wifi connect "USER_HOTSPOT_NAME" password "USER_PASSWORD"
EOF

chmod +x /tmp/reconnect_hotspot.sh

# Create test script
cat > /tmp/uvg_connection_test.sh << 'EOF'
#!/bin/bash
LOG="/tmp/uvg_test_$(date +%s).log"
exec 1> >(tee -a "$LOG")
exec 2>&1

echo "=== UVG Connection Test $(date) ==="

# Pre-test: Check connection exists
if ! nmcli connection show "UVG CAMPUS CENTRAL" >/dev/null 2>&1; then
    echo "ERROR: UVG CAMPUS CENTRAL connection not found"
    nmcli connection show
    exit 1
fi

# Attempt connection
echo "Attempting connection to UVG CAMPUS CENTRAL..."
timeout 30 nmcli connection up "UVG CAMPUS CENTRAL"
CONNECT_RESULT=$?

echo "Connection exit code: $CONNECT_RESULT"

if [ $CONNECT_RESULT -eq 0 ]; then
    echo "Connection succeeded!"

    # Test internet
    echo "Testing internet access..."
    if timeout 10 ping -c 3 8.8.8.8; then
        echo "Internet works!"
    else
        echo "Connection established but no internet"
    fi

    # Check IP
    echo "IP address:"
    ip addr show wlan0 | grep "inet "
else
    echo "Connection failed!"
fi

# Capture logs
echo ""
echo "=== NetworkManager logs (last 50 lines) ==="
journalctl -u NetworkManager --since "2 minutes ago" -n 50

# Disconnect
echo ""
echo "Disconnecting from UVG..."
nmcli connection down "UVG CAMPUS CENTRAL"

# Wait a moment
sleep 2

# Reconnect to hotspot
echo "Reconnecting to hotspot..."
/tmp/reconnect_hotspot.sh

echo ""
echo "=== Test Complete ==="
echo "Log saved to: $LOG"
EOF

chmod +x /tmp/uvg_connection_test.sh

# Run it
echo "Running test in 3 seconds... (Ctrl+C to cancel)"
sleep 3
sudo /tmp/uvg_connection_test.sh

# Wait for completion
sleep 45

# Show results
echo ""
echo "=== Test Results ==="
cat /tmp/uvg_test_*.log | tail -100
```

---

## Key Files to Check

### 1. Certificate Files
```bash
ls -lh ~/.aruba-onboard/
# Expected:
# - UVG CAMPUS CENTRAL-ca.pem (~895 bytes)
# - UVG CAMPUS CENTRAL-user.pem (~1261 bytes)
# - UVG CAMPUS CENTRAL-key.p12 (~2276 bytes)
```

### 2. NetworkManager Connection
```bash
sudo cat /etc/NetworkManager/system-connections/UVG\ CAMPUS\ CENTRAL.nmconnection
# Should contain all 802-1x parameters
```

### 3. Enrollment Logs
```bash
cat ~/.aruba-onboard/ui.log | tail -50
# Look for:
# - "status": 70 (enrollment progressed)
# - Certificate download messages
# - Errors
```

### 4. D-Bus Capture Logs
```bash
ls -lh /tmp/aruba-dbus-capture-*.log
cat /tmp/aruba-dbus-capture-*.log | grep "private-key-password"
```

---

## Known Working Configuration

From successful deployments:

```yaml
OS: Arch Linux (kernel 6.18+)
NetworkManager: 1.54.3+
wpa_supplicant: 2.11+
WiFi Card: Intel iwlwifi (Framework 13in tested)

Connection Profile:
  type: wifi
  ssid: UVG CAMPUS CENTRAL
  wifi-sec.key-mgmt: wpa-eap
  802-1x.eap: tls
  802-1x.identity: anonymous@52e107d4ed9b11ec8b7e52e2ed9a47ed.cloudauth.net
  802-1x.ca-cert: <path to CA PEM>
  802-1x.client-cert: <path to user PEM>
  802-1x.private-key: <path to PKCS12>
  802-1x.private-key-password: <32-char random string>
  802-1x.phase1-auth-flags: 1
```

---

## Critical Reminders

1. **iwd DOES NOT WORK** - If iwd is running, connection will fail with `eapFail` errors. Must use NetworkManager + wpa_supplicant.

2. **Testing disrupts connectivity** - Always prepare test scripts that auto-reconnect to hotspot.

3. **Password is random** - The PKCS#12 password is randomly generated during enrollment. It cannot be guessed or derived. It MUST be captured from D-Bus or found in logs.

4. **Framework laptop quirk** - wlan0 doesn't auto-create on some Intel WiFi setups. The `wifi-interface.service` fixes this.

5. **Certificates expire in 1 year** - If connection fails and certs are old, re-enrollment is needed.

---

## Success Indicators

Connection is working when:
- [ ] `nmcli connection up "UVG CAMPUS CENTRAL"` exits with code 0
- [ ] `nmcli device status` shows wlan0 as "connected"
- [ ] `ping 8.8.8.8` works
- [ ] `journalctl -u NetworkManager` shows "successfully activated"
- [ ] No `eapFail` or `reason: 23` errors in logs

---

## Escalation

If none of the above fixes work:

1. **Capture full logs**:
   ```bash
   # Save diagnostic info
   bash /tmp/diagnostic.sh > /tmp/full_diagnostic.txt

   # Save connection attempt with full NM logs
   sudo nmcli connection up "UVG CAMPUS CENTRAL" 2>&1 | tee /tmp/connection_attempt.log
   journalctl -u NetworkManager --since "5 minutes ago" > /tmp/nm_logs.txt

   # Package for review
   tar czf /tmp/uvg_debug_$(date +%s).tar.gz \
       /tmp/full_diagnostic.txt \
       /tmp/connection_attempt.log \
       /tmp/nm_logs.txt \
       ~/.aruba-onboard/ui.log

   echo "Debug package created: /tmp/uvg_debug_*.tar.gz"
   ```

2. **Check with professor** - Share the debug package

3. **Re-enrollment** - If certificates seem wrong, try enrolling again:
   ```bash
   # Backup old certs
   mv ~/.aruba-onboard ~/.aruba-onboard.bak

   # Delete old connection
   nmcli connection delete "UVG CAMPUS CENTRAL"

   # Re-enroll via portal
   ```

---

## Additional Resources

- Full technical documentation: `UVG-CAMPUS-CENTRAL-documentation.md` (in parent directory)
- Package README: `README.md`
- Student guide: `STUDENT-GUIDE.md`
- Professor's notes: `FOR-PROFESSOR.md`

---

## Quick Reference Commands

```bash
# Check what's running
systemctl is-active iwd NetworkManager

# List connections
nmcli connection show

# Try connecting
nmcli connection up "UVG CAMPUS CENTRAL"

# Live logs
journalctl -u NetworkManager -f

# Check certificates
ls ~/.aruba-onboard/
openssl x509 -in ~/.aruba-onboard/*-user.pem -noout -dates

# Check interface
nmcli device status
ip addr show wlan0

# Manual interface creation (if needed)
sudo iw phy phy0 interface add wlan0 type station
sudo ip link set wlan0 up

# Full diagnostic
bash /tmp/diagnostic.sh
```

---

**Good luck! Remember to always prepare reconnect scripts before testing the UVG connection.**
