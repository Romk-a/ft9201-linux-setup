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

## Desktop integration on Astra Linux (Fly)

This section is Astra Fly-specific. By the steps above, fingerprint already works on
the **text console (`login`), `su` and `sudo`**. The graphical components need extra work.

### The Fly lock screen problem

The stock Fly locker (`fly-dm_locker_modern`) has **no fingerprint UI** and its helper
`fly-wmpam` only starts the PAM auth cycle *after the password form is submitted*. So on
the locked screen the reader is idle until you **press Enter** (empty/wrong password is
fine) — only then does `pam_fprintd` arm and you can scan. Dead ends we hit (don't bother):

- Replacing the `fly-dm_locker_modern` binary with a wrapper → `fly-wm` waits for an
  unlock *IPC*, not process exit, and re-locks in a loop.
- `NoPassEnable` in `/etc/X11/fly-dm/fly-dmrc` is "passwordless login", not biometrics.
- `kglobalaccel` does not bind `Meta+L` (checked) — the Fly WM owns it.

### Lock screen with fingerprint via xsecurelock + Matrix rain

We bind `Win+L` to [xsecurelock](https://github.com/google/xsecurelock) instead, with a
`unimatrix` saver (Matrix rain with Japanese katakana) and a background fingerprint
watcher (touch the sensor → unlock; no Enter).

1. Build xsecurelock and install screensaver tools. The rain is drawn by `unimatrix`
   (a single Python script; katakana needs the `Noto Sans CJK JP` font):
   ```bash
   sudo apt-get install -y autoconf automake pkg-config xterm x11-utils \
     fonts-noto-mono fonts-noto-cjk \
     libpam0g-dev libx11-dev libxmu-dev libxcomposite-dev libxext-dev libxfixes-dev \
     libxrandr-dev libxss-dev libxft-dev
   git clone --depth 1 https://github.com/google/xsecurelock.git
   cd xsecurelock && sh autogen.sh && ./configure --with-pam-service-name=xsecurelock
   make && sudo make install      # -> /usr/local/bin/xsecurelock

   # unimatrix (Matrix rain with katakana support) -> /usr/local/bin/unimatrix
   sudo curl -sL -o /usr/local/bin/unimatrix \
     https://raw.githubusercontent.com/will8211/unimatrix/master/unimatrix.py
   sudo chmod 755 /usr/local/bin/unimatrix
   ```

2. PAM service for xsecurelock — **password only** (fingerprint is handled by the watcher,
   so it must NOT be in this stack, or the two fight over the device):
   ```bash
   sudo tee /etc/pam.d/xsecurelock >/dev/null <<'EOF'
   #%PAM-1.0
   auth     required  pam_unix.so try_first_pass nullok
   account  include   common-account
   EOF
   ```

3. Saver (draws into the xsecurelock saver window via an embedded xterm). Font is
   `Noto Sans Mono` with a `Noto Sans CJK JP` fallback (Mono has no katakana):
   ```bash
   sudo tee /usr/local/libexec/xsecurelock/saver_cmatrix >/dev/null <<'EOF'
   #!/bin/sh
   WID="$XSCREENSAVER_WINDOW"
   [ -z "$WID" ] && exec /usr/local/libexec/xsecurelock/saver_blank
   W=1920; H=1080
   command -v xwininfo >/dev/null 2>&1 && \
     eval "$(xwininfo -id "$WID" 2>/dev/null | awk '/Width:/{print "W="$2} /Height:/{print "H="$2}')"
   COLS=$(( W / 8 )); ROWS=$(( H / 16 ))
   [ "$COLS" -lt 20 ] && COLS=80; [ "$ROWS" -lt 10 ] && ROWS=24
   exec xterm -into "$WID" -geometry "${COLS}x${ROWS}+0+0" \
     -fa "Noto Sans Mono,Noto Sans CJK JP" -fs 16 -bg black -fg green \
     -b 0 +sb -bc -e unimatrix -l m -s 89 -i -o
   EOF
   sudo chmod 755 /usr/local/libexec/xsecurelock/saver_cmatrix
   ```
   Tuning the rain — edit the last line (the file is re-read on every lock): `-s` is
   speed (0–100, higher = faster; default 85); `-l m` is the character set (`m` =
   katakana + digits + symbols, `k` = katakana only); `-fs` is the font size. Full
   list: `unimatrix --help`.

4. Wrapper that runs xsecurelock + a background `fprintd-verify` watcher; on a match it
   cleanly unlocks (`SIGTERM` makes xsecurelock kill its children and exit):
   ```bash
   sudo tee /usr/local/bin/xsecurelock-fp >/dev/null <<'EOF'
   #!/bin/sh
   export XSECURELOCK_SAVER=/usr/local/libexec/xsecurelock/saver_cmatrix
   export XSECURELOCK_PASSWORD_PROMPT=cursor
   export XSECURELOCK_SHOW_DATETIME=1
   export XSECURELOCK_SHOW_HOSTNAME=0
   export XSECURELOCK_SAVER_RESET_ON_AUTH_CLOSE=1
   /usr/local/bin/xsecurelock &
   xpid=$!
   ( while kill -0 "$xpid" 2>/dev/null; do
       if fprintd-verify 2>/dev/null | grep -q 'verify-match'; then
           kill -TERM "$xpid" 2>/dev/null; break
       fi
       sleep 0.2
     done ) &
   watcher=$!
   wait "$xpid"
   kill "$watcher" 2>/dev/null
   pkill -INT -u "$(id -u)" -x fprintd-verify 2>/dev/null
   exit 0
   EOF
   sudo chmod 755 /usr/local/bin/xsecurelock-fp
   ```

5. Bind `Win+L` to it — in the **system** file `/usr/share/fly-wm/keyshortcutrc`. On this Fly
   version `fly-wm` regenerates `~/.fly/keyshortcutrc` from the system file on restart, so a
   binding written only to the user copy does **not** survive — it must live in the system file
   (mirror it into the user copy too). Replace the stock `FLYWM_LOCK` line in both:
   ```bash
   sudo sed -i 's#^Mod4|l = FLYWM_LOCK.*#Mod4|l = "/usr/local/bin/xsecurelock-fp"#' \
     /usr/share/fly-wm/keyshortcutrc
   sed -i 's#^Mod4|l = FLYWM_LOCK.*#Mod4|l = "/usr/local/bin/xsecurelock-fp"#' \
     ~/.fly/keyshortcutrc
   ```
   Use the spaced, quoted form exactly — the no-space form `Mod4|l="…"` is rejected and the file
   gets renamed to `keyshortcutrc.bad`. No WM restart is needed: `fly-wm` picks the binding up
   live (if it doesn't, log out and back in). Do **not** run `fly-wmfunc FLYWM_RESTART` from your
   working terminal — it tears down the WM session (closing any terminal in it), and there is no
   lightweight "reload shortcuts" command.

Result: `Win+L` shows the Matrix rain; touch the sensor to unlock instantly, or press a
key to type the password. No countdown, no failed-attempt penalty (the fingerprint is
outside the locker's PAM stack).

> Note: the menu "Lock" item, idle auto-lock and lock-on-suspend still go through
> `FLYWM_LOCK` → the stock locker (Enter-then-scan). After a `fly-wm` package upgrade the
> stock `Mod4|l = FLYWM_LOCK` line is restored in the system file — re-apply step 5.

### Avoid the 30 s fingerprint wait at graphical login

Because `pam_fprintd` is **first** in `common-auth`, the graphical greeter (`fly-dm`) waits
out its timeout before accepting a typed password. Remove fingerprint from the greeter only
(keep it for console/`sudo`/`su`) by replacing `@include common-auth` in `/etc/pam.d/fly-dm`
with an inline copy of `common-auth` **without** the `pam_fprintd` line. The jump offsets
stay correct: the removed line was the target of `default=2`/`success=1`, which then land on
`pam_unix`, whose `success=2` still lands on `pam_permit`. Back up first
(`sudo cp -a /etc/pam.d/fly-dm /etc/pam.d/fly-dm.orig`) and verify the greeter prompts for a
password with no "place finger". Do **not** edit `common-auth` itself — that would disable
fingerprint everywhere.

---

## Credits

- Proprietary FocalTech TOD driver packaging: [`ryenyuku/libfprint-ft9201`](https://github.com/ryenyuku/libfprint-ft9201)
- [libfprint](https://fprint.freedesktop.org/) / [fprintd](https://gitlab.freedesktop.org/libfprint/fprintd)

Contributions welcome — if you get this working on another distro or hardware revision,
please open an issue or PR.
