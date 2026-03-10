# 📄 [Fix_Palera1n](./Fix_Palera1n.md) · [FAQ](./FAQ.md)
# s3ra1n — palera1n on Samsung Galaxy S3 (postmarketOS)
### Jailbreaking an iPhone X from a 2012 Android phone. Yes, really.

---

## What is this?

This repo documents running **[palera1n](https://github.com/palera1n/palera1n)** on a **Samsung Galaxy S3 (ARMv7)** running **postmarketOS**, used to successfully jailbreak and iCloud bypass an **iPhone X running iOS 16.7.10**.

This is believed to be the first documented case of palera1n being executed on a repurposed 2012 Android device running Linux. The checkm8 exploit completed successfully (`Checkmate!`) on the **first attempt**, PongoOS booted, the kernel loaded, and a full rootful jailbreak was achieved — all over USB from a Galaxy S3.

The setup was operated remotely over SSH via tmux from a Pixel 4a, making the full stack:

```
Pixel 4a → SSH → Galaxy S3 (postmarketOS) → USB OTG → iPhone X
```

> **TL;DR:** A potato from 2012 running Linux jailbroke an iPhone X in 2026. The exploit worked first try. checkm8 hit in a single attempt — something that takes 10–15 retries on x86 palen1x.

---

## System Specs

| Device | Samsung Galaxy S3 (GT-I9300) |
|---|---|
| CPU | Samsung Exynos 4412 (ARMv7, quad-core 1.4GHz) |
| RAM | 1GB LPDDR2 |
| OS | postmarketOS (Alpine Linux base, ARMv7) |
| Kernel | Linux 6.18.9-postmarketos-exynos4 |
| Target | iPhone X (A11), iOS 16.7.10 |

---

## What works

| Feature | Status |
|---|---|
| checkm8 exploit (`Checkmate!`) | ✅ Confirmed (first try!) |
| PongoOS boot | ✅ Confirmed |
| Kernel boot | ✅ Confirmed |
| Rootful jailbreak (`-f` flag) | ✅ Confirmed |
| iCloud bypass (Broque Ramdisk, SN swap + activation token) | ✅ Confirmed |
| Rootless jailbreak (`-l` flag) | ✅ Confirmed |
| `--pongo-shell` | ✅ Confirmed |
| `--device-info` | ❌ Broken |
| `--force-revert` | ❔ Unknown |
| `--safe-mode` | ✅ Confirmed |
| `--verbose-boot` | ✅ Confirmed |
| `--enter-recovery` / `--exit-recovery` | ✅ Confirmed |

---

## Requirements

- Samsung Galaxy S3 (or any ARMv7 device) running postmarketOS
- USB OTG cable + USB host support
- iPhone with A8–A11 chip (iPhone 6 through iPhone X)
- iOS 15.0–16.7.x on the target device
- The following packages installed on pmOS:

```bash
sudo apk add \
  cmake build-base vim curl git \
  libimobiledevice-dev libimobiledevice \
  libirecovery libirecovery-dev \
  libusbmuxd-dev libusbmuxd \
  libimobiledevice-glue-1.0 \
  libplist-dev libplist \
  mbedtls-dev readline-dev libusb-dev \
  usbmuxd usbmuxd-udev \
  python3 python3-dev py3-pip
```

---

## Method 1: Use the Official Prebuilt Binary (Recommended)

This is the simplest and most reliable method. The official `palera1n-linux-armel` release binary is statically linked and works out of the box on postmarketOS ARMv7 — no compilation required.

### 1. Download the latest release

```bash
curl -L https://github.com/palera1n/palera1n/releases/download/v2.2.1/palera1n-linux-armel \
  -o ~/palera1n
chmod +x ~/palera1n
```

> Check for newer releases at: https://github.com/palera1n/palera1n/releases

### 2. Kill usbmuxd before running

**Critical:** usbmuxd and palera1n fight over the USB device during the exploit. Always kill it first.

```bash
sudo killall usbmuxd 2>/dev/null; sudo rm -f /var/run/usbmuxd
```

### 3. First run — create fakefs

On the very first jailbreak, create the fakefs partition:

```bash
sudo ~/palera1n -f -B
```

Wait up to 5 minutes for bindfs creation. The iPhone will reboot into iOS when done.

### 4. Subsequent runs — jailbreak

After fakefs is set up, every reboot just needs:

```bash
sudo killall usbmuxd 2>/dev/null
sudo ~/palera1n -f
```

---

## Method 2: Build from Source (Advanced)

Building palera1n-c from source on ARMv7 postmarketOS is possible but requires patches and has a known KPF version mismatch issue (see Pitfalls section). Use Method 1 unless you specifically need a custom build.

### 1. Clone palera1n-c

```bash
git clone --recursive https://github.com/palera1n/palera1n-c
cd palera1n-c
```

### 2. Apply patches

**Patch 1 — Missing `sys/stat.h` headers (Alpine gcc is strict):**
```bash
grep -rl "struct stat" src/*.c | xargs -I {} sed -i \
  's/#include <sys\/mman.h>/#include <sys\/mman.h>\n#include <sys\/stat.h>/' {}
```

**Patch 2 — Remove `-static` linker flag (no static libs for these deps on pmOS):**
```bash
sed -i \
  's/LDFLAGS += -static -no-pie -Wl,--gc-sections/LDFLAGS += -no-pie -Wl,--gc-sections/' \
  Makefile
```

**Patch 3 — Create dep_root/lib and symlink system shared libs:**
```bash
mkdir -p dep_root/lib dep_root/include

ln -s /usr/lib/libimobiledevice-1.0.so dep_root/lib/libimobiledevice-1.0.a
ln -s /usr/lib/libirecovery-1.0.so dep_root/lib/libirecovery-1.0.a
ln -s /usr/lib/libusbmuxd-2.0.so dep_root/lib/libusbmuxd-2.0.a
ln -s /usr/lib/libimobiledevice-glue-1.0.so dep_root/lib/libimobiledevice-glue-1.0.a
ln -s /usr/lib/libplist-2.0.so dep_root/lib/libplist-2.0.a
```

### 3. Build

```bash
make CHECKRA1N_NAME=linux-armel LIBS="-L/usr/lib \
  -l:libimobiledevice-1.0.so \
  -l:libirecovery-1.0.so \
  -l:libusbmuxd-2.0.so \
  -l:libimobiledevice-glue-1.0.so \
  -l:libplist-2.0.so \
  -lmbedtls -lmbedcrypto -lmbedx509 \
  -lreadline -lusb-1.0 -lpthread -lc"
```

Binary will be at `src/palera1n`.

> ⚠️ **KPF version mismatch:** palera1n-c's embedded `checkra1n-kpf-pongo` blob may not match the PongoOS version for your iOS target. If you see `Bad command: rootfs` or `Bad command: overlay` on the iPhone screen, the KPF module is being rejected by PongoOS. The fix is to replace `src/checkra1n-kpf-pongo` with the blob from the matching official release binary. Use Method 1 to avoid this entirely.

---

## Post-Jailbreak: pymobiledevice3

Start usbmuxd **after** palera1n completes and the phone is booted into iOS:

```bash
sudo usbmuxd -f -v &
```

Check for stale usbmuxd instances if you get `lockdown error -8`:

```bash
# Kill zombie instances
sudo kill -9 $(pgrep usbmuxd) && sudo rm -f /var/run/usbmuxd
sudo usbmuxd -f -v &
```

Pair the device (iPhone must be unlocked, screen on):

```bash
sudo pymobiledevice3 lockdown pair
# Watch iPhone screen for Trust popup — tap Trust
```

Check device info:

```bash
pymobiledevice3 lockdown info
```

---

## Pitfalls & Lessons Learned

### 1. usbmuxd conflict
palera1n manages its own USB communication during the exploit. Running usbmuxd simultaneously causes `LIBUSB_ERROR_NOT_FOUND` and drops the device mid-exploit. Always kill usbmuxd before running palera1n.

### 2. KPF version mismatch (source builds only)
The `checkra1n-kpf-pongo` blob in palera1n-c may be built for an older PongoOS/iOS version. Symptoms: `Bad command: rootfs`, `Bad command: overlay`, `Bad command: kpf_flags` on the iPhone's PongoOS console. Root cause: modload succeeds (file uploads fine, byte count matches) but PongoOS silently rejects the module due to API version mismatch. Fix: use the official prebuilt binary which ships with matching KPF+PongoOS versions.

### 3. Zombie usbmuxd after crashes
If the iPhone disconnects unexpectedly (e.g. battery death mid-exploit), usbmuxd leaves a stale process and socket file. Next launch fails with `Another instance is already running`. Fix: `sudo kill -9 <pid> && sudo rm -f /var/run/usbmuxd`.

### 4. Battery death mid-exploit
If the iPhone battery dies during PongoOS boot, the fakefs partition can get corrupted. Symptoms: kernel panic on every subsequent run. Fix: `sudo ~/palera1n -f --force-revert` then `sudo ~/palera1n -f -B` to recreate.

### 5. checkm8 timing on ARMv7
checkm8 is a USB packet timing exploit, not a CPU exploit. The iPhone's DFU stack doesn't care what's on the other end. Linux's USB host stack is deterministic enough to hit the race condition — in testing it succeeded on the **first attempt**, faster and more reliably than x86 palen1x which typically takes 10–15 tries.

### 6. libimobiledevice version
Tested with `libimobiledevice 1.4.0` and `libirecovery 1.3.1` on postmarketOS. The dynamic linking approach (Patch 3) works at runtime but the binary is not portable to other distros without recompiling.

---

## Notes

> ⚠️ **Battery warning:** Make sure your iPhone has at least 30% charge before starting. Battery death during PongoOS/kernel boot will abort the process and may corrupt the fakefs.

> ⚠️ **BusyBox xxd:** pmOS ships BusyBox `xxd` which doesn't support the `-C` flag needed during source builds. Install real xxd: `sudo apk add vim`

> ℹ️ **SSH + tmux workflow:** The entire process was operated remotely via SSH from a Pixel 4a into the S3. tmux is recommended so sessions survive SSH drops mid-exploit.

> ℹ️ **pymobiledevice3 on ARMv7:** Install in a venv. The `cryptography` package may require building without the Rust backend: `CRYPTOGRAPHY_DONT_BUILD_RUST=1 pip install cryptography`

---

## Credits

- [palera1n team](https://github.com/palera1n) — for palera1n
- [checkra1n team](https://checkra.in) — for checkm8 and checkra1n
- postmarketOS — for making a 2012 phone run Linux in 2026
- **Noxbit** — for being insane enough to try this
