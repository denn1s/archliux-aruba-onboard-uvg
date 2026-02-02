# Testing Checklist for aruba-onboard-uvg Package

Before distributing to students, test these scenarios:

## Pre-Test Setup

- [ ] Fresh Arch Linux VM or test partition
- [ ] Internet connection available
- [ ] Access to UVG enrollment portal

---

## Test 1: Clean Install (iwd user switching to NM)

**Starting state**: Fresh Arch with iwd

```bash
# Verify starting state
systemctl is-active iwd  # Should be: active
systemctl is-active NetworkManager  # Should be: inactive

# Install package
cd arch-package
makepkg -si

# Follow post-install instructions
sudo systemctl disable --now iwd
sudo systemctl enable --now NetworkManager

# Check wlan0
nmcli device status | grep wlan0

# If missing (Framework):
sudo systemctl enable --now wifi-interface
sudo systemctl restart NetworkManager
nmcli device status | grep wlan0
```

**Expected**:
- [ ] iwd stops successfully
- [ ] NetworkManager starts successfully
- [ ] wlan0 appears (with or without wifi-interface service)
- [ ] Can connect to test network: `nmcli device wifi connect "TEST-NETWORK"`

---

## Test 2: Enrollment Process

```bash
# Connect to internet first
nmcli device wifi connect "UVG VISITAS"  # Or any network

# Open enrollment portal in browser
# Click enrollment link (arubadp://...)
```

**Expected**:
- [ ] Browser recognizes arubadp:// protocol
- [ ] Aruba Onboard window opens automatically
- [ ] Can log in with UVG credentials
- [ ] Enrollment completes without errors
- [ ] "Enrollment successful!" notification appears
- [ ] Certificates created in `~/.aruba-onboard/`:
  - [ ] UVG CAMPUS CENTRAL-ca.pem
  - [ ] UVG CAMPUS CENTRAL-user.pem
  - [ ] UVG CAMPUS CENTRAL-key.p12

---

## Test 3: Auto-Configuration

**After enrollment completes:**

```bash
# Check if connection was auto-created
nmcli connection show | grep "UVG CAMPUS CENTRAL"

# Inspect the connection
nmcli connection show "UVG CAMPUS CENTRAL"

# Check for password (should be set)
sudo cat /etc/NetworkManager/system-connections/UVG\ CAMPUS\ CENTRAL.nmconnection | grep private-key-password

# Try connecting
nmcli connection up "UVG CAMPUS CENTRAL"
```

**Expected**:
- [ ] Connection profile exists
- [ ] All 802-1x parameters are set:
  - [ ] eap=tls
  - [ ] identity=anonymous@[scope-id].cloudauth.net
  - [ ] ca-cert path correct
  - [ ] client-cert path correct
  - [ ] private-key path correct
  - [ ] private-key-password is set (not empty)
  - [ ] phase1-auth-flags=1
- [ ] Connection succeeds
- [ ] Internet access works
- [ ] Can ping external hosts

---

## Test 4: D-Bus Password Capture

**During enrollment:**

```bash
# Before clicking enrollment link, check:
pgrep -a dbus-monitor  # Should show the enrollment helper's monitor

# Check D-Bus log after enrollment
ls /tmp/aruba-dbus-capture-*.log

# Verify password was captured
grep "private-key-password" /tmp/aruba-dbus-capture-*.log
```

**Expected**:
- [ ] dbus-monitor runs during enrollment
- [ ] D-Bus log file created
- [ ] Password visible in log (for debugging)
- [ ] Password correctly inserted into NM connection

---

## Test 5: Connection Persistence

```bash
# Disconnect
nmcli connection down "UVG CAMPUS CENTRAL"

# Reconnect
nmcli connection up "UVG CAMPUS CENTRAL"

# Reboot
sudo reboot

# After reboot, check auto-configuration
systemctl is-active NetworkManager  # Should be active
nmcli device status | grep wlan0  # Should exist

# Manual connect
nmcli connection up "UVG CAMPUS CENTRAL"
```

**Expected**:
- [ ] Reconnection works without re-entering password
- [ ] After reboot, NetworkManager starts automatically
- [ ] wlan0 exists after reboot
- [ ] Can connect to UVG CAMPUS CENTRAL
- [ ] Internet access works

---

## Test 6: Framework Laptop (wlan0 issue)

**Only if you have a Framework 13 or similar:**

```bash
# Stop NetworkManager
sudo systemctl stop NetworkManager

# Check if wlan0 disappears
ip link show wlan0  # Should fail

# Enable wifi-interface service
sudo systemctl enable --now wifi-interface

# Check if wlan0 is created
ip link show wlan0  # Should succeed

# Start NetworkManager
sudo systemctl start NetworkManager

# Check NM sees wlan0
nmcli device status | grep wlan0
```

**Expected**:
- [ ] wlan0 disappears when NM stops (confirms the issue)
- [ ] wifi-interface service creates wlan0
- [ ] NetworkManager sees and manages wlan0
- [ ] WiFi connections work

---

## Test 7: Error Handling

### Password capture fails

Simulate failure by killing dbus-monitor early:

```bash
# During enrollment, find and kill dbus-monitor
pkill dbus-monitor
```

**Expected**:
- [ ] Notification warns about password capture failure
- [ ] Connection still created (without password)
- [ ] User directed to check ui.log
- [ ] Can manually add password with nmcli

### Enrollment portal unreachable

Disconnect internet before clicking enrollment link.

**Expected**:
- [ ] Aruba Onboard shows connection error
- [ ] Doesn't crash
- [ ] Can retry after reconnecting

---

## Test 8: Package Removal

```bash
# Remove package
sudo pacman -R aruba-onboard-uvg

# Check what remains
ls ~/.aruba-onboard/  # Certificates should still exist
nmcli connection show | grep "UVG CAMPUS CENTRAL"  # Connection should exist
```

**Expected**:
- [ ] Package removes cleanly
- [ ] Certificates NOT deleted (user data preserved)
- [ ] NetworkManager connections NOT deleted
- [ ] NetworkManager still works
- [ ] User can still connect to UVG CAMPUS CENTRAL

---

## Test 9: Reinstall iwd (Rollback)

```bash
# Stop NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager

# Install and start iwd
sudo pacman -S iwd
sudo systemctl enable --now iwd

# Check connection
iwctl station wlan0 connect "TEST-NETWORK"
```

**Expected**:
- [ ] iwd installs successfully
- [ ] wlan0 appears under iwd
- [ ] Can connect to regular networks with iwd
- [ ] UVG CAMPUS CENTRAL still fails with iwd (expected)

---

## Test 10: Multiple Enrollments

```bash
# Enroll once
# [Complete enrollment process]

# Wait for success notification

# Immediately enroll again (click portal link again)
```

**Expected**:
- [ ] Second enrollment overwrites first
- [ ] OR second enrollment updates existing connection
- [ ] No duplicate connections created
- [ ] Final connection works

---

## Regression Tests

After any code changes, re-run:

- [ ] Test 1: Clean install
- [ ] Test 2: Enrollment
- [ ] Test 3: Auto-configuration
- [ ] Test 5: Connection persistence

---

## Known Issues to Document

### Issue: dbus-monitor might require session bus

If password capture fails in testing:
- Check if dbus-monitor has access to session bus
- User might need to be in certain groups
- Document workaround in README

### Issue: zenity not installed

If notifications don't appear:
- zenity is optional dependency
- Fallback to terminal messages works
- Consider adding zenity to dependencies

---

## Success Criteria

Package is ready to distribute when:

- [ ] All core tests (1-5) pass
- [ ] At least one successful end-to-end enrollment
- [ ] Connection works at UVG campus
- [ ] No manual certificate/password configuration needed
- [ ] README and guides are clear
- [ ] Students can follow STUDENT-GUIDE.md successfully

---

## Test Results Log

Document your test results:

**Test Date**: ___________

**Tester**: ___________

**Hardware**:
- CPU: ___________
- WiFi Card: ___________
- Laptop Model: ___________

**Test Results**:
```
Test 1 (Clean Install): PASS / FAIL
Test 2 (Enrollment): PASS / FAIL
Test 3 (Auto-Config): PASS / FAIL
Test 4 (D-Bus Capture): PASS / FAIL
Test 5 (Persistence): PASS / FAIL
Test 6 (Framework Fix): PASS / FAIL / N/A
Test 7 (Error Handling): PASS / FAIL
Test 8 (Removal): PASS / FAIL
Test 9 (Rollback): PASS / FAIL
Test 10 (Multiple Enrollments): PASS / FAIL
```

**Notes**: ___________

**Issues Found**: ___________

**Ready to Distribute**: YES / NO
