# Getting the FocalTech FT9201 (`2808:93a9`) fingerprint reader working on Linux

*Language: **English** | [Русский](README.ru.md)*

A step-by-step guide to make the **Focal-systems.Corp FT9201Fingerprint**
(`USB ID 2808:93a9`) reader work on Linux — all the way to real fingerprint login
(PAM: display manager, `login`, `sudo`).

Verified on **Astra Linux 1.8 SE** (Debian-based, Fly desktop), but the approach
applies to other Debian-derived distributions (substitute your display manager /
PAM profile name).

> If your device has a different product ID (e.g. `2808:9338` — GPD Win 4), the
> USB-id table patch in step 7 is not needed — the driver supports it out of the box.

> ⚠️ **Disclaimer.** This chip has NO upstream support in libfprint/fprintd.
> The guide uses a **proprietary, unofficial FocalTech TOD driver** plus a **manual
> binary patch** of its USB-id table. This is an unofficial solution; use at your own
> risk, especially on a certified/hardened system. Every step is reversible (see
> "Rollback").

---

## 0. Tested environment

| | |
|---|---|
| OS | Astra Linux 1.8 SE (`ID_LIKE=debian`) |
| Kernel | 6.12.60-1-generic |
| Desktop | Fly (`fly-dm`) |
| Device | `Bus ... ID 2808:93a9 Focal-systems.Corp FT9201Fingerprint` |
| Secure Boot | disabled (irrelevant here — no kernel modules are built) |

Check the device:
```bash
lsusb | grep -i 2808
# Bus 001 Device 009: ID 2808:93a9 Focal-systems.Corp FT9201Fingerprint.
```

---

## 1. Why this way (background)

- `FT9201 (2808:93a9)` is supported by neither upstream libfprint nor fprintd.
- A proprietary FocalTech TOD driver exists, packaged in the fork
  [`ryenyuku/libfprint-ft9201`](https://github.com/ryenyuku/libfprint-ft9201) — a
  `.deb` built for Ubuntu 22.04 (`libfprint-2-2 1.94.4+tod1`) that fully replaces the
  system `libfprint-2-2` (a library with the driver baked in + udev rules).
- Two problems to work around:
  1. **Version.** `fprintd` on Astra requires `libfprint-2-2 (>= 1:1.94.5)`, while the
     `.deb` ships `1:1.94.4+tod1` — lower. Otherwise apt considers the dependency
     broken and rolls the driver back. → Rebuild the `.deb` with version
     `1:1.94.5+tod1-astra1`.
  2. **ID table.** The driver's table of supported USB devices contains a single entry —
     `2808:9338` (the GPD Win 4 reader). Our `2808:93a9` is not there, so fprintd
     reports `No driver found`. → Patch one `FpIdEntry`: `pid 0x9338 → 0x93a9`.
     The driver detects the actual sensor type after connecting (via OTP), so the
     `93a9` hardware turns out to be compatible.

---

## 2. Preliminary checks

```bash
# Distribution and device
cat /etc/os-release | grep -E 'NAME|VERSION_ID'
lsusb -d 2808:93a9

# What is available in the repositories (we need fprintd and libpam-fprintd)
apt-cache policy fprintd libpam-fprintd libfprint-2-2

# The version fprintd requires (note the lower bound, usually >= 1:1.94.5)
apt-cache show fprintd | grep -E '^(Version|Depends)'
```

The tools for rebuilding the `.deb` (`dpkg-deb`) and patching (`python3`) are part of
the base system.

---

## 3. Download the driver and check compatibility

```bash
mkdir -p ~/ft9201 && cd ~/ft9201

URL="https://github.com/ryenyuku/libfprint-ft9201/releases/download/1.94.4_20250219/libfprint-2-2_1.94.4%2Btod1-0ubuntu1.22.04.2_amd64_20250219.deb"
curl -sL -o ft9201.deb "$URL"

# Metadata and file list
dpkg-deb -I ft9201.deb
dpkg-deb -c ft9201.deb
```

ABI compatibility check (optional but useful): the maximum required glibc symbol
version must be ≤ the system's, and the dependent libraries must be present.
```bash
dpkg-deb -x ft9201.deb ex
SO=ex/usr/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0
objdump -T "$SO" | grep -oE 'GLIBC_[0-9.]+' | sort -V | uniq | tail -3   # e.g. GLIBC_2.29
ldd --version | head -1                                                  # system, e.g. 2.36
objdump -p "$SO" | awk '/NEEDED/{print $2}'                             # all must exist on the system
```
The separate `libgusb.so.2` from the release is NOT needed if the system already has
`libgusb2 (>= 0.3.0)`.

---

## 4. Rebuild the `.deb` with a bumped version

```bash
cd ~/ft9201
rm -rf build && mkdir build
dpkg-deb -R ft9201.deb build

# Bump the version above fprintd's requirement AND above the repo candidate
sed -i 's/^Version:.*/Version: 1:1.94.5+tod1-astra1/' build/DEBIAN/control

# Verify the version comparison
dpkg --compare-versions "1:1.94.5+tod1-astra1" ge "1:1.94.5" && echo "OK: >= fprintd requirement"

dpkg-deb -b build ft9201-astra.deb
```

> If `fprintd` on your system requires a different lower bound, set a version above it.

---

## 5. Install the driver + fprintd in a single transaction

```bash
cd ~/ft9201
sudo apt-get install -y --allow-downgrades ./ft9201-astra.deb fprintd libpam-fprintd

# Hold libfprint so system updates don't roll the driver back
sudo apt-mark hold libfprint-2-2

# Verify
dpkg -l libfprint-2-2 fprintd libpam-fprintd | grep '^ii'
# libfprint-2-2 must be 1:1.94.5+tod1-astra1
```

---

## 6. Apply the udev rules

The `.deb` already installs `/usr/lib/udev/rules.d/60-libfprint-2.rules`, where the
FocalTech driver is assigned to vendor `2808` with any product, and `0660 plugdev`
permissions are set.

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add --attr-match=idVendor=2808 --attr-match=idProduct=93a9

# The device node should become "crw-rw---- root plugdev"
ls -la /dev/bus/usb/001/   # find your Device number from lsusb
```

Verify the udev property is set on the device:
```bash
SYS=$(for d in /sys/bus/usb/devices/*; do \
  [ "$(cat $d/idVendor 2>/dev/null)" = 2808 ] && \
  [ "$(cat $d/idProduct 2>/dev/null)" = 93a9 ] && echo $d; done)
udevadm info -q property -p "$SYS" | grep LIBFPRINT_DRIVER
# LIBFPRINT_DRIVER=FocalTech Systems Co., Ltd Fingerprint
```

---

## 7. ⭐ Key step: patch the USB-id table

The driver's table only knows `2808:9338`. We change the single entry to `2808:93a9`.
The script searches for the byte pattern `FpIdEntry{pid=0x9338, vid=0x2808}`
(`38 93 00 00 08 28 00 00`) — more robust than a hard-coded offset.

```bash
SOSYS=$(realpath /usr/lib/x86_64-linux-gnu/libfprint-2.so.2)

# Back up OUTSIDE the library directory (see "gotcha" below)!
sudo cp "$SOSYS" /root/libfprint-2.so.2.0.0.orig-backup

sudo python3 - "$SOSYS" <<'PY'
import sys
f = sys.argv[1]
data = bytearray(open(f, 'rb').read())
old = bytes.fromhex('3893000008280000')   # pid 0x9338 + vid 0x2808
new = bytes.fromhex('a993000008280000')   # pid 0x93a9 + vid 0x2808
n = data.count(old)
assert n == 1, f"Expected 1 entry for 2808:9338, found {n}. Do not patch blindly!"
data[:] = data.replace(old, new)
open(f, 'wb').write(data)
print("Patched: 2808:9338 -> 2808:93a9")
PY

sudo ldconfig

# Verify the symlink points to the patched file (NOT the backup)
ls -la /usr/lib/x86_64-linux-gnu/libfprint-2.so.2
```

> 🐞 **Gotcha (important!).** Do not leave the `.so` backup in `/usr/lib/.../` — it has
> the same `libfprint-2.so.2` soname, and `ldconfig` may repoint the symlink to the
> backup (unpatched), so the driver won't "see" the device. Keep the backup outside the
> library directory (e.g. in `/root`).

---

## 8. Enable fingerprint login in PAM

```bash
sudo pam-auth-update --enable fprintd

# Verify: the pam_fprintd line should appear in common-auth
grep pam_fprintd /etc/pam.d/common-auth
```
`common-auth` is included by `login`, `fly-dm` (display manager) and `sudo`. The setup
is safe: on fingerprint failure it falls back to the password — the system won't lock
you out.

---

## 9. Verify it works

```bash
sudo systemctl restart fprintd

# The device should appear: object path ".../Device/0"
dbus-send --system --print-reply --dest=net.reactivated.Fprint \
  /net/reactivated/Fprint/Manager net.reactivated.Fprint.Manager.GetDevices
```

Enroll a fingerprint — **as the target user (without sudo)**, separately for each
account (root and a regular user have separate storage):
```bash
fprintd-enroll          # press the finger ~13 times until "enroll-completed"
fprintd-verify          # press once -> "verify-match"
fprintd-list "$USER"    # list enrolled fingers
```

The scan type is **press** (touch, not swipe). The messages `enroll-swipe-too-short`
and `enroll-remove-and-retry` during the process are normal (finger lifted too early /
needs to be removed and re-applied); the driver simply skips them.

After this, fingerprint login works on the `fly-dm` display manager, on unlock, and for
`sudo` (unless `sudo` is configured as NOPASSWD).

---

## 10. Rollback

```bash
# Disable fingerprint login
sudo pam-auth-update --disable fprintd

# Restore the original (unpatched) library
sudo cp /root/libfprint-2.so.2.0.0.orig-backup \
        $(realpath /usr/lib/x86_64-linux-gnu/libfprint-2.so.2)
sudo ldconfig

# Unhold the version and/or remove the packages
sudo apt-mark unhold libfprint-2-2
# Full removal:
sudo apt-get remove --purge fprintd libpam-fprintd
# Restore the stock libfprint from the distro repo:
sudo apt-mark unhold libfprint-2-2 && sudo apt-get install --reinstall --allow-downgrades libfprint-2-2

# Delete enrolled fingerprints
sudo rm -rf /var/lib/fprint/*
```

---

## 11. Maintenance and risks

- `libfprint-2-2` is held (`apt-mark hold`). Don't unhold it unnecessarily — an update
  from the distro repo will overwrite the patched library.
- After a **kernel** update there's nothing to do (the driver is userspace, not a kernel
  module).
- If someone unholds and updates libfprint, or reinstalls the package — you must **redo
  step 7** (patch) and step 9 (restart fprintd).
- The driver is proprietary; behavior may differ on a specific `93a9` hardware revision.
  If `enroll`/`verify` are unstable — a different driver build may be required.

---

## TL;DR (everything in one block)

```bash
mkdir -p ~/ft9201 && cd ~/ft9201
URL="https://github.com/ryenyuku/libfprint-ft9201/releases/download/1.94.4_20250219/libfprint-2-2_1.94.4%2Btod1-0ubuntu1.22.04.2_amd64_20250219.deb"
curl -sL -o ft9201.deb "$URL"
rm -rf build && mkdir build && dpkg-deb -R ft9201.deb build
sed -i 's/^Version:.*/Version: 1:1.94.5+tod1-astra1/' build/DEBIAN/control
dpkg-deb -b build ft9201-astra.deb
sudo apt-get install -y --allow-downgrades ./ft9201-astra.deb fprintd libpam-fprintd
sudo apt-mark hold libfprint-2-2
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add --attr-match=idVendor=2808 --attr-match=idProduct=93a9
SOSYS=$(realpath /usr/lib/x86_64-linux-gnu/libfprint-2.so.2)
sudo cp "$SOSYS" /root/libfprint-2.so.2.0.0.orig-backup
sudo python3 - "$SOSYS" <<'PY'
import sys; f=sys.argv[1]; d=bytearray(open(f,'rb').read())
old=bytes.fromhex('3893000008280000'); new=bytes.fromhex('a993000008280000')
assert d.count(old)==1; d[:]=d.replace(old,new); open(f,'wb').write(d); print("patched")
PY
sudo ldconfig
sudo pam-auth-update --enable fprintd
sudo systemctl restart fprintd
fprintd-enroll && fprintd-verify
```

---

## Credits

- Proprietary FocalTech TOD driver packaging: [`ryenyuku/libfprint-ft9201`](https://github.com/ryenyuku/libfprint-ft9201)
- [libfprint](https://fprint.freedesktop.org/) / [fprintd](https://gitlab.freedesktop.org/libfprint/fprintd)

Contributions welcome — if you get this working on another distro or hardware revision,
please open an issue or PR.
