# HTC Desire 12+ (`brepdugl`) — Android 11 GSI Project

Bringing **Android 11 (LineageOS 18.1 GSI)** to the **HTC Desire 12+** (`brepdugl`), a
Qualcomm **Snapdragon 450 (MSM8937)** device running the **3.18.71** kernel, via **Project
Treble**. The device is a **Treble ARM64, A-only (system-as-system / non-system-as-root)**
layout, so the stock **Android 8.0 boot image** is kept and the GSI is flashed only to
`/system`. Android 11 is the highest Android version supported by a 3.18 kernel.

**Status:** the GSI **boots to `sys.boot_completed=1`** (the Google setup screen), then
~1 minute later the device triggers a **"factory data reset" and reboots in a loop**. Every
flashing problem encountered so far has been solved; the final boot-loop root cause is still
being investigated (see [Root cause & final fix (pending)](#root-cause--final-fix-pending)).

> **Warning:** flashing is inherently risky and can brick a device. This document records what
> was done on a specific device for reference. Proceed at your own risk.

---

## Table of contents

1. [Device & partition layout](#device--partition-layout)
2. [Prerequisites, files & ROMs](#prerequisites-files--roms)
3. [Boot flow (A-only / non-system-as-root)](#boot-flow-a-only--non-system-as-root)
4. [Every problem → fix](#every-problem--fix)
5. [The ADB "catch" procedure](#the-adb-catch-procedure)
6. [Current status & open issue](#current-status--open-issue)
7. [Credits](#credits)
8. [License / attribution](#license--attribution)

---

## Device & partition layout

| Partition | Block device | Notes |
|---|---|---|
| `system` | `mmcblk0p24` | Holds the Android 11 (LineageOS 18.1) GSI |
| `vendor` | `mmcblk0p25` | Stock Android 8.0 vendor (`fstab` + HALs) |
| `cache` | `mmcblk0p26` | |
| `boot` | `mmcblk0p21` | **Stock Android 8.0 boot image** (stock kernel + ramdisk) |
| `userdata` (`/data`) | `mmcblk0p53` | ~23 GB userdata partition |

- **Boot device path:** `/dev/block/platform/soc/7824900.sdhci`
- **`by-name` symlinks:** `/dev/block/bootdevice/by-name/<name>` (e.g. `/dev/block/bootdevice/by-name/userdata`)

**Device summary**

| Property | Value |
|---|---|
| Device | HTC Desire 12+ |
| Codename | `brepdugl` |
| SoC | Snapdragon 450 (MSM8937) |
| Kernel | 3.18.71 |
| Treble | ARM64, **A-only** (system-as-system / non-system-as-root) |
| Stock OS | Android 8.0 |

---

## Prerequisites, files & ROMs

### 1. TWRP recovery — `twrp-3.7.0_9-0-brepdugl.img`

Used to flash the GSI and to reformat `/data`/`/system` from recovery (the phone has no usable
custom ROM installer; everything is done by hand from TWRP's shell).

- **Device page:** <https://twrp.me/htc/htcdesire12+.html>
- **Download mirrors:**
  - Primary (Americas): <https://dl.twrp.me/brepdugl>
  - Primary (Europe): <https://eu.dl.twrp.me/brepdugl>
- **Device tree / files:** <https://github.com/TeamWin/android_device_htc_brepdugl>
- Maintainer: `sabpprook`

### 2. LineageOS 18.1 (Android 11) GSI — phhusson / TrebleDroid

The GSI is a generic **system image** from **phhusson's Treble Experimentations** (now
**TrebleDroid**). Two variants were used here:

| File | Variant | Meaning |
|---|---|---|
| `lineage-18.1-arm64_agS.img` | **GApps** | A-only + Google Apps + superuser (root) |
| `lineage-18.1-arm64_avS.img` | **Vanilla** | A-only + vanilla (no GApps) + superuser (root) |

- **Canonical releases page:** <https://github.com/phhusson/treble_experimentations/releases>
- **Project home:** <https://github.com/phhusson/treble_experimentations>

> **Naming note.** The Android 11 (LineageOS 18.1) builds are *older* releases; upstream now
> ships Android 12.1+ under a different asset naming (`system-squeak-arm64-*`). Historically the
> naming is: `arm64` = 64-bit; `a` = **A-only** (system-as-system, *this device*), `b` = A/B;
> `g` = GApps, `v` = vanilla, `o` = Go GApps; `S` = superuser (root), `N` = no-superuser. Upstream
> assets are compressed as `.img.xz` — decompress before flashing. Because release tags rotate and
> specific asset URLs change, the **canonical releases page is linked above** rather than a
> hard-coded asset URL; pick the `arm64` **A-only** (`a`) variant for your build.

### 3. Android SDK platform-tools — **use the 64-bit (x64) build**

- **Download:** <https://developer.android.com/studio/releases/platform-tools>

You **must use the x64 (64-bit) build** of `adb`/`fastboot`. The 32-bit x86 builds cannot buffer
more than ~2 GB and fail with:

```
terminate called after throwing an instance of 'std::bad_alloc'
```

which is exactly what happens when trying to `fastboot flash`/`boot` a ~2.7 GB GSI directly.
This is worked around entirely (see [problem 2](#problem-2-fastboot-stdbad_alloc--2-gb)).

### 4. Helper scripts (this project)

Small Python helpers used to move large images and to re-catch the device during boot:

- `dd_stream.py` — streams a large image over `adb` and writes it with `dd` in chunks.
- `make_sparse.py` — builds an Android sparse image for fastboot.
- `adb_race.py` / `adb_race2.py` — background pollers that issue `adb reboot bootloader` the
  moment `adb` sees the device (see [The ADB "catch" procedure](#the-adb-catch-procedure)).
- `capture_logcat.py`, `unpack_boot.py`, `extract_ramdisk.py` — boot logcat capture and boot
  image/ramdisk inspection helpers.

---

## Boot flow (A-only / non-system-as-root)

This is a **non-system-as-root** device, so:

- The **boot image stays stock Android 8.0** (stock 3.18 kernel + stock Android 8.0 ramdisk).
- The **GSI is flashed only to `/system`** (`mmcblk0p24`); there is no `system`-as-`root`.

Boot proceeds roughly as follows:

1. Bootloader loads the stock **Android 8.0 boot image** (`mmcblk0p21`): kernel + Android 8.0 ramdisk.
2. The Android 8.0 `init` runs `on fs`, which calls
   `mount_all /vendor/etc/fstab.qcom`
   (reached via `/vendor/etc/init/hw/init.qcom.rc` → `init.target.rc`).
3. Android 8.0's `fs_mgr` mounts `/dsp` and `/cota`, but on this device it historically **failed to
   mount `/data`** (the fix is [problem 5](#problem-5-data-never-mounted-at-boot)).
4. Once the GSI's userspace comes up, **Android 11's `vold` (Vold 3.0)** is responsible for FBE
   (fscrypt) on `/data`.

Because the ramdisk is Android 8.0 but the system is Android 11, the mount/encryption expectations
of the two halves differ — this mismatch is the root of several issues below (and the current
factory-reset loop).

---

## Every problem → fix

### Problem 1: "no space left on device" / 1.4 GB limit

**Symptom.** Writing the ~2.7 GB GSI to `/system` repeatedly failed around **1.4 GB** with
`no space left on device`, even though `/system` is large enough.

**Cause.** *Not* a hardware fault. The eMMC was in a bad TRIM/discard state, so the partition
couldn't accept the full write.

**Fix.** Run `mke2fs` on the target partition **before** writing — `mke2fs` issues a discard/TRIM
that clears the stale state. After that the full 2.7 GB GSI `dd`s cleanly.

```sh
# From TWRP (or an adb root shell): discard & re-init the filesystem first
mke2fs -t ext4 /dev/block/mmcblk0p24

# then write the image (bs=1048576, see problem 4)
dd if=/sdcard/lineage-18.1-arm64_agS.img of=/dev/block/mmcblk0p24 bs=1048576
```

### Problem 2: `fastboot` `std::bad_alloc` (>2 GB)

**Symptom.** `fastboot flash system <gsy>.img` (or `fastboot boot`) aborted with:

```
terminate called after throwing an instance of 'std::bad_alloc'
```

**Cause.** `fastboot.exe` (and `adb.exe`) is a **32-bit x86** binary and cannot buffer images
larger than ~2 GB.

**Fix.** Don't push the image through fastboot at all. Boot TWRP, then push the image over ADB and
`dd` it from inside TWRP.

```sh
# 1. Boot TWRP over fastboot (this small image is fine)
fastboot boot twrp-3.7.0_9-0-brepdugl.img

# 2. Push the image into the device (e.g. to /sdcard or /tmp)
adb push lineage-18.1-arm64_agS.img /sdcard/

# 3. Write it with dd from TWRP's shell
adb shell dd if=/sdcard/lineage-18.1-arm64_agS.img of=/dev/block/mmcblk0p24 bs=1048576
```

> Also use the **64-bit platform-tools** (see [Prerequisites](#prerequisites-files--roms)) so the
> host tools are never the bottleneck.

### Problem 3: TWRP can't mount encrypted `/data`

**Symptom.** After booting TWRP, `/data` would not mount (it was still encrypted from stock
Android 8.0 FDE), so nothing could be inspected or flashed there.

**Fix.** Reformat `/data` with `mke2fs`, then mount it.

```sh
# From TWRP shell
mke2fs -t ext4 /dev/block/bootdevice/by-name/userdata
mount /data
```

> This **destroys all user data** on `/data`. Do it only when you don't need the old contents.

### Problem 4: busybox `dd` quirks (no `4M` suffix, no `conv=notrunc`)

**Symptom.** TWRP's busybox `dd` rejects the common flags/suffixes used on a desktop `dd`.

**Cause.** busybox `dd`:
- does **not** support the `4M` suffix (only plain byte counts), and
- does **not** support `conv=notrunc`.

**Fix.** Use a plain byte count with `bs=1048576` (1 MiB) and omit `conv=notrunc`.

```sh
# WRONG on busybox:
#   dd if=img of=/dev/block/mmcblk0p24 bs=4M conv=notrunc
#     -> "4M" suffix not understood, "conv=notrunc" not supported

# CORRECT:
dd if=/sdcard/lineage-18.1-arm64_agS.img of=/dev/block/mmcblk0p24 bs=1048576
```

### Problem 5: `/data` never mounted at boot (zygote crash)

**Symptom.** At boot, zygote crashed with:

```
Error creating cache dir /data/dalvik-cache/arm64
```

because `/data` was never mounted.

**Cause.** The `fstab.qcom` `/data` entry used `/dev/block/bootdevice/by-name/userdata`. That
`by-name` symlink is created by `ueventd` and is **not yet ready at `on fs`** time. By contrast,
`/dsp` and `/cota` mount fine because they use the direct `mmcblk0pN` nodes. (Confirmed against
AOSP `fs_mgr` source — the symlink is simply too late.)

**Fix.** Add a **direct mount** in `init.target.rc` that waits for the node, then mounts it with
the direct `by-name` path (which is ready by the time the `wait` returns).

Append to `/vendor/etc/init/hw/init.target.rc`:

```rc
on fs
    wait /dev/block/bootdevice/by-name/userdata
    mount ext4 /dev/block/bootdevice/by-name/userdata /data noatime nosuid nodev barrier=1
```

This is what finally made `/data` mount and let boot reach `boot_completed=1`.

### Problem 6: "Decryption unsuccessful / Forcing factory reset"

**Symptom.** After flashing the Android 11 GSI, the device showed the
**"Decryption unsuccessful / Forcing factory reset"** screen.

**Cause.** The stock `/data` fstab entry carried `forceencrypt=footer`, i.e. full-disk encryption
(FDE). **FDE is unsupported in Android 11**, which only supports file-based encryption (FBE).

**Fix.** Remove the `forceencrypt=footer` (and related `check`, `quota`, `discard`) flags from the
`/data` line in `/vendor/etc/fstab.qcom`.

The `/data` line now reads:

```
/dev/block/bootdevice/by-name/userdata  /data  ext4  noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait
```

---

## The ADB "catch" procedure

The factory-reset recovery screen has **USB off** (no adb/fastboot data connection), and the
hardware **Vol-Down + Power combo does not enter fastboot** on this QRD device. To get the device
back under control:

1. From the **"Try again / Factory data reset"** recovery screen, tap **"Try again"**.
2. During the subsequent boot there is a brief window where `adb` reports the device as `device`.
   Run `adb_race.py` / `adb_race2.py` in the background — they poll and fire
   `adb reboot bootloader` the instant the device appears.
3. Once in bootloader/fastboot, boot TWRP:

```sh
fastboot boot twrp-3.7.0_9-0-brepdugl.img
```

This is the reliable way to re-enter TWRP when the device is stuck in the reset loop.

---

## Current status & open issue

**Working:** the device boots through to the Google setup screen and reaches
`sys.boot_completed=1`.

**Broken:** ~15 seconds after `boot_completed`, the device triggers a **factory reset**
(`--prompt_and_wipe_data` → recovery "Try again / Factory data reset") and loops. The root cause
**has now been identified** (see below).

Observed in `logcat_boot.txt`:

- A **Bluetooth app crash** (twice):

  ```
  SQLiteCantOpenDatabaseException: Cannot open database
  '/data/user_de/0/com.android.bluetooth/databases/bluetooth_db':
  Directory /data/user_de/0/com.android.bluetooth/databases doesn't exist
  ```

  This is the *only* crash, and it is an **app** crash, not a system crash.
- **No** `FATAL EXCEPTION` in `system_server`, **no** RescueParty message captured, and no clear
  reboot reason in the logcat (capture ended at boot phase 1000, before the reset trigger).
- `Power-off reason: Triggered from PS_HOLD` — a normal software shutdown, consistent with either
  a RescueParty-triggered reset or a "Decryption unsuccessful" reset.

### Root cause (confirmed from a full fresh-boot logcat capture)

vold's **user-0 storage preparation fails**, which escalates to a factory reset:

```
vold: fscrypt_prepare_user_storage for volume null, user 0, serial 0, flags 2
vold: Preparing: /data/system_ce/0        (ok)
vold: Preparing: /data/misc_ce/0          (ok)
vold: Preparing: /data/vendor_ce/0
cutils: Failed to mkdir(/data/vendor_ce/0): No such file or directory   <-- FAILS
PackageManager: Destroying user 0 on volume null because we failed to prepare
uncrypt: received command [--prompt_and_wipe_data]                     <-- factory reset
```

`/data/vendor_ce` (a base FBE directory) was never created, so `mkdir /data/vendor_ce/0` fails,
`PackageManager` destroys user 0, and the system requests a factory reset.

**Why the tree was incomplete:** `/data` was mounted **unencrypted by a direct `on fs` `mount`**
in `init.target.rc`, with **no** `fileencryption=` flag in `fstab.qcom`. Therefore
`ro.crypto.type` was never set (`ro.crypto.state=unsupported`), and Android 11 `vold`
(`fscrypt_is_native()` → false) never built the full FBE CE/DE directory tree.

**Ruled out:**

- **RescueParty** — `persist.sys.disable_rescue` + a DeviceConfig flag were set; RescueParty logged
  "Disabled because of DeviceConfig flag" and the wipe still fired via a non-RescueParty path.
- **`updatable process` crash (livedisplay HAL)** — an older `console-ramoops.txt` showed this, but
  the current GSI has no livedisplay HAL and `ro.init.updatable_crashing` was empty; that log was
  stale.
- The **Bluetooth SQLite crash** is a **symptom** of the missing `user_de` dirs, not the trigger.

Notes on `/data` as currently configured:

- `/vendor/etc/fstab.qcom` `/data` line: `noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait`
  (with `forceencrypt=footer`, `check`, `quota`, `discard` removed).
- `/data` was reformatted with:

  ```sh
  mke2fs -t ext4 -O encrypt,quota,project /dev/block/mmcblk0p53
  ```

  (adds the missing `encrypt`, `quota`, `project` filesystem features — still did not stop the
  reset loop).

---

## Root cause & final fix

**Root cause:** `/data` was mounted unencrypted via a direct `init` `on fs` mount (no
`fileencryption=` in the fstab), so Android 11 `vold` never owned the volume, the FBE CE/DE
directory tree was incomplete (`/data/vendor_ce` missing), user-0 storage preparation failed, and
`PackageManager` requested a factory reset (`--prompt_and_wipe_data`).

**Fix (verified working):** keep `/data` unencrypted with the direct `on fs` mount, and pre-create
the two directories that `vold`'s user-storage preparation needs but never builds on this
Android-8-init + Android-11-vold combo:

```sh
# from TWRP:
mount -t ext4 -o rw /dev/block/bootdevice/by-name/vendor /vendor
# (restore original fstab.qcom: /data unencrypted, by-name userdata, 'wait')
# (restore original init.target.rc: with the direct 'on fs' mount)
mke2fs -t ext4 /dev/block/mmcblk0p53
mount -t ext4 /dev/block/mmcblk0p53 /data
mkdir -p /data/vendor_ce /data/media/0
chown system:system /data/vendor_ce
chown -R media_rw:media_rw /data/media
chmod 0770 /data/vendor_ce /data/media /data/media/0
# reboot
```

Result: `sys.boot_completed=1` holds indefinitely (no `--prompt_and_wipe_data`), `/data/user/0`
populates (~230 app dirs), `/storage/emulated/0` mounts, and Bluetooth no longer crash-loops.

**Attempts that did NOT work (documented for completeness):**

| Attempt | fstab `/data` flag | Result |
|---|---|---|
| FBE | `fileencryption=aes-256-xts` (with `mke2fs -O encrypt`) | **Boot hang** — Android 8.0 `init`/fs_mgr FBE path does not cooperate with Android 11 vold |
| FDE | `forceencrypt=footer` (plain `mke2fs`, 16 KB footer reserved) | Encryption succeeded (`ro.crypto.state=encrypted`) but still reset — vold still couldn't build the user-storage tree |

The working fix is a **band-aid** (unencrypted `/data` + pre-created `vendor_ce`/`media/0`), not
true FBE/FDE. A proper encrypted setup would require an Android-11-compatible boot image, which
does not exist for this device.

---

## Credits

- **[LineageOS](https://lineageos.org/)** — for the Android 11 (LineageOS 18.1) base system.
- **[phhusson / TrebleDroid](https://github.com/phhusson/treble_experimentations)** — for the
  **phh's Treble Experimentations** GSIs that make a generic Android system image boot on this
  Treble-enabled device.
- **[TWRP / TeamWin](https://twrp.me/)** — for the `brepdugl` recovery (`twrp-3.7.0_9-0`),
  device tree, and recovery tooling.
- **The XDA community** — for the countless Treble/GSI guides, fstab/encryption documentation,
  and troubleshooting threads that informed this work.

---

## License / attribution

This repository is a **documentation / work-log** for a personal device-flashing effort. The
project does not redistribute the referenced ROMs, recoveries, or tools, and it makes **no claim**
to relicense them. All referenced software (LineageOS, the TrebleDroid/phh GSIs, TWRP, Android
platform-tools) remains the property of its respective projects and retains its original
licensing terms. Refer to each upstream project for its license.

## Update 2026-11-11
A-only `agS` GSI flashed, boots fine (boot_completed=1, Wi-Fi/BT OK). See [docs/agS-flash-2026-11-11.md](docs/agS-flash-2026-11-11.md).
