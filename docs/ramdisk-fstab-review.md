# ramdisk / fstab review — HTC Desire 12+ (`brepdugl`) Android 11 FBE

**Author:** boot-image/fstab engineer (delegated agent)
**Date:** 2026-11-11 (Asia/Singapore)
**Scope:** inspect the STOCK Android 8.0 ramdisk + boot.img, reconcile with the CURRENT on-device state,
and produce the concrete final `/data` config for Android 11 FBE.

---

## 0. Files actually inspected (evidence base)

| Artifact | Path | What it is |
|---|---|---|
| Stock boot image | `C:\Users\iMac2019\brepdugl\boot.img` (64 MB) | stock kernel 3.18.71 + Android 8.0 ramdisk |
| Extracted ramdisk | `C:\Users\iMac2019\brepdugl\ramdisk\` | 55 cpio entries, extracted by `extract_ramdisk.py` |
| Boot logcat | `C:\Users\iMac2019\brepdugl\logcat_boot.txt` (16 423 lines) | **the current on-device boot**, best evidence of the live fstab |
| Prior write-up | `brepdugl-lineageos18.1-gsi\README.md` | previous session's problem→fix log |
| Handover | `HANDOVER.md` | current-state summary |

> **Important limitation up front:** the STOCK boot ramdisk contains **no fstab at all**. The fstab lives on the
> `/vendor` partition (`/vendor/etc/fstab.qcom`) and the vendor partition is **not dumped locally** — there is no
> `vendor.img` in `C:\Users\iMac2019\brepdugl\`. Therefore I cannot quote the *byte-exact original* vendor fstab.
> What I *can* quote exactly is (a) the boot.img header/cmdline, (b) the ramdisk `init.rc`/`ueventd.rc`, and
> (c) the **actual live mount sequence from `logcat_boot.txt`**, which reveals the real on-device fstab sources and
> options as the kernel/fs_mgr executed them. I flag every place where the task prompt's assumptions diverge from
> the captured evidence.

---

## 1. boot.img structure (parsed from the header)

Parsed with the existing `unpack_boot.py` plus a full header dump (offsets are the Android boot header v0 layout):

```
magic        : b'ANDROID!'
kernel_size  : 24148729  bytes        kernel_addr   0x80008000
ramdisk_size : 1882934   bytes        ramdisk_addr  0x81000000
second_size  : 0                       second_addr   0x80f00000
tags_addr    : 0x80000100
page_size    : 2048
header field @40 (v0=dt_size / v1=header_version) : 0   -> header v0 (Android 8.0 style)
os_version   : 268435746  (a=0, b=2, c=0x22 -> Android 8.0.0)
name         : 16 zero bytes
cmdline      : console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom msm_rtb.filter=0x237 ehci-hcd.park=3 lpm_levels.sleep_disabled=1 androidboot.bootdevice=7824900.sdhci earlycon=msm_hsl_uart,0x78af000 buildvariant=user
ramdisk      : gzip (magic 1f8b) -> decompresses to 4 315 904 bytes of cpio "newc", 55 entries
```

Key deductions from the cmdline:

- **`androidboot.hardware=qcom`** → `ro.hardware = qcom`, so init imports
  `/init.qcom.rc` and `/vendor/etc/init/hw/init.qcom.rc`.
- **`androidboot.bootdevice=7824900.sdhci`** → boot-device path `/dev/block/platform/soc/7824900.sdhci`,
  the parent of the `by-name/` symlinks.
- **No `skip_initramfs`**, and the ramdisk contains a normal Android 8.0 `init`/`init.rc` (not a recovery
  combined root) → **system-as-system (NOT system-as-root)**. Confirmed A-only / non-SAR. The ramdisk is
  **gzip**, not lz4.

---

## 2. Stock ramdisk contents and what init actually does

`glob **/*` + a recursive listing of `C:\Users\iMac2019\brepdugl\ramdisk\` show **55 entries and NO `fstab*`
file**. The text `.rc` files present are:

```
init.rc (28 763 B)      init.carrier.rc    init.environ.rc    init.recovery.qcom.rc
init.usb.rc             init.usb.configfs.rc  init.zygote32.rc  init.zygote64_32.rc
ueventd.rc              default.prop (symlink -> system/etc/prop.default)
```

There is **no `init.qcom.rc`, no `init.target.rc`, no `init.${ro.hardware}.rc`, no `ueventd.qcom.rc` in the
boot ramdisk** — those live on `/vendor`. This is normal for a Treble A-only device: the boot ramdisk only
carries the generic AOSP init, and the vendor partition carries the device fstab + platform rc files.

`init.rc` (quoted exactly) imports the platform rc files:

```
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc              # -> /init.qcom.rc  (NOT present in ramdisk, skipped silently)
import /vendor/etc/init/hw/init.${ro.hardware}.rc   # -> /vendor/etc/init/hw/init.qcom.rc  (vendor)
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc                # -> /init.zygote64_32.rc (this is a 64/32 device)
```

`init.rc` has **no `mount_all` line**. `mount_all /vendor/etc/fstab.qcom` is executed by the vendor rc, and the
live logcat pins it down:

```
01-01 01:01:18.350  I init : Command 'mount_all /vendor/etc/fstab.qcom' action=fs (/vendor/etc/init/hw/init.target.rc:50) returned 0 took 117.695ms.
```

So the mount flow is exactly: `init.rc` → `on late-init` → `trigger fs` → **`/vendor/etc/init/hw/init.target.rc`
`on fs` → `mount_all /vendor/etc/fstab.qcom`**. This is the Android 8.0 `fs_mgr` doing the mounts.

`init.rc` also contains the encryption-state branches (quoted exactly):

```
on zygote-start && property:ro.crypto.state=unencrypted
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary
on zygote-start && property:ro.crypto.state=unsupported
    ... (same body)
on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    ... (same body)
...
on nonencrypted
    class_start main
    class_start late_start
```

and the `/data` post-fs block (quoted exactly):

```
on post-fs-data
    chown system system /data
    chmod 0771 /data
    restorecon /data
    start vold
    installkey /data
    ...
    restorecon --recursive --skip-ce /data
```

(`installkey /data` and `restorecon --recursive --skip-ce /data` are the Android 8.0 FBE hooks.)

---

## 3. Mount entries — what the device ACTUALLY mounts (quoted from `logcat_boot.txt`)

The current boot (the "already applied" state) mounts the following via `mount_all /vendor/etc/fstab.qcom`.
`[libfs_mgr]__mount(source=...)` prints the **fstab's own source string**, so this is a faithful trace of the
current on-device fstab:

| # | source (as in fstab) | target | real dev | type | mount opts (kernel) | log line |
|---|---|---|---|---|---|---|
| 1 | `/dev/block/platform/soc/7824900.sdhci/by-name/system` | `/system` | mmcblk0p24 | ext4 | `barrier=1,discard` (no journal) | 959–960 |
| 2 | `/dev/block/platform/soc/7824900.sdhci/by-name/vendor` | `/vendor` | mmcblk0p25 | ext4 | `barrier=1,discard` | 961–962 |
| 3 | `/dev/block/bootdevice/by-name/userdata` | `/data` | **mmcblk0p53** | ext4 | `barrier=1,noauto_da_alloc` | 1461–1465 |
| 4 | `/dev/block/bootdevice/by-name/cache` | `/cache` | mmcblk0p26 | ext4 | `barrier=1` | 1466–1471 |
| 5 | `/dev/block/bootdevice/by-name/modem` | `/firmware` | mmcblk0p1 | vfat | (mountpoint labeling) | 1472–1473 |
| 6 | `/dev/block/bootdevice/by-name/dsp` | `/dsp` | mmcblk0p12 | ext4 | `barrier=1` | 1474–1478 |
| 7 | `/dev/block/bootdevice/by-name/cota` | `/cota` | mmcblk0p51 | ext4 | `barrier=1` | 1479–1484 |

Exact log lines for the three entries the task asks about:

```
959:  EXT4-fs (mmcblk0p24): mounted filesystem without journal. Opts: barrier=1,discard
960:  init: [libfs_mgr]__mount(source=/dev/block/platform/soc/7824900.sdhci/by-name/system,target=/system,type=ext4)=0
961:  EXT4-fs (mmcblk0p25): mounted filesystem with ordered data mode. Opts: barrier=1,discard
962:  init: [libfs_mgr]__mount(source=/dev/block/platform/soc/7824900.sdhci/by-name/vendor,target=/vendor,type=ext4)=0

1461: init: [libfs_mgr]superblock s_max_mnt_count:65535,/dev/block/bootdevice/by-name/userdata
1462: init: [libfs_mgr]Requested quota status is match on /dev/block/bootdevice/by-name/userdata
1463: EXT4-fs (mmcblk0p53): mounted filesystem with ordered data mode. Opts: barrier=1,noauto_da_alloc
1464: SELinux : initialized (dev mmcblk0p53, type ext4), uses xattr
1465: init: [libfs_mgr]__mount(source=/dev/block/bootdevice/by-name/userdata,target=/data,type=ext4)=0

1474: init: [libfs_mgr]superblock s_max_mnt_count:65535,/dev/block/bootdevice/by-name/dsp
1476: EXT4-fs (mmcblk0p12): mounted filesystem with ordered data mode. Opts: barrier=1
1478: init: [libfs_mgr]__mount(source=/dev/block/bootdevice/by-name/dsp,target=/dsp,type=ext4)=0

1479: init: [libfs_mgr]superblock s_max_mnt_count:10,/dev/block/bootdevice/by-name/cota
1482: EXT4-fs (mmcblk0p51): mounted filesystem with ordered data mode. Opts: barrier=1
1484: init: [libfs_mgr]__mount(source=/dev/block/bootdevice/by-name/cota,target=/cota,type=ext4)=0
```

### Discrepancy to note (task assumption vs evidence)

The task brief states *"/dsp and /cota in the original fstab use direct mmcblk0pN nodes."* The **live logcat
contradicts this for the current state**: `/dsp` and `/cota` are mounted through
`/dev/block/bootdevice/by-name/dsp` and `.../cota` (i.e. ueventd symlinks, **not** direct `mmcblk0pN`), exactly
like `/data` and `/cache`. And crucially, **`/data` also mounts successfully via `by-name/userdata` at `on fs`**
in this log (line 1465 = 0, and `mount_all` returned 0). I cannot see the *original* stock fstab (vendor not
dumped), but the current on-device fstab demonstrably uses `by-name` for dsp/cota/userdata/cache and the full
`platform/soc/.../by-name/...` path for system/vendor.

### Original `/data` fstab options

From the prior write-up (`README.md`, "Problem 6"), the stock `/data` line carried **`forceencrypt=footer`**
(FDE) plus `check`, `quota`, `discard`. The current on-device line (same source) is:

```
/dev/block/bootdevice/by-name/userdata  /data  ext4  noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait
```

I could not retrieve the byte-exact original `/system` and `/vendor` lines; the live trace above is the closest
available ground truth and matches the handover's partition map (system=mmcblk0p24, vendor=mmcblk0p25).

### The encryption-state evidence (the crux)

```
1740: I init : processing action (ro.crypto.state=unsupported && zygote-start)
2878: I init : processing action (nonencrypted)
```

`ro.crypto.state=unsupported` means the Android 8.0 `fs_mgr` found **no `forceencrypt=` and no `fileencryption=`
flag** in the `/data` fstab entry, so it classified `/data` as "encryption not supported" and fired the
`nonencrypted` path. Meanwhile the Android 11 `vold` is simultaneously trying to do FBE:

```
1594: D vold : Vold 3.0 (the awakening) firing up
1606: D vold : fscrypt_init_user0
1607: D vold : fscrypt_prepare_user_storage for volume null, user 0, serial 0, flags 1
1653: D vold : fscrypt_unlock_user_key 0 serial=0 token_present=0
1654: E vold : Failed to chmod /data/media/0: No such file or directory
```

i.e. the two halves of this Frankenstein boot disagree: the Android 8.0 init says "unsupported/unencrypted",
the Android 11 vold expects FBE and prepares fscrypt storage but has no `fileencryption=` directive to apply.
This is the mismatch the config below is meant to resolve.

---

## 4. Answers

### Q1 — Drop the direct `on fs` mount; mount `/data` via fstab only.

**Recommendation: fstab-only. Remove the appended `on fs` block from `/vendor/etc/init/hw/init.target.rc`.**

Justification:

1. **It is now redundant.** The live log proves `mount_all /vendor/etc/fstab.qcom` already mounts `/data`
   successfully through `by-name/userdata` (line 1465 = 0). The earlier "/data never mounts" failure was
   almost certainly caused by `forceencrypt=footer` (FDE) and/or `check` (e2fsck) on the old line — not by the
   `by-name` symlink (see Q2). Once those flags were stripped, fstab mounting works. The direct `mount` line is
   a second mount of an already-mounted filesystem.
2. **It fights vold.** Android 11 expects **vold** to own the `/data` volume. A hand-rolled `init` `mount` leaves
   vold's internal `VolumeInfo` in an inconsistent state — the log literally shows
   `fscrypt_prepare_user_storage for volume null` ("null", not `emulated`). This is candidate cause #2 in the
   handover and is exactly the kind of thing that can trip the factory-reset path.

**Exact fstab.qcom `/data` line (recommended):**

```
/dev/block/mmcblk0p53  /data  ext4  noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait,quota,fileencryption=aes-256-xts
```

Fields: `<source>  <mountpoint>  <fs>  <mount opts>  <fs_mgr flags>` (whitespace-delimited; spacing is cosmetic).

- **source = `/dev/block/mmcblk0p53`** (direct node, see Q2).
- **mount opts** kept identical to the currently-working mount (`barrier=1,noauto_da_alloc`, plus the standard
  `noatime,nosuid,nodev`). Do **not** re-add `discard` (this eMMC has a known TRIM/discard state issue, problem 1).
- **flags = `wait,quota,fileencryption=aes-256-xts`**.
  - `fileencryption=aes-256-xts` is the FBE (file-based encryption / fscrypt v1) flag. **This is what makes
    Android 11 vold actually enable FBE** instead of treating `/data` as unencrypted.
  - `quota` keeps project-quota active (the filesystem was already formatted `-O quota,project`, and the log
    shows `Requested quota status is match`, i.e. the fs_mgr quota check is happy).
  - `wait` = wait for the block node before mounting (harmless, keep it).
- **Do NOT use `encryptable=footer`.** `encryptable=footer` / `forceencrypt=footer` are **FDE (full-disk,
  dm-crypt)** flags. Android 11 (vold 3.0) **removed FDE entirely** — only FBE (`fileencryption=…`) is supported.
  The research agent's `encryptable=footer` suggestion was wrong for Android 11; it is an Android-8.0-era
  "let it boot unencrypted but stay FDE-capable" idea. For Android 11 FBE the correct token is `fileencryption=`.
- **cipher choice — `aes-256-xts`, not `adiantum`.** Snapdragon 450 (MSM8937) is 8× Cortex-A53 with **no ARMv8
  Crypto Extensions**, which is precisely the CPU class Google's
  [Adiantum doc](https://source.android.com/docs/security/features/encryption/adiantum) targets. **However,**
  Adiantum requires kernel support (`CONFIG_CRYPTO_ADIANTUM` + fscrypt v2 / `FS_IOC_ADD_ENCRYPTION_KEY`) that
  landed in Linux 5.0 and was only backported to Android common kernels 4.9/4.14/4.19. This device's **stock
  3.18.71 (Android 8.0) kernel predates Adiantum and will not have it.** `aes-256-xts` is the fscrypt-v1 default,
  present in every Android 7.0+ kernel (incl. 3.18), and works on Cortex-A53 in software (slower than Adiantum
  would be, but correct and supported). So: `aes-256-xts` for this kernel. If you ever move to a 4.9+ kernel,
  switch to `fileencryption=adiantum`.

**Minimal fallback (only if the Android 8.0 fs_mgr rejects `quota`):**

```
/dev/block/mmcblk0p53  /data  ext4  noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait,fileencryption=aes-256-xts
```

(And to *diagnose* boot-loop vs FBE, a no-encryption line is just the same line with `fileencryption=…`
removed — which is what's on the device today.)

### Q2 — Avoid the `by-name` `wait`; use the direct `mmcblk0p53` node.

**Answer: yes — use `/dev/block/mmcblk0p53`, and you don't need the `wait /dev/block/bootdevice/by-name/userdata`.**

Two separate points, stated precisely:

1. **The "by-name symlink not ready at `on fs`" theory is contradicted by the evidence.** ueventd is started in
   `on early-init` and creates `/dev/block/bootdevice/by-name/*` (and `/dev/block/platform/soc/7824900.sdhci/by-name/*`)
   long before `on fs` runs (from `on late-init`). The current logcat shows **every** `by-name` mount —
   system, vendor, userdata, cache, dsp, cota — succeeding *inside* the single `mount_all … action=fs`. So the
   `by-name/userdata` symlink **is** ready at `on fs`. The handover's "symlink too late" root-cause was a
   misdiagnosis; the actual blocker was the old `forceencrypt=footer`/`check` flags.
2. **Still, the direct node is the more robust choice** (belt-and-suspenders), because `/dev/block/mmcblk0p53`
   is the kernel-created block node and has **no dependency at all** on ueventd's partition-table symlink
   generation, GPT `PARTNAME` handling, or the `bootdevice` symlink. It removes one failure mode for free and
   matches what the research agent (correctly) recommended. There is no downside on an A-only fixed-partition
   device where `userdata == mmcblk0p53` is stable.

So: **fstab source = `/dev/block/mmcblk0p53`**, and **delete** the appended
`wait /dev/block/bootdevice/by-name/userdata` + direct `mount` block entirely.

### Q3 — Reformat `/data` for FBE, and relabel.

**Exact reformat (from TWRP shell):**

```sh
mke2fs -t ext4 -O encrypt,quota,project /dev/block/mmcblk0p53
```

- `encrypt`  → enables ext4 **fscrypt** (per-file encryption) — required for FBE.
- `quota`, `project` → enable project quota (Android 11 vold uses project IDs for FBE key scoping).
- `-t ext4` on the **direct node**. (This is the same command already applied once; it is still correct. Add
  `-F` only if `mke2fs` complains the device is in use/unmounted.) Do **not** use `-O encrypt` alone; the
  quota+project features are what Android 11 FBE expects alongside `encrypt`.

**Relabel (SELinux):**

```sh
# from a rooted ADB shell AFTER the GSI boots (uses the Android 11 file_contexts):
restorecon -RF /data

# equivalent init builtin already run by init on first boot:
#   restorecon --recursive --skip-ce /data
```

- `restorecon -RF /data` = force re-label recursively (`-R` recursive, `-F` force-reset). Toybox/busybox on
  Android accept this form.
- **Do it from Android, not TWRP.** TWRP's `restorecon` uses TWRP's own (recovery) file_contexts, which is the
  wrong policy. The boot init compiles the Android 11 `plat_sepolicy.cil` + vendor `nonplat_sepolicy.cil`
  (log lines 964–982) and then `post-fs-data` runs `restorecon /data` + `restorecon --recursive --skip-ce /data`
  on first boot, so a freshly-`mke2fs`'d `/data` is already labelled correctly during the first boot. Run the
  manual `restorecon -RF /data` only as a belt-and-suspenders once the GSI is up (or if `/data` was written by
  TWRP while mounted).

### Q4 — `/system/etc/fstab.qcom` (the GSI's fstab) and overrides.

**Answer: no change to `/system/etc/fstab.qcom` is needed — it is ignored. Fix `/vendor/etc/fstab.qcom`.**

`vold` (and `fs_mgr`) locate the fstab via `android::fs_mgr::GetFstabPath()`, which (Android 8.0–11) probes, in
order, using `fstab_suffix` / `hardware` / `hardware.platform` (`qcom` here):

```
/odm/etc/fstab.qcom
/vendor/etc/fstab.qcom     <-- found here first (odm fstab doesn't exist on this device)
/fstab.qcom
```

`/system/etc/fstab.qcom` is **not** in that list (it was removed because first-stage init runs before `/system`
is mounted). So the GSI's copy at `/system/etc/fstab.qcom` is dead weight — vold always reads the vendor copy
first and stops. The only file that matters is **`/vendor/etc/fstab.qcom`**.

The one override you *must* make is therefore in `/vendor/etc/fstab.qcom` (the `fileencryption=` line from Q1),
plus removing the appended `on fs` block from `/vendor/etc/init/hw/init.target.rc`. Keep `/system/etc/fstab.qcom`
in sync if you want tidiness, but it has **no functional effect**.

### Q5 — Ordered on-device steps (next TWRP session; `fastboot` is 32-bit, so `adb push` + `dd`).

Everything below is run with the 64-bit `adb.exe`
(`C:\Users\iMac2019\brepdugl\platform-tools-x64\platform-tools\adb.exe`); never use the 32-bit tool for >2 GB.

```sh
# 0) Reach TWRP (device is in the reset loop -> catch it first)
#    - on the "Try again / Factory data reset" screen tap "Try again"
#    - run adb_race.py in the background to fire `adb reboot bootloader` when adb appears
#    - then boot TWRP:
fastboot boot twrp-3.7.0_9-0-brepdugl.img

# 1) Mount /vendor read-write (vendor = mmcblk0p25)
adb shell mount -t ext4 /dev/block/mmcblk0p25 /vendor

# 2) Point the /data fstab line at the direct node and add FBE (single sed, in place)
adb shell "sed -i 's|^/dev/block/[^ ]*userdata.*|/dev/block/mmcblk0p53  /data  ext4  noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait,quota,fileencryption=aes-256-xts|' /vendor/etc/fstab.qcom"

# 3) REMOVE the appended direct-mount block from /vendor/etc/init/hw/init.target.rc
#    (delete exactly these lines, leave the original `on fs` -> mount_all section intact):
#        on fs
#            wait /dev/block/bootdevice/by-name/userdata
#            mount ext4 /dev/block/bootdevice/by-name/userdata /data noatime nosuid nodev barrier=1
adb shell "sed -i '/wait \/dev\/block\/bootdevice\/by-name\/userdata/d; /mount ext4 \/dev\/block\/bootdevice\/by-name\/userdata \/data/d' /vendor/etc/init/hw/init.target.rc"

# 4) (Optional) mirror the vendor fstab into the GSI so both files agree (cosmetic only)
adb shell "cp /vendor/etc/fstab.qcom /system/etc/fstab.qcom"

# 5) Reformat /data for FBE (DESTROYS all /data)
adb shell "umount /data"
adb shell "mke2fs -t ext4 -O encrypt,quota,project /dev/block/mmcblk0p53"

# 6) Verify the fstab edit before rebooting
adb shell "grep -n userdata /vendor/etc/fstab.qcom"
adb shell "grep -n 'fileencryption' /vendor/etc/fstab.qcom"

# 7) Reboot into Android
adb reboot

# 8) After the GSI comes up (root shell), force-relabel /data with the Android 11 contexts
adb root
adb shell "restorecon -RF /data"
```

Notes:

- **Step 2 sed** matches any existing `/dev/block/...userdata...` line and rewrites it in place; safe against
  the source already being `by-name` vs a full path. Always re-check with step 6.
- **Step 3 sed** deletes the two direct-mount lines. If your editor/`sed` mangling is a concern, pull the file,
  edit, and `adb push` it back instead — but the `dd`/`sed` approach above avoids host-side line-ending issues.
- `dd` is **not** needed here (no image write). If you *do* re-flash the GSI, remember busybox `dd` rules from
  problem 4: plain byte count `bs=1048576` (no `4M`), no `conv=notrunc`.
- The reformat in step 5 uses the **direct node** `mmcblk0p53`; the `-O encrypt,quota,project` features are what
  Q3 requires.

---

## 5. Bottom line

1. **Remove the direct `on fs` mount; go fstab-only.** `mount_all /vendor/etc/fstab.qcom` already mounts `/data`
   (proven by the log); the direct mount is redundant and fights vold.
2. **Use the direct node `/dev/block/mmcblk0p53`**, not `by-name`. The `by-name` symlink is actually ready at
   `on fs` (log-proven), but the direct node is strictly more robust. The `wait /dev/block/bootdevice/by-name/userdata`
   line should go.
3. Exact `/data` fstab line:
   `noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait,quota,fileencryption=aes-256-xts` on
   `/dev/block/mmcblk0p53`. **`encryptable=footer` is FDE and is wrong; use `fileencryption=`.** Choose
   **`aes-256-xts`** (fscrypt v1, works on the 3.18 kernel) over `adiantum` (needs a 4.9+/5.0 kernel).
4. `/system/etc/fstab.qcom` is ignored by vold (`GetFstabPath` reads `/odm`→`/vendor`→`/fstab.`); no override
   needed there — fix `/vendor/etc/fstab.qcom`.
5. Reformat `mke2fs -t ext4 -O encrypt,quota,project /dev/block/mmcblk0p53`, relabel with `restorecon -RF /data`
   from the GSI (not TWRP), and apply the whole thing from TWRP via the Q5 sequence.

**Honest caveat:** this is the *correct FBE configuration*, but it is **not yet proven** to stop the
`boot_completed=1 → factory-reset` loop — that loop's trigger was never isolated (logcat capture ended before
the reset; there was no `FATAL EXCEPTION` in system_server and no RescueParty line). The config above removes
the known vold/init `/data` conflict and enables real FBE, which is the most likely fix; if the loop persists,
the next step is a longer logcat capture focused on `vold`/`fscrypt`/`StorageManagerService` immediately before
the reset.
