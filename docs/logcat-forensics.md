# Logcat Forensics — HTC Desire 12+ (brepdugl) factory-reset loop

- **Device:** HTC Desire 12+ (`brepdugl`), MSM8937/SDM450, kernel `3.18.71-perf-gcb9049a`
- **ROM:** Android 11 GSI (LineageOS 18.1 + GApps) on `/system` (mmcblk0p24); stock Android 8.0 vendor (mmcblk0p25) + stock boot image
- **Input file:** `C:\Users\iMac2019\brepdugl\logcat_boot.txt` — 2,141,228 bytes, **16,423 lines**
- **Capture scope:** kernel + main + system + crash buffers (`beginning of kernel` @L1, `beginning of main` @L422, `beginning of system` @L1593, `beginning of crash` @L11691). Single boot attempt.
- **Capture end:** `01-21 09:46:03.750` — mid **OnBootPhase_1000** (last line is `OnBootPhase_1000_com.android.server.blob.BlobStoreManagerService`).
- **Clock note:** timestamps start as `01-01 01:01:xx` (kernel RTC not yet set) and jump to `01-21 09:45:52` during `StartAlarmManagerService` (@L8894) when the RTC is synced. The two bases are continuous but offset.

---

## 1. Boot timeline (key phases / timestamps)

| Time | Event |
|---|---|
| `01-01 01:01:07.283` | Kernel start — `Linux version 3.18.71-perf-gcb9049a` (L4) |
| `01-01 01:01:07.857` | PMIC **power-on reason: "Hard Reset and cold boot"**; **power-off reason: "Triggered from PS_HOLD (MSM controlled shutdown)"** (L328-329) → previous shutdown was a *clean software reboot*, not a panic/power-loss |
| `01-01 01:01:08.618/.624` | fs_mgr mounts `/system` (mmcblk0p24) and `/vendor` (mmcblk0p25) (L960,962) |
| `01-01 01:01:12.2` | SELinux policy compiled + loaded (L964-987) |
| `01-01 01:01:14` | init parses `.rc` files (many `invalid keyword` errors, see §2) |
| `01-01 01:01:18.283` | **`/data` mounted**: `EXT4-fs (mmcblk0p53): mounted filesystem with ordered data mode. Opts: barrier=1,noauto_da_alloc` (L1463) |
| `01-01 01:01:18.283` | `init: [libfs_mgr]__mount(source=/dev/block/bootdevice/by-name/userdata,target=/data,type=ext4)=0` (L1465) |
| `01-01 01:01:18.350` | `Command 'mount_all /vendor/etc/fstab.qcom' action=fs (/vendor/etc/init/hw/init.target.rc:50) returned 0 took 117.695ms` (L1486) |
| `01-01 01:01:25.648` | vold service started (L1587-1588) |
| `01-01 01:01:25.791` | `Vold 3.0 (the awakening) firing up` (L1594) |
| `01-01 01:01:25.816` | `fscrypt_init_user0` → `fscrypt_prepare_user_storage …` → `fscrypt_unlock_user_key` → **`Failed to chmod /data/media/0`** (L1606-1654) |
| `01-01 01:01:26.175` | `init: processing action (ro.crypto.state=unsupported && zygote-start)` (L1740) → zygote starts (L1750) |
| `01-01 01:01:26.844` | `AndroidRuntime: >>>>>> START com.android.internal.os.ZygoteInit uid 0 <<<<<<` (pid 824, L1887) |
| `01-01 01:01:27.0–27.6` | zygote + surfaceflinger killed/restarted via property triggers (density etc.) — see §5 |
| `01-01 01:01:31–32` | final zygote pair: zygote32 pid 1531, zygote64 pid 1723 (preload completes) |
| `01-01 01:01:36.316` | **system_server** `SystemServerTiming: InitBeforeStartServices` (pid 1896, L6824) |
| `01-01 01:01:37.265` | **OnBootPhase_100** (L7141) |
| `01-01 01:01:41.190` | `prepareAppData` / `AppDataFixup` (L8540-8550) — installd migration storm begins |
| `01-21 09:45:52.003` | RTC sync; system_server continues (`StartAlarmManagerService` L8894) |
| `01-21 09:45:52.478` | `StartStorageManagerService` (L9049) |
| `01-21 09:45:52.497` | StorageManagerService `Thinking about init/reset, mBootCompleted=false` (L9052-9055) |
| `01-21 09:45:53.907` | **`No primary storage defined yet; hacking together a stub`** (L9712) |
| `01-21 09:45:54.219/.262/.555/.686` | **OnBootPhase 480 / 500 / 520 / 550** |
| `01-21 09:45:54.539` | `UserDataPreparer: Found /data/user_de/0 with serial number -1` (L10643) |
| `01-21 09:45:55.158` | **Bluetooth FATAL EXCEPTION #1** (pid 2040) (L11692) |
| `01-21 09:45:55.189` | `Process com.android.bluetooth (pid 2040) has died: psvc PER` (L11817) |
| `01-21 09:45:56.430` | **Bluetooth FATAL EXCEPTION #2** (pid 2350) (L13018) |
| `01-21 09:45:56.497` | `StorageNotification: … VolumeInfo{private} … internalPath=null` (L13081-13085) |
| `01-21 09:45:56.919` | **OnBootPhase_600** (L13494) |
| `01-21 09:45:57.000` | `ssm.onStartUser-0` (L13806) |
| `01-21 09:45:57.252` | **SetupWizard starts** (`com.google.android.setupwizard`, pid 2546) (L14210-14297); `Is device provisioned = false` (L14531) |
| `01-21 09:46:03.665` | **OnBootPhase_1000 begins** (L16221) |
| `01-21 09:46:03.691` | StorageManagerService `Thinking about init, mBootCompleted=true` (L16292) |
| `01-21 09:46:03.694` | `fscrypt_unlock_user_key` → **`Failed to chmod /data/media/0` (3rd time)** (L16304-16305) |
| `01-21 09:46:03.696` | StorageManagerService `Resetting vold… / Reset vold` (L16309-16310) |
| `01-21 09:46:03.750` | **capture ends** (mid OnBootPhase_1000) |

**No `sys.boot_completed=1` is present in this file.** The grep for `boot_progress|boot_completed|BOOT_COMPLETED|finishBooting` returns only phase-1000 callbacks (`Bluetooth boot completed` L11074, `WifiService: Handle boot completed` L16331). The capture ends ~1 line before `finishBooting()` would set the property. So **this capture does not actually reach `boot_completed`**, and it stops *before* the reset trigger.

---

## 2. System-critical errors / warnings (chronological)

1. **init cannot parse Android-11 `.rc` keywords** (Android 8.0 init binary parsing the GSI's `.rc`):
   - `E init : /system/etc/init/hw/init.zygote64_32.rc: 15: invalid keyword 'task_profiles'` (L1071, also L1073)
   - `E init : /system/etc/init/cppreopts.rc: 31: invalid keyword 'umount_all'` (L1113, L1115)
   - `E init : /system/etc/init/apexd.rc: 8: invalid keyword 'reboot_on_failure'` (L1078, L1080)
   - `E init : /system/etc/init/bpfloader.rc: 56: invalid keyword 'reboot_on_failure'` (L1100)
   - `E init : /system/etc/init/lmkd.rc: 10: invalid keyword 'exec_background'` (L1159)
   - `E init : /system/etc/init/recovery-persist.rc: 3: invalid keyword 'exec_background'` (L1187)
   - These are **non-fatal** (init logs and skips the line), but they show the stock 8.0 init ignores GSI features.

2. **keymaster/QSEE fails initially then recovers:**
   - `E QSEECOMAPI: Error::Cannot open the file /firmware/image/keymaster.mdt errno = 2` (L3232, L3494, L3500)
   - keymaster@3.0 eventually registers (`01-01 01:01:28.099 Registration complete`, L3517). Not fatal.

3. **`/data` is mounted unencrypted by init/fs_mgr, not by vold** (the root architectural problem — see §8):
   - `EXT4-fs (mmcblk0p53): mounted … Opts: barrier=1,noauto_da_alloc` (L1463) — no `encrypt`/`fileencryption`/`quota` mount options.

4. **vold cannot set up emulated storage (the key error):**
   - `E vold : Failed to chmod /data/media/0: No such file or directory` — **3×**: `01-01 01:01:25.884`, `01-01 01:01:26.082`, `01-21 09:46:03.694`.

5. **StorageManagerService has no real primary volume:**
   - `W StorageManagerService: No primary storage defined yet; hacking together a stub` — 2×: `09:45:53.907`, `09:45:55.016`.

6. **user 0 DE storage has no fscrypt serial number:**
   - `V UserDataPreparer: Found /data/user_de/0 with serial number -1` (L10643)
   - `D UserDataPreparer: Serial number missing on /data/user_de/0; assuming current is valid` (L10644)

7. **installd migration storm (~40 entries)** — every package's DE dir is "not active" and migrated from `/data/data/…`:
   - `W installd: Requested default storage /data/user_de/0/com.android.providers.telephony is not active; migrating from /data/data/com.android.providers.telephony` (L8522, and dozens more L8522-13239). This is the CE/DE storage never becoming "active/unlocked".

8. **Bluetooth app crash** (only real crash; details §3).

9. **`/cache` max-mount-count warnings** (cosmetic, not cause):
   - `W EXT4-fs (mmcblk0p26): warning: maximal mount count reached, running e2fsck is recommended` (L1468; also p51/cota/p50).

10. **`/factory` missing** (expected on GSI, not cause):
    - `W init : Couldn't load properties from /factory/factory.prop: No such file or directory` (L1530).

---

## 3. Bluetooth SQLite crash — full stack + preceding state

### State of `/data` and `user_de` immediately before the crash
- `/data` is a freshly-formatted, **unencrypted** ext4 (mounted directly by init, L1463).
- vold did create the DE skeleton: `/data/system_de/0`, `/data/misc_de/0`, `/data/vendor_de/0`, `/data/user_de/0` are prepared by `vold_prepare_subdirs` (L1608-1618). **But the per-app subdir `com.android.bluetooth` under `/data/user_de/0` was never created** by installd, and `/data/media/0` (CE/emulated) does not exist.
- The app's first attempt to create its own dirs fails because the **parent dir is missing**:
  - `09:45:54.760  W ContextImpl: Failed to ensure /data/user_de/0/com.android.bluetooth/cache: mkdir failed: ENOENT` (L11355)
  - `09:45:54.761  W ContextImpl: Failed to ensure /data/user_de/0/com.android.bluetooth/code_cache: mkdir failed: ENOENT` (L11358)
  - `09:45:55.142  W ContextImpl: Failed to ensure /data/user_de/0/com.android.bluetooth/databases: mkdir failed: ENOENT` (L11657)
- installd confirms the migration after the fact:
  - `09:45:56.629  W installd: Requested default storage /data/user_de/0/com.android.bluetooth is not active; migrating from /data/data/com.android.bluetooth` (L13239)

### Full stack trace (first occurrence, pid 2040)
```
01-21 09:45:55.137  2040  2040 D BluetoothDatabase: start()
01-21 09:45:55.140  2040  2040 D BluetoothDatabase: Load Database
01-21 09:45:55.142  2040  2190 W ContextImpl: Failed to ensure /data/user_de/0/com.android.bluetooth/databases: mkdir failed: ENOENT (No such file or directory)
01-21 09:45:55.144  2040  2190 E SQLiteLog: (14) cannot open file at line 38590 of [d2e6722037]
01-21 09:45:55.145  2040  2190 E SQLiteLog: (14) os_unix.c:38590: (2) open(/data/user_de/0/com.android.bluetooth/databases/bluetooth_db) -
01-21 09:45:55.157  2040  2190 E SQLiteDatabase: Failed to open database '/data/user_de/0/com.android.bluetooth/databases/bluetooth_db'.
01-21 09:45:55.157  2040  2190 E SQLiteDatabase: android.database.sqlite.SQLiteCantOpenDatabaseException: Cannot open database '/data/user_de/0/com.android.bluetooth/databases/bluetooth_db': Directory /data/user_de/0/com.android.bluetooth/databases doesn't exist
        at android.database.sqlite.SQLiteConnection.open(SQLiteConnection.java:252)
        at android.database.sqlite.SQLiteConnection.open(SQLiteConnection.java:205)
        at android.database.sqlite.SQLiteConnectionPool.openConnectionLocked(SQLiteConnectionPool.java:505)
        ...
        at com.android.bluetooth.btservice.storage.MetadataDao_Impl.load(MetadataDao_Impl.java:255)
        at com.android.bluetooth.btservice.storage.MetadataDatabase.load(MetadataDatabase.java:94)
        at com.android.bluetooth.btservice.storage.DatabaseManager$DatabaseHandler.handleMessage(DatabaseManager.java:130)
        ...
01-21 09:45:55.157  2040  2190 E SQLiteDatabase: Caused by: android.database.sqlite.SQLiteCantOpenDatabaseException: unknown error (code 14 SQLITE_CANTOPEN): Could not open database
        at android.database.sqlite.SQLiteConnection.nativeOpen(Native Method)
        ...
--------- beginning of crash
01-21 09:45:55.158  2040  2190 E AndroidRuntime: FATAL EXCEPTION: BluetoothDatabaseManager
01-21 09:45:55.158  2040  2190 E AndroidRuntime: Process: com.android.bluetooth, PID: 2040
01-21 09:45:55.158  2040  2190 E AndroidRuntime: android.database.sqlite.SQLiteCantOpenDatabaseException: Cannot open database '/data/user_de/0/com.android.bluetooth/databases/bluetooth_db': Directory /data/user_de/0/com.android.bluetooth/databases doesn't exist
        ... (same frames as above) ...
01-21 09:45:55.189  1896  1949 I ActivityManager: Process com.android.bluetooth (pid 2040) has died: psvc PER
```

A **second identical crash** occurs ~1.3 s later at `01-21 09:45:56.430` (pid 2350, L12989-13020), then `FATAL EXCEPTION: BluetoothDatabaseManager` again.

**Interpretation:** `com.android.bluetooth` is a **persistent** system app (`psvc PER` in the death record). Its DE data dir was never created because user 0's storage was never fully prepared/unlocked, so it crash-loops on startup. This is a **symptom**, not the root cause, but because it is persistent it is the most plausible *trigger* for a RescueParty factory reset (see §5/§8).

---

## 4. vold / fscrypt / StorageManager messages (exact lines)

### vold startup + fscrypt (all "volume null" occurrences)
```
01-01 01:01:25.791   807   807 I vold    : Vold 3.0 (the awakening) firing up
01-01 01:01:25.805   807   807 D vold    : VoldNativeService::start() completed OK
01-01 01:01:25.816   807   810 D vold    : fscrypt_init_user0
01-01 01:01:25.816   807   810 D vold    : fscrypt_prepare_user_storage for volume null, user 0, serial 0, flags 1
01-01 01:01:25.883   807   810 D vold    : fscrypt_unlock_user_key 0 serial=0 token_present=0
01-01 01:01:25.884   807   810 E vold    : Failed to chmod /data/media/0: No such file or directory
```
The `fscrypt_init_user0` / `prepare_user_storage` / `unlock_user_key` triplet repeats at `01-01 01:01:26.012–26.082` (second run, via `vdc`), and `fscrypt_unlock_user_key 0 serial=0 token_present=0` + the `chmod /data/media/0` failure repeat a **third time at `01-21 09:46:03.694`** during phase 1000 (`StorageManagerService: Setting up emulation state, initlocked=false`).

> Note on `volume null`: this is **normal** vold wording for the internal (non-adoptable) volume; the `null` is not itself an error. The real anomaly is `Failed to chmod /data/media/0`.

`vold_prepare_subdirs` only prepares the **DE** tree — `/data/system/users/0`, `/data/misc/profiles/cur/0`, `/data/system_de/0`, `/data/misc_de/0`, `/data/vendor_de/0`, `/data/user_de/0` (L1608-1618). It does **not** create any CE/emulated path (`/data/user/0`, `/data/media/0`).

### StorageManagerService
```
01-21 09:45:52.497  1896  1977 D StorageManagerService: Thinking about init, mBootCompleted=false, mDaemonConnected=true
01-21 09:45:52.497  1896  1977 D StorageManagerService: Thinking about reset, mBootCompleted=false, mDaemonConnected=true
01-21 09:45:53.907  1896  1921 W StorageManagerService: No primary storage defined yet; hacking together a stub
01-21 09:45:55.016  1896  1911 W StorageManagerService: No primary storage defined yet; hacking together a stub
01-21 09:45:56.497  2063  2063 D StorageNotification: Notifying about private volume: VolumeInfo{private}:
01-21 09:45:56.497  2063  2063 D StorageNotification:     type=PRIVATE diskId=null partGuid=null mountFlags=0 mountUserId=-10000
01-21 09:45:56.497  2063  2063 D StorageNotification:     state=MOUNTED
01-21 09:45:56.497  2063  2063 D StorageNotification:     fsType=null fsUuid=null fsLabel=null
01-21 09:45:56.497  2063  2063 D StorageNotification:     path=/data internalPath=null
...
01-21 09:46:03.691  1896  1917 D StorageManagerService: FUSE flags. Settings: true. Default: true
01-21 09:46:03.692  1896  1977 D StorageManagerService: Thinking about init, mBootCompleted=true, mDaemonConnected=true
01-21 09:46:03.692  1896  1977 D StorageManagerService: Setting up emulation state, initlocked=false
01-21 09:46:03.694   807   807 D vold    : fscrypt_unlock_user_key 0 serial=0 token_present=0
01-21 09:46:03.694   807   807 E vold    : Failed to chmod /data/media/0: No such file or directory
01-21 09:46:03.695  1896  1977 D StorageManagerService: Thinking about reset, mBootCompleted=true, mDaemonConnected=true
01-21 09:46:03.696  1896  1977 I StorageSessionController: Started resetting external storage service...
01-21 09:46:03.696  1896  1977 I StorageSessionController: Finished resetting external storage service
01-21 09:46:03.696  1896  1977 I StorageManagerService: Resetting vold...
01-21 09:46:03.697  1896  1977 I StorageManagerService: Reset vold
01-21 09:46:03.697   807   807 I vold    : onUserAdded: 0
```

> **Important:** the `Resetting vold… / Reset vold` at `09:46:03.696` is **not** a factory reset. It is StorageManagerService's normal `resetIfBootedAndConnected()` re-init of the vold binder at boot-completed. Do not confuse it with the wipe.

**Key takeaway:** the "private" volume reports `diskId=null`, `fsUuid=null`, `fsType=null`, `internalPath=null`. There is **no** `/storage/emulated`, no `onVolumeStateChanged→mounted` for an emulated volume, no `mountEmulated`/`sdcardfs` mount of `/data/media`. vold never built a real internal volume because `/data` was already mounted by init, not by vold.

---

## 5. RescueParty / repeated system_server / zygote restarts

- **RescueParty: zero matches.** `grep RescueParty|rescue|Rescue` → **no matches**. No `--prompt_and_wipe_data`, no `RecoverySystem.reboot…`, no `boot-recovery` intent in this capture.
- **system_server: no crash, no restart.** pid 1896 starts at `01-01 01:01:36` and is still alive at `09:46:03.750`. No `FATAL EXCEPTION` in system_server, no `Watchdog` timeout (only normal `Watchdog: Pausing/Resuming HandlerChecker … packagemanagermain/moveab/dexopt`).
- **zygote early restarts are benign property-triggered restarts, not a crash loop.** In the `01:01:26–32` window init logs several signal-9 kills:
  - `processing action (init.svc.zygote=running)` (L2230) followed by `Sending signal 9 to service 'surfaceflinger'` and `'zygote'` (L2263,2285) — triggered by `processing action (ro.sf.lcd_density=*)` (L2262). This is the density property being set and restarting SF/zygote.
  - `Service 'zygote' (pid 824) killed by signal 9` (L2295), restart → pid 1063, kill again (L3267), then final stable zygote pair pid 1531 (zygote32) / 1723 (zygote64). `property_set("ro.boottime.zygote") failed: property already set` (L2985, L5171) confirms repeated re-start of the same service, not crashes.
- **Only persistent-app crash observed = `com.android.bluetooth`** (twice, `psvc PER`). Because it is persistent, each restart can be counted by RescueParty, but **the ~5-crash threshold is not reached within this capture** (capture ends 7 s after the 2nd crash).

→ **No RescueParty/restart loop is proven inside this file.** The reset trigger lies *after* the capture end.

---

## 6. SELinux `avc: denied` lines (notable, incl. `/data`)

SELinux is **enforcing** (all denials show `permissive=0`; the `Starting in permissive mode` at L100 is only the pre-policy kernel state). `/data` (mmcblk0p53) is labeled with xattr (L1464).

`/data`-related denials:
```
01-01 01:01:41.979  1896  1896 W system_server: type=1400 audit(0.0:64): avc: denied { write } for name="/" dev="mmcblk0p53" ino=2 scontext=u:r:system_server:s0 tcontext=u:object_r:system_data_root_file:s0 tclass=dir permissive=0
01-01 01:01:41.979  1896  1896 W system_server: type=1400 audit(0.0:65): avc: denied { write } for name="/" dev="mmcblk0p53" ino=2 scontext=u:r:system_server:s0 tcontext=u:object_r:system_data_root_file:s0 tclass=dir permissive=0
01-01 01:01:27.309   961   961 W tftp_server: type=1400 audit(0.0:10..33): avc: denied { search } for name="tombstones" dev="mmcblk0p53" ino=1048595 scontext=u:r:rfs_access:s0 tcontext=u:object_r:vendor_tombstone_data_file:s0 tclass=dir permissive=0   (×9)
```
Other denials (all typical vendor-mismatch noise on a GSI, none fatal):
- `avc: denied { mounton } for path="/sbin/adbd"` (L1424)
- property reads: `bluetooth_prop`, `exported_camera_prop`, `vendor_system_prop`, `alarm_boot_prop`, `userspace_reboot_exported_prop`, `serialno_prop`, `qemu_hw_mainkeys_prop` (zygote/system_server/bootanim/wcnss)
- vendor HAL hwservice `find` denials: `broadcastradio`, `samsung.hardware.light/miscpower/biometrics`, `xiaomi/oppo/goodix fingerprint`, `lineage.power/touch/trust` (system_server) — expected GSI probe noise
- `diag` chr_file read/write denials for many qcom daemons; `qti_debugfs` search denials.

**No denial blocks `/data/media` creation** — vold (root, `vold` domain) is not denied anything. The missing `/data/media/0` is a logic/architecture problem, not an SELinux block. (Still, `restorecon -RF /data` after `mke2fs` is a cheap recommended step per handover.)

---

## 7. Clue about WHAT initiated the factory reset

Searched: `wipe_data`, `boot_reason`, `recovery`, `factory`, `shutdown`, `reboot`, `Unmount`, `unmount`, `--wipe_data`, `prompt_and_wipe`, `resetUserData`, `wipeData`.

**Findings — all negative for a wipe/reset trigger inside the capture:**
- No `--wipe_data`, no `--prompt_and_wipe_data`, no `resetUserData`, no `wipeData`, no `RecoverySystem` reboot, no `shutdown`/`reboot` command at the end.
- `recovery`/`factory` matches are all benign: `init.qcom.factory.rc` parse (L1046), `StartRecoverySystemService`/`OnBootPhase_*_recoverysystem` (normal service), `update_verifier_nonencrypted` (normal), `install-recovery.sh` missing (L2879, normal on GSI), `updateTopComponentForFactoryTest` (L10944).
- **Only shutdown/boot-reason clue** is at kernel init:
  - `PMIC@SID0 Power-on reason: Triggered from Hard Reset and 'cold' boot` (L328)
  - `PMIC@SID0: Power-off reason: Triggered from PS_HOLD (PS_HOLD/MSM controlled shutdown)` (L329)
  → The previous power-down was a **clean MSM-controlled software shutdown**, consistent with a RescueParty/RecoverySystem "reboot to recovery" (the recovery screen then offered "Try again / Factory data reset", and the user chose/auto "Try again" → this capture is the next boot).

**Conclusion on §7:** This capture does **not** contain the reset trigger. It ends at `09:46:03.750`, mid-OnBootPhase_1000, before `sys.boot_completed=1` and before the ~1-min-later reset.

---

## 8. Best evidence-based conclusion

### What most likely triggers the reset (root-cause chain)

**The emulated/FBE storage layer is never brought up because `/data` is mounted directly as plain ext4 by init/fs_mgr instead of by Android-11 vold.** Evidence, all consistent:

1. `/data` is mounted by fs_mgr with **no** `fileencryption=`/`encryptable=` and **no** `quota` mount option (L1463: `Opts: barrier=1,noauto_da_alloc`), even though the FS has `encrypt,quota,project` features.
2. vold therefore never owns the volume: `fscrypt_prepare_user_storage for volume null` only builds the **DE** tree, and the CE/emulated tree is never created → `Failed to chmod /data/media/0: No such file or directory` (3×).
3. Framework sees no real primary volume: `No primary storage defined yet; hacking together a stub` (2×) and `VolumeInfo{private} … internalPath=null … fsUuid=null` with no `/storage/emulated`.
4. `UserDataPreparer: Found /data/user_de/0 with serial number -1` → user-0 storage is not fully initialized.
5. Downstream symptom: per-app DE dirs are never created, so the persistent app `com.android.bluetooth` crash-loops (`SQLiteCantOpenDatabaseException` ×2 in 1.3 s, death record `psvc PER`).

**Most probable trigger (hypothesis, not yet confirmed):** the persistent `com.android.bluetooth` crash-loop (and/or other persistent apps that fail the same way on missing `/data/user_de/0/<pkg>`) trips **RescueParty** (`noteAppCrash` on a persistent app), which escalates to `RecoverySystem.rebootPromptAndWipeUserData()` → the "Try again / Factory data reset" recovery screen → reboot loop. This is consistent with (a) the clean `PS_HOLD` software shutdown reason, (b) Bluetooth being persistent, and (c) the reset occurring ~1 min *after* boot. Bluetooth crashed twice within the capture window; a few more crashes after the capture end would cross RescueParty's threshold.

The alternative contributor is the **direct-mount vs vold conflict itself**: because vold never mounts an emulated volume, Android-11 storage-dependent system services (MediaProvider, external storage, SystemUI, SetupWizard) operate without a primary volume, and any subsequent storage corruption/misbehavior can push the system into the rescue/reset path.

### What is MISSING to confirm it

The capture ends before the reset, so the trigger is **not proven**. To confirm, capture **next** (in order of value):

1. **A longer continuous logcat** spanning from boot through ~3–5 min past `boot_completed=1` — specifically watch for the first occurrence of:
   - `RescueParty` (`RescueParty: Attempting rescue level …`, `Executing rescue level FACTORY_RESET`, `resetDevice`)
   - `RecoverySystem` / `uncrypt` / `--prompt_and_wipe_data` / `rebootAndWipeUserData`
   - any `FATAL EXCEPTION` in a *persistent* process after phase 1000, and repeated `Process com.android.bluetooth … has died`
   - vold/StorageManagerService right after `Reset vold` (e.g. `onVolumeStateChanged`, `mountEmulated`, `EmulatedVolume`, `resetUserData`)
2. **`adb shell dmesg` and pstore/console-ramoops** (kernel side of the crash/reboot), plus `adb shell cat /proc/last_kmsg` after a reset.
3. **`/misc` partition check** for the rescue arguments: `adb shell dd if=/dev/block/bootdevice/by-name/misc bs=1k count=1 | strings` — look for `--prompt_and_wipe_data` / `boot-recovery`.
4. **On-device storage state while it is still up (during the ~1 min window):** `ls -laZ /data/media /data/user/0 /data/user_de/0/com.android.bluetooth`, `mount | grep -E "media|emulated|data"`, `ls -laZ /storage/emulated`, `getprop sys.boot_completed`, `getprop vold.…`.
5. **A screenshot/log of the exact reset UI** to disambiguate "Decryption unsuccessful" vs "Can't load Android system (Try again / Factory data reset)" vs a SetupWizard-driven wipe.

### Recommended next fix to test (ties directly to the evidence)

Make vold — not init — own `/data` so it can build the emulated volume:
- Remove the direct `mount … /data` from `/vendor/etc/init/hw/init.target.rc`, and give the fstab `/data` entry a direct node + an FBE flag so Android-11 vold mounts it, e.g. `/dev/block/mmcblk0p53 /data ext4 noatime,nosuid,nodev,barrier=1 wait,fileencryption=aes-256-xts` (or `encryptable=footer` per the research agent), then `mke2fs -t ext4 -O encrypt,quota,project` and reboot.
- Run `restorecon -RF /data` after any reformat (SELinux labels).
- As a diagnostic shortcut: temporarily delete the Bluetooth DE dir expectation by confirming installd creates `/data/user_de/0/com.android.bluetooth`; if Bluetooth stops crash-looping, RescueParty should not fire — which would both confirm the trigger and give a workaround while the storage fix is finalized.

---

*Report generated from `logcat_boot.txt` (16,423 lines). All quoted lines are verbatim with timestamps. The capture is a single boot and ends before `sys.boot_completed=1` and before any reset trigger.*
