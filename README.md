# palera1n on Samsung Galaxy S3 (postmarketOS)
### Jailbreaking an iPhone X from a 2012 Android phone. Yes, really.

---

## What is this?

This repo documents (and provides patches for) compiling and running **[palera1n-c](https://github.com/palera1n/palera1n-c)** on a **Samsung Galaxy S3 (ARMv7)** running **postmarketOS**, used to successfully jailbreak an **iPhone X running iOS 16.7.10**.

This is believed to be the first time palera1n has been compiled from source and executed on a 2012 Android device. The checkm8 exploit completed successfully (`Checkmate!`), PongoOS booted, and the kernel loaded — all over USB from a Galaxy S3.

> **TL;DR:** A potato from 2012 running Linux jailbroke an iPhone X in 2026. The exploit worked first try.

---

## System Specs

| Device | Samsung Galaxy S3 (GT-I9300) |
|---|---|
| CPU | Samsung Exynos 4412 (ARMv7, quad-core 1.4GHz) |
| RAM | 1GB LPDDR2 |
| OS | postmarketOS (Alpine Linux base, ARMv7) |
| Kernel | pmOS downstream kernel (latest) |
| Target | iPhone X (A11), iOS 16.7.10 |

---

## What works

| Feature | Status |
|---|---|
| Compiling palera1n-c from source on ARMv7 | ✅ Confirmed |
| checkm8 exploit (`Checkmate!`) | ✅ Confirmed |
| PongoOS boot | ✅ Confirmed |
| Kernel boot | ✅ Confirmed |
| Rootful jailbreak (`-f` flag) | ✅ Confirmed |
| Rootless jailbreak (`-l` flag) | ❓ Untested (should work) |
| `--setup-fakefs` | ❓ Untested |
| `--setup-partial-fakefs` | ❓ Untested |
| `--pongo-shell` | ❓ Untested |
| `--device-info` | ❓ Untested |
| `--force-revert` | ❓ Untested |
| `--safe-mode` | ❓ Untested |
| `--verbose-boot` | ❓ Untested |
| `--enter-recovery` / `--exit-recovery` | ❓ Untested |

---

## Requirements

- Samsung Galaxy S3 (or any ARMv7 device) running postmarketOS
- USB OTG cable / USB host support
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
  usbmuxd usbmuxd-udev
```

---

## Build Instructions

### 1. Clone palera1n-c

```bash
git clone --recursive https://github.com/palera1n/palera1n-c
cd palera1n-c
```

### 2. Download prebuilt deps (checkra1n binaries, ramdisk, binpack)

```bash
./download_deps.sh
```

### 3. Apply patches

**Patch 1 — Missing `sys/stat.h` in source files:**
```bash
grep -rl "struct stat" src/*.c | xargs -I {} sed -i \
  's/#include <sys\/mman.h>/#include <sys\/mman.h>\n#include <sys\/stat.h>/' {}
```

**Patch 2 — Remove `-static` linker flag (ARMv7 has no static libs for these deps):**
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

### 4. Build

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

---

## Running

### 1. Start usbmuxd

```bash
sudo rc-service usbmuxd start
# or
sudo usbmuxd -f &
```

### 2. Run palera1n (rootful)

```bash
sudo ./src/palera1n -f
```

Follow the on-screen prompts to enter DFU mode. If successful you'll see:

```
[Info]: Checkmate!
[Info]: Entered download mode
[Info]: Booting PongoOS...
[Info]: Found PongoOS USB Device
[Info]: Booting Kernel...
```

---

## Notes & Warnings

> ⚠️ **Battery warning:** Make sure your iPhone has sufficient charge before starting. If the iPhone dies mid-exploit (during PongoOS boot), the process will fail with `LIBUSB_ERROR_NOT_FOUND`. Charge to at least 30% first.

> ⚠️ **This links dynamically.** The binary depends on system `.so` libs on pmOS. It is NOT portable to other distros without recompiling.

> ⚠️ **BusyBox xxd issue:** pmOS ships BusyBox `xxd` which doesn't support the `-C` flag. Install `vim` to get the real `xxd`: `sudo apk add vim`

> ℹ️ **Why does timing work on ARMv7?** checkm8 is a USB packet timing exploit, not a CPU exploit. The iPhone's DFU stack doesn't care what's on the other end of the cable — Linux's USB host stack is deterministic enough to hit the race condition reliably.

> ℹ️ **pymobiledevice3** can be installed in a venv alongside this for post-jailbreak tooling. The `cryptography` package may require building without the Rust backend on ARMv7.

---

## Credits

- [palera1n team](https://github.com/palera1n) — for palera1n-c
- [checkra1n team](https://checkra.in) — for checkm8 and checkra1n
- postmarketOS — for making a 2012 phone run Linux in 2026
- **Noxbit** — for being insane enough to try this
