# Independent Verifier — Factory-Reset Loop Root Cause & Test Plan
**Device:** HTC Desire 12+ (`brepdugl`), MSM8937, kernel 3.18.71, Treble ARM64 A-only.
**GSI:** `lineage-18.1-20240121-UNOFFICIAL-arm64_agS.img` (Andy Yan / TrebleDroid, Android 11).
**Verified by:** independent tester — re-read `HANDOVER.md`, `logcat_boot.txt` (2.1 MB), `console-ramoops.txt` (pstore, 262 KB), and the stock ramdisk. No subagents used.

---

## 1. Independent root-cause assessment

**Conclusion (one paragraph):** The "factory data reset" loop is **Android 11's updatable-process crash recovery**, triggered by the LineageOS **`vendor.livedisplay-hal-2-0-legacymm`** HAL, which fails its SELinux domain transition and exits `status 1` every ~5 s forever. Because init classifies that service as an **"updatable process"**, the 4th exit in a boot makes init set **`ro.init.updatable_crashing=1`**. That property is the documented Android 11 signal for "an APEX/updatable module is crashing"; apexd then logs *"Updatable crashing, attempting rollback"* and *"Rollback failed … no active sessions"*, and `flags_health_check UPDATABLE_CRASHING` runs. This is the well-known Android 11 recovery path that escalates to a **factory reset** (the `--prompt_and_wipe_data` → recovery "Try again / Factory data reset" screen). It re-occurs every boot because the HAL crashes every boot, so the loop is persistent. The handover's logcat simply **ended at boot phase 1000** (before the ~1-min reset), which is why no "RescueParty"/reset line was captured there — but the pstore ramoops *does* capture the trigger. The Bluetooth crash and the vold `/data/media/0` errors are **symptoms of a secondary problem (CE/DE storage + SELinux label), not the reset trigger**.

### Evidence lines

**From `console-ramoops.txt` (pstore kernel log — this is the boot that ended in the reset):**
```
[   22.941384] init: File /vendor/bin/hw/vendor.lineage.livedisplay@2.0-service-legacymm (labeled "u:object_r:vendor_file:s0") has incorrect label or no domain transition from u:r:init:s0 to another SELinux domain defined. Have you configured your service correctly? https://source.android.com/security/selinux/device-policy#label_new_services_and_address_denials
[   22.978808] init: Service 'vendor.livedisplay-hal-2-0-legacymm' (pid 1126) exited with status 1
[   32.995347] init: updatable process 'vendor.livedisplay-hal-2-0-legacymm' exited 4 times before boot completed
[   32.996299] apexd: Updatable crashing, attempting rollback
[   32.996571] apexd: Rollback failed : Rollback requested, when there are no active sessions.
[   32.998393] init: processing action (ro.init.updatable_crashing=1) from (/system/etc/init/flags_health_check.rc:10)
[   32.999136] init: starting service 'exec 11 (/system/bin/flags_health_check UPDATABLE_CRASHING)'...
```
The same `updatable process 'vendor.livedisplay-hal-2-0-legacymm' exited 4 times before boot completed` block repeats every ~5 s for the entire capture (last at `[ 774.066540]`, ~12.9 min) — a continuous, runaway crash loop.

**From `logcat_boot.txt` (android log — boot reached setup wizard, capture ended at phase 1000):**
```
01-01 01:01:07.857   qcom,qpnp-power-on: PMIC@SID0: Power-off reason: Triggered from PS_HOLD (PS_HOLD/MSM controlled shutdown)
01-01 01:01:14.157 E init    : /system/etc/init/apexd.rc: 2: invalid keyword 'interface'
01-01 01:01:14.157 E init    : /system/etc/init/apexd.rc: 8: invalid keyword 'reboot_on_failure'
01-01 01:01:17.837 I apexd   : This device does not support updatable APEX. Exiting
01-01 01:01:25.884 E vold    : Failed to chmod /data/media/0: No such file or directory
01-21 09:45:55.157 E SQLiteDatabase: android.database.sqlite.SQLiteCantOpenDatabaseException: Cannot open database '/data/user_de/0/com.android.bluetooth/databases/bluetooth_db': Directory /data/user_de/0/com.android.bluetooth/databases doesn't exist
01-01 01:01:41.979 W system_server: avc: denied { write } for name="/" dev="mmcblk0p53" ino=2 scontext=u:r:system_server:s0 tcontext=u:object_r:system_data_root_file:s0 tclass=dir permissive=0
01-21 09:46:03.694 E vold    : Failed to chmod /data/media/0: No such file or directory
01-21 09:46:03.709 I StorageManagerService: Resetting vold...  ->  Reset vold
```
No `RescueParty`, no `FATAL EXCEPTION`, no reset line — the capture ends at `01-21 09:46:03.750` (boot phase 1000, `sys.boot_completed=1`, SetupWizard running).

### Agree / disagree with the handover's 4 candidate causes

| # | Handover candidate | Verdict | Reason |
|---|---|---|---|
| 1 | vold FBE (fscrypt) setup failing | **DISAGREE** (as the reset trigger) | vold 3.0 *succeeded*: `fscrypt_init_user0`, `fscrypt_prepare_user_storage`, `fscrypt_unlock_user_key 0` all ran cleanly. Only minor errors (`Failed to chmod /data/media/0`) — there is **no** "decryption unsuccessful" / fscrypt-failure in the log. |
| 2 | Direct mount vs vold conflict | **PARTIALLY AGREE** (contributor, not trigger) | Real bug: init mounts `/data` directly, so vold never owns the volume; the `/data` root keeps label `system_data_root_file` → enforcing SELinux denial `avc: denied { write } … mmcblk0p53`. This causes the secondary CE/DE/Bluetooth symptoms but does **not** itself reset the device. |
| 3 | Rescue party from system_server/zygote restarts | **AGREE in mechanism, WRONG trigger** | It *is* Android 11's rescue/recovery, but the trigger is **not** system_server/zygote restarts — it is the **"updatable process" `vendor.livedisplay-hal-2-0-legacymm`** crashing, setting `ro.init.updatable_crashing=1`. This was invisible in the handover's logcat because the capture stopped at phase 1000; the pstore ramoops exposes it. |
| 4 | CE/DE (user_de) setup incomplete | **AGREE as symptom, NOT cause** | `/data/user_de/0/com.android.bluetooth/…` missing → Bluetooth `SQLiteCantOpenDatabaseException` (the only app crash). This is downstream of the direct-mount/vold conflict and wrong SELinux labels; it does not trigger the reset. |

---

## 2. Risk-ranked hypothesis list (most-likely first)

1. **(~80%) `ro.init.updatable_crashing=1` → updatable-crash recovery → factory reset.** The `vendor.livedisplay-hal-2-0-legacymm` HAL crashes in a loop (SELinux domain-transition failure); 4 exits set the updatable-crashing flag; apexd can't roll back. *Test it:* on next boot run `adb shell getprop ro.init.updatable_crashing` and `adb shell dmesg | grep -iE "updatable|livedisplay"` — expect `1` and the crash loop.
2. **(~12%) Rescue Party (Package Watchdog) escalation from the persistent `com.android.bluetooth` crash.** Bluetooth (a `FLAG_PERSISTENT` app) crashes repeatedly because its CE dir is missing. *Test it:* capture `adb logcat -b all | grep -iE "RescueParty|PackageWatchdog|boot loop"` through a reset.
3. **(~5%) apexd/`flags_health_check` reboot loop because the stock Android 8.0 init mis-parses Android 11 rc syntax.** `apexd.rc` lines `interface`/`reboot_on_failure` and `bootanim.rc` `task_profiles` are `invalid keyword` to the 8.0 init, so the 8.0 init runs a half-configured Android 11 system. *Test it:* grep dmesg for the `invalid keyword` errors (already present) and confirm apexd exits "does not support updatable APEX".
4. **(~3%) vold/direct-mount conflict → wrong `/data` SELinux context → StorageManagerService `Resetting vold` at phase 1000.** *Test it:* `adb shell ls -Zd /data` (expect `system_data_root_file` instead of `system_data_file`), `adb shell getprop ro.crypto.state` (expect `unencrypted`), and capture the `Resetting vold` sequence.

> Hypotheses 2–4 are **secondary** and should be addressed *after* #1 is confirmed fixed. #1 is the one that explains the ~1-min reset with hard evidence.

---

## 3. Exact ordered fix commands (copy-paste ready)

### Environment (Windows host)
```powershell
# Use the 64-bit tools for everything (the 32-bit SDK adb/fastboot std::bad_alloc on >2GB buffers):
$ADB  = 'C:\Users\iMac2019\brepdugl\platform-tools-x64\platform-tools\adb.exe'
$FASTBOOT = 'C:\Users\iMac2019\brepdugl\platform-tools-x64\platform-tools\fastboot.exe'
$TWRP = 'C:\Users\iMac2019\brepdugl\twrp-3.7.0_9-0-brepdugl.img'
$PY   = 'C:\Python315\python.exe'
$WS   = 'E:\Users\Ninja hp LAPTOP\Documents\deepseek-harness\default-workspace'
```
> `fastboot boot twrp.img` is used (not `fastboot flash`) — avoids the >2 GB buffering problem entirely; TWRP then exposes a reliable `adb`.

### Step 0 — get into TWRP (the ADB race, then `fastboot boot`)
```powershell
# Terminal A (leave running). On the recovery screen tap "Try again"; this catches the brief adb 'device' window
# and auto-issues `adb reboot bootloader`.
& $PY "$WS\adb_race2.py"

# When it prints "FASTBOOT READY", boot TWRP (network over USB):
& $FASTBOOT boot $TWRP
# wait ~20-40s until `adb devices` shows TWRP as "device" (TWRP's own adbd).
```

### Hypothesis 1 (PRIMARY) — disable the crashing livedisplay HAL + hard-disable rescue

**(a) Quick, lowest-risk, durable first shot (no /system edit needed):** set the Rescue Party kill-switch. Run during the brief boot adb window (before the ~1 min reset) *or* from TWRP's `/data`:
```powershell
# From adb (Andy Yan GSI is userdebug, so `adb root` works):
& $ADB root
& $ADB shell "setprop persist.sys.disable_rescue true; setprop persist.sys.enable_rescue false"
& $ADB shell "getprop persist.sys.disable_rescue"     # expect: true
```
```powershell
# From TWRP (guaranteed, survives any reset because it is written before one can happen):
& $ADB shell "mount /data 2>/dev/null; mkdir -p /data/property; printf true > /data/property/persist.sys.disable_rescue; printf false > /data/property/persist.sys.enable_rescue; chmod 600 /data/property/persist.sys.*; ls -l /data/property/persist.sys.*"
```
> `persist.sys.disable_rescue` is confirmed present in AOSP `RescueParty.java` (Android 8.0 through 11: `PROP_DISABLE_RESCUE = "persist.sys.disable_rescue"`). It also has `PROP_DEVICE_CONFIG_DISABLE_FLAG = "persist.device_config.configuration.disable_rescue_party"` in Android 11 — set that too for belt-and-suspenders:
```powershell
& $ADB shell "printf true > /data/property/persist.device_config.configuration.disable_rescue_party 2>/dev/null; chmod 600 /data/property/persist.device_config.configuration.disable_rescue_party 2>/dev/null"
```

**(b) The real fix — stop the HAL from starting (do this from TWRP so /system is writable):**
```powershell
# Mount system (and vendor if it is a separate partition) read-write:
& $ADB shell "mount -o rw /system; mount -o rw /vendor 2>/dev/null; mount -o rw /system/vendor 2>/dev/null; echo READY"

# Locate the service declaration and the binary:
& $ADB shell "find /system /vendor -iname '*livedisplay*' 2>/dev/null; echo '--- rc files ---'; grep -rln 'vendor.livedisplay-hal-2-0-legacymm' /system/etc/init /vendor/etc/init /system/vendor/etc/init 2>/dev/null"
```
Expected output gives you the rc path (typically something like
`/system/vendor/etc/init/vendor.lineage.livedisplay@2.0-service.legacymm.rc`
or `/vendor/etc/init/vendor.lineage.livedisplay@2.0-service.legacymm.rc`).
Substitute that path for `$RC` below and run:

```powershell
# Backup, then append the 'disabled' keyword to the service line (idempotent, safe with toybox sed):
& $ADB shell 'RC=/system/vendor/etc/init/vendor.lineage.livedisplay@2.0-service.legacymm.rc; cp "$RC" "$RC.orig"; sed -i "s|^service[[:space:]]*vendor\.livedisplay-hal-2-0-legacymm.*|& disabled|" "$RC"; echo "--- result ---"; grep -n livedisplay "$RC"'
```
```powershell
# Fallback if sed is too fiddly: rename the whole rc so init no longer imports it (harmless "could not import" warning if imported explicitly).
& $ADB shell 'RC=/system/vendor/etc/init/vendor.lineage.livedisplay@2.0-service.legacymm.rc; mv "$RC" "$RC.disabled"; ls -l /system/vendor/etc/init/ | grep livedisplay'
```
```powershell
# Also rename the binary as a belt-and-suspenders (in case another rc re-declares the service):
& $ADB shell "mv /system/vendor/bin/hw/vendor.lineage.livedisplay@2.0-service-legacymm /system/vendor/bin/hw/vendor.lineage.livedisplay@2.0-service-legacymm.bak 2>/dev/null; ls -l /system/vendor/bin/hw/ | grep livedisplay"
```
> **Important:** renaming the binary *alone* is **not sufficient** — init would still try to exec it, `execvpe()` fails, and that exit still counts toward "exited 4 times" and still sets `ro.init.updatable_crashing=1`. The rc edit (or rc rename) is the effective step; the binary rename is only extra insurance.

**(c) Reboot and verify (see §4).**

### Hypothesis 2 (secondary) — persistent Bluetooth app crash → Rescue Party
Once #1 is applied, if `adb logcat` still shows `RescueParty`/`PackageWatchdog` escalation, fix the CE dir and re-create the missing Bluetooth storage:
```powershell
& $ADB root
& $ADB shell "mkdir -p /data/user_de/0/com.android.bluetooth/databases /data/user_de/0/com.android.bluetooth/cache /data/user_de/0/com.android.bluetooth/code_cache /data/user_de/0/com.android.bluetooth/shared_prefs; chown -R 1002:1002 /data/user_de/0/com.android.bluetooth; restorecon -RF /data/user_de/0/com.android.bluetooth"
```

### Hypothesis 3 (secondary) — stock 8.0 init mis-parses Android 11 rc (`interface`, `reboot_on_failure`, `task_profiles`)
This is inherent to booting an Android 11 GSI with the stock Android 8.0 boot image. The only *clean* long-term fix is to **stop booting with the stock 8.0 ramdisk** (use a LineageOS 18.1 boot image for `brepdugl`, if one exists), or **patch the rc files** to drop the unsupported keywords. Patching `apexd.rc` on the GSI (low priority since apexd already self-disables):
```powershell
# From TWRP, mount /system rw, then strip the keywords the 8.0 init rejects:
& $ADB shell 'mount -o rw /system; sed -i "s/^[[:space:]]*interface[[:space:]].*//; s/^[[:space:]]*reboot_on_failure.*//" /system/etc/init/apexd.rc; sed -i "s/^[[:space:]]*task_profiles[[:space:]].*//" /system/etc/init/bootanim.rc; grep -nE "interface|reboot_on_failure|task_profiles" /system/etc/init/apexd.rc /system/etc/init/bootanim.rc'
```

### Hypothesis 4 (secondary) — direct-mount/vold conflict → wrong /data label + CE/DE incomplete
Only if you want full FBE and to fix the Bluetooth/SELinux symptoms *after* the reset loop is gone. This is the riskiest change (the direct mount is what finally mounted /data), so keep the rollback (§5) ready.
```powershell
# From TWRP: remove the appended direct "on fs" mount from init.target.rc (keep a backup),
# then make the fstab /data entry use the DIRECT node (avoids the ueventd by-name race) with FBE.
& $ADB shell '
mount -o rw /vendor
cp /vendor/etc/init/hw/init.target.rc /vendor/etc/init/hw/init.target.rc.orig
# remove the whole appended "on fs ... mount ext4 ... /data" block:
sed -i "/^on fs$/,/^[[:space:]]*mount ext4 .* \/data /d" /vendor/etc/init/hw/init.target.rc
grep -n "userdata" /vendor/etc/init/hw/init.target.rc
'
```
```powershell
# fstab: point /data at the direct mmcblk node + fileencryption (FBE). Adapt the exact existing line.
& $ADB shell '
mount -o rw /vendor
cp /vendor/etc/fstab.qcom /vendor/etc/fstab.qcom.orig
sed -i "s|^/dev/block/bootdevice/by-name/userdata .*|/dev/block/mmcblk0p53 /data ext4 noatime,nosuid,nodev,barrier=1 wait,fileencryption=aes-256-xts|" /vendor/etc/fstab.qcom
cat /vendor/etc/fstab.qcom
'
```
```powershell
# Recreate /data with the FBE features (this is the mke2fs the handover already used — re-run after the fstab change so vold owns first mount):
& $ADB shell "mke2fs -t ext4 -O encrypt,quota,project /dev/block/bootdevice/by-name/userdata"
```
> If `aes-256-xts` is unavailable on this 3.18 kernel's fscrypt backport, use `fileencryption=adiantum`. `adb shell cat /sys/fs/fscrypt/...` (or `/proc/crypto | grep -i adiantum`) tells you.
> **Block-device sanity:** the handover says system=mmcblk0p24, vendor=mmcblk0p25, userdata=mmcblk0p53, but the logcat shows `/system` on `mmcblk0p34`. Verify before touching fstab: in TWRP run `ls -l /dev/block/bootdevice/by-name/` and match `system`, `vendor`, `userdata` to real `mmcblk0pN`.

---

## 4. Verification — what confirms each fix worked

**Hypothesis 1 fixed — the decisive checks:**
```powershell
& $ADB shell "getprop ro.init.updatable_crashing"          # empty or "0", NOT "1"
& $ADB shell "getprop persist.sys.disable_rescue"          # "true"
& $ADB shell "getprop sys.boot_completed"                  # 1, and stays up > 5 min with NO reset
& $ADB shell "dmesg | grep -iE 'updatable|livedisplay'"    # expect NO new "updatable process ... exited 4 times"
& $ADB shell "ps -A | grep -i livedisplay"                 # expect empty (service gone)
& $ADB shell "getprop ro.init.updatable_crashing_process_name"  # empty
```

**Hypothesis 2 fixed:**
```powershell
& $ADB shell "logcat -d -b all | grep -iE 'RescueParty|PackageWatchdog|com.android.bluetooth'"
# expect: no RescueParty escalation; bluetooth no longer logs SQLiteCantOpenDatabaseException
```

**Hypothesis 3 fixed:**
```powershell
& $ADB shell "dmesg | grep -iE 'invalid keyword'"         # apexd.rc/bootanim.rc errors gone
```

**Hypothesis 4 fixed (FBE + vold):**
```powershell
& $ADB shell "getprop ro.crypto.state"                    # "encrypted" (was "unencrypted")
& $ADB shell "getprop ro.crypto.type"                     # "file"
& $ADB shell "ls -Zd /data"                               # u:object_r:system_data_file:s0 (not system_data_root_file)
& $ADB shell "ls -ld /data/media/0 /data/user_de/0/com.android.bluetooth"  # both exist
& $ADB shell "dmesg | grep -iE 'fscrypt|vold'"            # no 'Failed to chmod /data/media/0'
```

**Log capture commands (run these exactly):**
```powershell
# Long logcat through the reset (capture_logcat.py currently waits 360s; run a longer variant below):
& $PY -c "import subprocess,time; adb=r'C:\Users\iMac2019\brepdugl\platform-tools-x64\platform-tools\adb.exe'; subprocess.run([adb,'logcat','-b','all','-v','threadtime','>','C:\Users\iMac2019\brepdugl\logcat_full.txt'], shell=True)"
# (better: start a background stream that keeps writing past boot_completed)
& $PY "$WS\capture_logcat.py"      # then, after it ends, also dump the ring buffer:
& $ADB logcat -b all -d > C:\Users\iMac2019\brepdugl\logcat_ringbuffer.txt

# Kernel log:
& $ADB shell dmesg > C:\Users\iMac2019\brepdugl\dmesg.txt

# pstore (capture BEFORE it is overwritten by the next crash/reboot):
& $ADB shell "ls -l /sys/fs/pstore/"
& $ADB shell "cat /sys/fs/pstore/console-ramoops" > C:\Users\iMac2019\brepdugl\console-ramoops.txt
& $ADB shell "cat /sys/fs/pstore/dmesg-ramoops* 2>/dev/null" > C:\Users\iMac2019\brepdugl\dmesg-ramoops.txt
```

---

## 5. Rollback for each risky change

| Change | Rollback (from TWRP, /system and /vendor mounted rw) |
|---|---|
| **H1b — livedisplay rc edited (`disabled` appended)** | `adb shell 'RC=/system/vendor/etc/init/vendor.lineage.livedisplay@2.0-service.legacymm.rc; cp "$RC.orig" "$RC"'` |
| **H1b — livedisplay rc renamed (`*.disabled`)** | `adb shell 'mv <RC>.disabled <RC>; mv /system/vendor/bin/hw/vendor.lineage.livedisplay@2.0-service-legacymm.bak /system/vendor/bin/hw/vendor.lineage.livedisplay@2.0-service-legacymm'` |
| **H1a — `persist.sys.disable_rescue=true`** | `adb shell 'rm /data/property/persist.sys.disable_rescue /data/property/persist.device_config.configuration.disable_rescue_party /data/property/persist.sys.enable_rescue'` |
| **H4 — `init.target.rc` direct mount removed** | `adb shell 'cp /vendor/etc/init/hw/init.target.rc.orig /vendor/etc/init/hw/init.target.rc'` |
| **H4 — `fstab.qcom` changed to `fileencryption=`** | `adb shell 'cp /vendor/etc/fstab.qcom.orig /vendor/etc/fstab.qcom'` |
| **H4 — `/data` reformatted** | irreversible (data loss). Always `mke2fs` only after backing up any wanted data via `adb pull /data/media/0`. |
| **H3 — `apexd.rc`/`bootanim.rc` keyword-stripped** | re-flash the GSI to `/system` (or restore the individual `.orig` if you made one first). |

---

## 6. What to capture FIRST when the device is reconnected

Do these in order, **before** applying any fix, so you get the definitive reset trigger:

1. **Long logcat through the reset.** Start before/at boot and keep writing past `sys.boot_completed=1` for ≥5 min (current `capture_logcat.py` stops at 360s and ends at phase 1000 — extend the timeout, or run `adb logcat -b all -v threadtime > file` as a background stream). Target strings: `RescueParty`, `PackageWatchdog`, `flags_health_check`, `apexd`, `Attempting rescue`, `rebootPromptAndWipeUserData`, `boot reason`.
2. **The exact vold/fscrypt error just before reset.** `adb logcat -b all | grep -iE "vold|fscrypt|StorageManagerService|MountService|encryption unsuccessful"` — confirm whether `Failed to chmod /data/media/0` / `Resetting vold… Reset vold` is the last vold activity (rule out the "decryption unsuccessful" path).
3. **Boot/shutdown reason.** `adb shell getprop ro.boot.bootreason; adb shell getprop sys.boot_completed; adb shell getprop persist.sys.rescue_level` — and confirm the PMIC line `Power-off reason: Triggered from PS_HOLD`.
4. **`dmesg` (kernel log)** — `adb shell dmesg > dmesg.txt` — look for the `updatable process 'vendor.livedisplay-hal-2-0-legacymm' exited 4 times before boot completed` loop and `apexd: Updatable crashing, attempting rollback`.
5. **pstore, immediately** (it is overwritten each crash/reboot) — `adb shell "ls -l /sys/fs/pstore/"` then `adb shell "cat /sys/fs/pstore/console-ramoops" > console-ramoops.txt` (and `dmesg-ramoops*` if present).
6. **`getprop ro.init.updatable_crashing` + `ro.init.updatable_crashing_process_name`** — the smoking gun; expect `1` and `vendor.livedisplay-hal-2-0-legacymm`.
7. **`ls -Zd /data` and `getprop ro.crypto.state`** — confirm `system_data_root_file` + `unencrypted` (secondary symptom, for hypothesis 4).

The single most important new datum is the **`RescueParty`/`apexd`/`flags_health_check` line printed in the ~60 s after `boot_completed=1`** — that is the exact line the current `logcat_boot.txt` is missing.

---

## Bottom line
Disable `vendor.livedisplay-hal-2-0-legacymm` (edit/rename its rc in `/system` **and** rename its binary) **and** set `persist.sys.disable_rescue=true`. That stops `ro.init.updatable_crashing=1` from ever being set and hard-disables the rescue escalation — this should break the factory-reset loop without touching the working `/data` direct mount. The Bluetooth/FBE/SELinux-label issues are a separate, lower-priority cleanup.
