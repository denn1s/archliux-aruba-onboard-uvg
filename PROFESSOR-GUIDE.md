# Aruba Onboard UVG Package - Professor's Guide

## What You Have

A complete Arch Linux package that automates UVG CAMPUS CENTRAL WiFi enrollment for your students.

**Student experience:**
1. Install package (5 seconds)
2. Click university enrollment link (30 seconds)
3. Connected (automatic)

**No manual certificate configuration required.**

---

## Distribution Options

### Option 1: Source Package (Recommended for GitHub/GitLab)

Share the entire `arch-package/` folder:

```bash
# Students clone/download, then:
cd arch-package
makepkg -si
```

### Option 2: Pre-built Binary Package

Build once, distribute the `.pkg.tar.zst`:

```bash
cd arch-package
makepkg -s
# Creates: aruba-onboard-uvg-1.0.0-1-x86_64.pkg.tar.zst

# Students install with:
sudo pacman -U aruba-onboard-uvg-1.0.0-1-x86_64.pkg.tar.zst
```

### Option 3: AUR Package (Best for large distribution)

Upload to AUR (Arch User Repository):
1. Create AUR account
2. Upload PKGBUILD and sources
3. Students install with: `yay -S aruba-onboard-uvg`

---

## Files in arch-package/

```
arch-package/
├── PKGBUILD                        # Package build script
├── Aruba_Onboard_Installer.deb     # Original Ubuntu package (8.6MB)
├── aruba-onboard-wrapper           # Manages srv + ui lifecycle
├── aruba-enrollment-helper         # Auto-configures NetworkManager
├── aruba-onboard.desktop           # Desktop integration
├── wifi-interface.service          # Framework laptop fix
├── aruba-onboard-uvg.install       # Post-install messages
├── README.md                       # Technical documentation
├── STUDENT-GUIDE.md                # Simple setup guide for students
├── TESTING-CHECKLIST.md            # For you to verify it works
└── FOR-PROFESSOR.md                # This file
```

---

## How It Works (Technical)

### Problem Solved

UVG's WiFi uses 802.1X EAP-TLS (certificate authentication). The challenges:

1. **Aruba tools are Ubuntu-only** → We repackage for Arch
2. **iwd doesn't work** → Package forces NetworkManager + wpa_supplicant
3. **PKCS#12 password is randomly generated** → We capture it from D-Bus
4. **Manual configuration is complex** → Fully automated

### The Automation Flow

```
Student clicks enrollment link (arubadp://)
    ↓
Wrapper starts onboard-srv (D-Bus service)
    ↓
Wrapper starts onboard-ui (enrollment GUI)
    ↓
Helper monitors D-Bus in background
    ↓
Student enters UVG credentials in UI
    ↓
Aruba CloudGuest issues certificates
    ↓
onboard-srv sends config to NetworkManager via D-Bus
    ↓
Helper captures PKCS#12 password from D-Bus
    ↓
Helper auto-creates NetworkManager connection
    ↓
Student sees "Enrollment successful!" notification
    ↓
Student connects: nmcli connection up "UVG CAMPUS CENTRAL"
```

**Zero manual steps after clicking the enrollment link.**

---

## Before Distributing

### 1. Test It Yourself

Follow `TESTING-CHECKLIST.md` on:
- Your own laptop (at UVG campus)
- A student's laptop (with permission)
- A VM if possible

**Critical tests:**
- [ ] Enrollment completes
- [ ] Password is auto-captured
- [ ] Connection works at UVG
- [ ] Internet access works

### 2. Update Checksums

After any modifications to scripts:

```bash
cd arch-package
sha256sum Aruba_Onboard_Installer.deb aruba-onboard-wrapper \
          aruba-enrollment-helper wifi-interface.service \
          aruba-onboard.desktop

# Update sha256sums=() array in PKGBUILD with these values
```

### 3. Customize

Edit these files with your contact info:

- **PKGBUILD**: Line 1 (Maintainer)
- **README.md**: Support section at bottom
- **STUDENT-GUIDE.md**: Questions section

### 4. Write a Short Announcement

Example:

> **Connecting to UVG CAMPUS CENTRAL on Arch Linux**
>
> I've created an automated package for Arch/EndeavourOS/Manjaro users to connect to our university WiFi.
>
> **Installation**: [link to GitHub/shared folder]
> **Guide**: See STUDENT-GUIDE.md
> **Support**: [your contact method]
>
> The entire process takes < 5 minutes.

---

## Student Support

### Common Issues and Solutions

**"wlan0 not found"** (Framework 13 laptops)
```bash
sudo systemctl enable --now wifi-interface
sudo systemctl restart NetworkManager
```

**"Password capture failed"**
- Usually works anyway (NM will prompt)
- Or: password is in `~/.aruba-onboard/ui.log`
- Add manually: `nmcli connection modify "UVG CAMPUS CENTRAL" 802-1x.private-key-password "PASSWORD"`

**"Connection failed" at UVG**
- Verify they're using NetworkManager (not iwd): `systemctl is-active NetworkManager`
- Check logs: `journalctl -u NetworkManager -f`
- Possible certificate issue (re-enroll)

**"Still using iwd"**
- They skipped the switch step
- Follow: `sudo systemctl disable --now iwd && sudo systemctl enable --now NetworkManager`

---

## For Ubuntu/Manjaro Students

### Ubuntu (16.04+)

The original .deb works directly:
```bash
sudo dpkg -i Aruba_Onboard_Installer.deb
sudo apt-get install -f
```

Then click enrollment link. NetworkManager is default on Ubuntu.

### Manjaro

Your Arch package works on Manjaro (they're Arch-based).

### Other Distros

- **Fedora/RHEL**: Rebuild as RPM (complex)
- **Debian**: Use the .deb directly
- **OpenSUSE**: Manual installation only

---

## Updating the Package

When you need to update (e.g., new Aruba version):

1. Get new `Aruba_Onboard_Installer.deb`
2. Replace in `arch-package/`
3. Update `pkgver` in PKGBUILD
4. Regenerate checksums
5. Test enrollment
6. Redistribute

---

## Security Considerations

### What This Package Does

**Safe:**
- Extracts official Aruba binaries
- Monitors D-Bus (read-only, user's own session)
- Writes NetworkManager configs (standard location)
- No root daemons, no network services

**Student's credentials:**
- Typed into official Aruba UI (not our scripts)
- Transmitted directly to Aruba CloudGuest (encrypted)
- Never logged or stored by our scripts

**PKCS#12 password:**
- Generated by Aruba, captured from D-Bus
- Stored in NM connection file (root-only readable, mode 600)
- Automatically cleaned from logs

### Code Review

All scripts are in plaintext:
- `aruba-onboard-wrapper` - 40 lines of bash
- `aruba-enrollment-helper` - 90 lines of bash
- No compiled binaries besides official Aruba tools

Students can (and should) review before installing.

---

## Maintenance

### Certificates Expire After 1 Year

Students need to re-enroll annually:
1. Click the enrollment portal link again
2. Re-enter credentials
3. Package auto-updates their connection

No need to reinstall the package.

### Package Updates

You control updates:
- **Via AUR**: Update PKGBUILD on AUR, students get notified
- **Via GitHub**: Push updates, students pull and rebuild
- **Via direct download**: Re-share the new package file

Typical update frequency: Once per year (or when Aruba releases new tools).

---

## Alternative: Manual Instructions

If you prefer not to use this package, students can:

1. Extract the .deb manually
2. Run the enrollment tools manually
3. Extract certificates manually
4. Configure NetworkManager manually

See `UVG-CAMPUS-CENTRAL-documentation.md` for the full manual process.

**But this package saves ~30 minutes per student** and eliminates configuration errors.

---

## Statistics (Estimated)

**Manual setup time**: 30-60 minutes per student
**With this package**: 5 minutes per student

**For a class of 30 students:**
- Time saved: 12-25 hours total
- Fewer support requests
- Higher success rate

---

## Feedback Loop

After distributing, track:
- How many students successfully connect
- Common issues encountered
- Time spent on support

Use this to improve:
- STUDENT-GUIDE.md (clarify confusing parts)
- Error messages in scripts
- Dependencies in PKGBUILD

---

## Legal/Policy Check

Before distributing:

- [ ] Check UVG IT policy on third-party network tools
- [ ] Confirm Aruba license allows redistribution of .deb (likely yes, it's freely downloadable)
- [ ] Consider liability (provide as-is, educational use)

**Recommendation**: Mention to UVG IT that you're helping Linux students. They may officially support it.

---

## Success Metrics

Package is working well when:

- [ ] >80% of students connect successfully on first try
- [ ] <10% need support after following guide
- [ ] Students recommend it to other Linux users
- [ ] Other professors adopt it

---

## Future Improvements

Possible enhancements (v2.0):

1. **GUI installer** instead of command line
2. **Auto-detect Framework laptops** and enable wifi-interface service automatically
3. **Retry mechanism** if enrollment fails
4. **Multi-language support** (Spanish for UVG)
5. **Certificate expiration warnings** (notify before 1-year mark)

---

## Questions?

If you need help with:
- Testing the package
- Debugging issues
- Updating for new Aruba versions
- AUR publication

Feel free to reach out (or reference `UVG-CAMPUS-CENTRAL-documentation.md` for deep technical details).

---

## Quick Start for You

1. **Test**: Follow `TESTING-CHECKLIST.md` on your laptop at UVG
2. **Customize**: Add your name/contact to PKGBUILD and guides
3. **Distribute**: Share `arch-package/` folder or build `.pkg.tar.zst`
4. **Announce**: Post in class Discord/Moodle/email with link to STUDENT-GUIDE.md
5. **Support**: Answer questions as they come

**Good luck helping your students connect!**
