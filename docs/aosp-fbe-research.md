# AOSP Android 11 storage/encryption research — HTC Desire 12+ (brepdugl) factory-reset loop

> Authoritative-source research on why the Android 11 GSI on this kernel-3.18 / A-only (non-system-as-root) device boots to `boot_completed=1` and then loops a "factory data reset".
> All citations below were verified by fetching the exact file from `android.googlesource.com` at tag `android-11.0.0_r48` (unless noted). Line numbers are for that revision.

---

## ⚠️ Critical device-specific fact that reframes everything

This device is **Treble ARM64 A-only (system-as-system, NOT system-as-root)** with a **stock Android 8.0 boot.img (kernel + ramdisk)**.

That means the process that actually runs `on fs`, `mount_all /vendor/etc/fstab.qcom`, parses the fstab, and sets `ro.crypto.state` / `ro.crypto.type` is the **Android 8.0 `init`/`fs_mgr` from the boot ramdisk** — NOT Android 11's. The Android 11 pieces that DO run (from the GSI `/system`) are `vold` (Vold 3.0), `zygote`/ART, `system_server`, and `StorageManagerService`.

This matters because:
- **Both** the Android 8 init (reads `/vendor/etc/fstab.qcom` via `mount_all`) and Android 11 vold (reads the same default fstab via `ReadDefaultFstab()` in `main.cpp` / `FsCrypt.cpp`) parse the same `/data` fstab line. The line must be valid for **both** parsers.
- The "who sets `ro.crypto.type`" decision is made by the **Android 8 init**'s `queue_fs_event()`; Android 11 vold merely *reads* `ro.crypto.type` to decide whether it is in native-FBE mode (`fscrypt_is_native()`).

I verified the Android 8.0 `init/builtins.cpp` `queue_fs_event()` is functionally identical to Android 11's for the FBE/FDE cases: `FS_MGR_MNTALL_DEV_FILE_ENCRYPTED` → `ro.crypto.type=file`, `MIGHT_BE_ENCRYPTED` → `ro.crypto.type=block` + `defaultcrypto`, `NEEDS_RECOVERY` → `--wipe_data --reason=fs_mgr_mount_all`.
- Android 8 init/builtins.cpp: `https://android.googlesource.com/platform/system/core/+/refs/tags/android-8.0.0_r4/init/builtins.cpp` (`queue_fs_event`, lines 459–495)
- Android 11 init/builtins.cpp: `https://android.googlesource.com/platform/system/core/+/refs/tags/android-11.0.0_r48/init/builtins.cpp` (`queue_fs_event`, lines 553–620)

---

## Q1. What triggers "Decryption unsuccessful / Forcing factory reset" and a rescue-party factory reset in Android 11? What does StorageManagerService / vold log right before wiping /data?

There are **three distinct wipe/reset mechanisms** in Android 11, and they have different log signatures. Conflating them is the root of the confusion.

### (a) Recovery "Can't load Android system … factory data reset" — triggered by init on an FBE/FDE setup failure
This is the screen most people call "the decryption-unsuccessful / factory-reset loop". It is the **recovery UI** rendered when recovery is invoked with `--prompt_and_wipe_data`. In Android 11 the init builtins that fire it are:

- `mkdir <dir> … encryption=Require` → `make_dir_with_options()` → `FscryptSetDirectoryPolicy()` fails →
  ```cpp
  return reboot_into_recovery(
      {"--prompt_and_wipe_data", "--reason=set_policy_failed:"s + options.target});
  ```
  `system/core/init/builtins.cpp`, lines 405–412.
- `installkey /data` → `do_installkey()` → `ExecVdcRebootOnFailure("enablefilecrypto")`, and `init_user0` → `do_init_user0()` → `ExecVdcRebootOnFailure("init_user0")`. On failure:
  ```cpp
  LOG(ERROR) << message << ": Rebooting into recovery, reason: " << reboot_reason;
  if (auto result = reboot_into_recovery(
          {"--prompt_and_wipe_data", "--reason="s + reboot_reason}); ...
  ```
  `system/core/init/builtins.cpp`, `ExecVdcRebootOnFailure` lines 1138–1167; `do_installkey`/`do_init_user0` lines 1188–1200.
  Reasons produced: `enablefilecrypto_failed` and `init_user0_failed`.
- `mount_all` → `queue_fs_event()` → `FS_MGR_MNTALL_DEV_NEEDS_RECOVERY` →
  ```cpp
  PLOG(ERROR) << "fs_mgr_mount_all suggested recovery, so wiping data via recovery.";
  const std::vector<std::string> options = {"--wipe_data", "--reason=fs_mgr_mount_all" };
  ```
  `system/core/init/builtins.cpp`, lines 584–592.

**Important for a GSI:** the Android 11 `ExecVdcRebootOnFailure` and `queue_fs_event` paths are **explicitly suppressed when running as a GSI**:
```cpp
if (fscrypt_is_native() && !android::gsi::IsGsiRunning()) {  // reboot into recovery …
} else {
    LOG(ERROR) << "Failure (reboot suppressed): " << reboot_reason;
}
```
(`builtins.cpp` lines 1148–1158), and `queue_fs_event`'s `FS_MGR_MNTALL_DEV_NEEDS_RECOVERY` branch:
```cpp
if (android::gsi::IsGsiRunning()) {
    return Error() << "cannot wipe within GSI";
}
```
(`builtins.cpp` lines 586–588). So **on this GSI, the init-side wipe paths are disabled** — and, in any case, the running init is Android 8.0's (which has no GSI check). The point stands that a pure init `mount_all`/fscrypt failure on a GSI does not, by itself, wipe.

### (b) FDE (cryptfs) "Decryption unsuccessful" — CryptKeeper, Settings app
The literal string **"Decryption unsuccessful"** is an FDE-only UI string in Settings:
- `crypt_keeper_data_corrupt_title` = `"Decryption unsuccessful"` (line 1112)
- `crypt_keeper_data_corrupt_summary` = `"The password you entered is correct, but unfortunately your data is corrupt. … you need to perform a factory reset …"` (line 1124)
  `packages/apps/Settings/res/values/strings.xml`: `https://android.googlesource.com/platform/packages/apps/Settings/+/refs/tags/android-11.0.0_r48/res/values/strings.xml`

It is shown by `CryptKeeper` (Settings) when the FDE password is correct but mounting the decrypted block device fails. The vold FDE side that leads to a wipe is in `cryptfs.cpp`:
```cpp
SLOGE("Checksums do not match - trigger wipe");          // line 2353
…
SLOGE("encryption failed - rebooting into recovery to wipe data\n");   // line 2420
const std::vector<std::string> options = {
    "--wipe_data\n--reason=cryptfs_enable_internal\n"};   // lines 2422–2423
```
`system/vold/cryptfs.cpp`: `https://android.googlesource.com/platform/system/vold/+/refs/tags/android-11.0.0_r48/cryptfs.cpp` (lines 2346–2433; the wipe is gated on `ro.vold.wipe_on_crypt_fail=1` at line 2417).

### (c) RescueParty factory reset — framework, post-boot
This is the mechanism that best matches "boots to setup, then ~1 min later wipes and reboots". It is covered in Q6.

### What StorageManagerService logs before a *vold* reset (not a wipe)
`StorageManagerService` never directly wipes `/data`. Its "reset" is a **vold reset + remount**, logged as:
```
StorageManagerService: Thinking about reset, mBootCompleted=…, mDaemonConnected=…
StorageManagerService: Resetting vold...
StorageManagerService: Reset vold
```
in `resetIfBootedAndConnected()`, `services/core/java/com/android/server/StorageManagerService.java` lines 1095–1145.
Its FUSE-mount failure path logs:
```
StorageManagerService: Failed to mount volume …
StorageManagerService: Scheduling reset in 10s
```
in `mount()` lines 2262–2268 (`FAILED_MOUNT_RESET_TIMEOUT_SECONDS = 10`, line 229). This resets/remounts storage; it does **not** format `/data`.

**Short answer Q1:** The "factory data reset" screen is the recovery prompt (`--prompt_and_wipe_data`), reachable from (1) init fscrypt-policy/key-setup failure (`--reason=set_policy_failed:…`, `enablefilecrypto_failed`, `init_user0_failed`, `fs_mgr_mount_all`), (2) FDE cryptfs failure (`--reason=cryptfs_enable_internal`), or (3) RescueParty `LEVEL_FACTORY_RESET` (`--reason=RescueParty`). The literal "Decryption unsuccessful" text is Settings/CryptKeeper's FDE data-corrupt screen.

---

## Q2. vold 3.0 fscrypt: what does "fscrypt_prepare_user_storage for volume null" mean? Is "null" normal?

**Normal.** The "null" is just how vold prints an *empty* volume UUID, and the internal emulated (non-adoptable) storage always has an empty UUID.

```cpp
static const char* escape_empty(const std::string& value) {
    return value.empty() ? "null" : value.c_str();
}
```
`system/vold/FsCrypt.cpp`, lines 113–115.

During boot, `fscrypt_init_user0()` prepares user 0's **Device-Encrypted (DE)** storage for the internal volume with an empty uuid:
```cpp
bool fscrypt_init_user0() {
    …
    if (!fscrypt_prepare_user_storage("", 0, 0, android::os::IVold::STORAGE_FLAG_DE)) {
        LOG(ERROR) << "Failed to prepare user 0 storage";
        return false;
    }
    …
}
```
`system/vold/FsCrypt.cpp`, lines 467–502 (call at 483).

And the log line itself:
```cpp
bool fscrypt_prepare_user_storage(const std::string& volume_uuid, userid_t user_id, int serial, int flags) {
    LOG(DEBUG) << "fscrypt_prepare_user_storage for volume " << escape_empty(volume_uuid)
               << ", user " << user_id << ", serial " << serial << ", flags " << flags;
    …
    if (volume_uuid.empty()) { … }  // internal storage branch (DE_sys/DE_n dirs)
```
`system/vold/FsCrypt.cpp`, lines 804–821.

So `volume null` = internal emulated volume. It is **not a bug symptom by itself.** The things to worry about are the *return value* (a following `Failed to prepare user 0 storage` LOG(ERROR) means the DE preparation failed) and whether `ro.crypto.type` is actually `file` (see Q3/Q4) — if it isn't, `fscrypt_prepare_user_storage` just creates plain directories with no encryption policy.

---

## Q3. Correct fstab /data config for Android 11 FBE on kernel 3.18 (no AES instructions)? adiantum vs aes-256-xts? Is footer (FDE) valid?

### FBE flag parsing (both Android 8 and 11)
```cpp
void ParseFileEncryption(const std::string& arg, FstabEntry* entry) {
    entry->fs_mgr_flags.file_encryption = true;
    entry->encryption_options = arg;
}
// …
} else if (StartsWith(flag, "forceencrypt=")) {
    entry->fs_mgr_flags.force_crypt = true;   entry->key_loc = arg;
} else if (StartsWith(flag, "encryptable=")) {
    entry->fs_mgr_flags.crypt = true;         entry->key_loc = arg;
} else if (StartsWith(flag, "fileencryption=")) {
    ParseFileEncryption(arg, entry);
} else if (StartsWith(flag, "forcefdeorfbe=")) {
    entry->fs_mgr_flags.force_fde_or_fbe = true;
    entry->key_loc = arg;
    entry->encryption_options = "aes-256-xts:aes-256-cts";
}
```
`system/core/fs_mgr/fs_mgr_fstab.cpp`, lines 104–107 and 185–243.

Decision in `handle_encryptable()`:
```cpp
if (needs_block_encryption(entry)) { … return FS_MGR_MNTALL_DEV_NEEDS_ENCRYPTION; }  // FDE
else if (should_use_metadata_encryption(entry)) { … }
else if (entry.fs_mgr_flags.file_encryption || entry.fs_mgr_flags.force_fde_or_fbe) {
    LINFO << entry.mount_point << " is file encrypted";
    return FS_MGR_MNTALL_DEV_FILE_ENCRYPTED;                                           // FBE
}
else if (entry.is_encryptable()) { return FS_MGR_MNTALL_DEV_NOT_ENCRYPTED; }
else { return FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE; }
```
`system/core/fs_mgr/fs_mgr.cpp`, lines 974–998.

### Kernel constraints (the decisive facts)
- **FBE requires kernel fscrypt** (`CONFIG_EXT4_ENCRYPTION` / `CONFIG_F2FS_FS_ENCRYPTION`). AOSP says it "is available in the Android common kernels, version **3.18 or higher**" — but that is Google's *Android common kernel*, not an arbitrary vendor 3.18 kernel. The stock HTC/QC `3.18.71` kernel for `brepdugl` shipped with **FDE (dm-crypt), not FBE**, so `CONFIG_EXT4_ENCRYPTION` is very likely absent (must be verified on-device: `/proc/config.gz`, `grep -i encrypt`, or `grep fscrypt /proc/filesystems`).
  - File-based encryption doc: `https://source.android.com/docs/security/features/encryption/file-based` ("Kernel Support for Ext4 encryption or F2FS encryption (Kernel config option: EXT4_FS_ENCRYPTION or F2FS_FS_ENCRYPTION)"; "Kernel support … available in the Android common kernels, version 3.18 or higher").
- **`fileencryption=aes-256-xts`** is the classic FBE contents mode. AOSP states contents mode "can be only `aes-256-xts`" (before Adiantum) and lists "Encryption performance in the kernel of at least **50MB/s using AES XTS**" as a dependency. MSM8937 uses Cortex-A53 **without** ARMv8 Crypto Extensions, so AES-XTS is slow software AES → fails the performance bar. Parsed correctly, but not a good fit here.
- **`fileencryption=adiantum`** is the intended mode for CPUs *without AES instructions*, but the Adiantum doc only provides kernel backport patches for **4.19 / 4.14 / 4.9 / 4.4** ("If your device uses a 4.4 kernel, also add CONFIG_CRYPTO_SEQIV=y"). **Kernel 3.18 is not listed → no Adiantum support exists for 3.18.**
  - Adiantum doc: `https://source.android.com/docs/security/features/encryption/adiantum` — "Adiantum is an encryption method designed for devices running Android 9 and higher whose CPUs lack AES instructions"; "Kernel version: 4.19 / 4.14 / 4.9"; userspace: `fileencryption=adiantum` in fstab for FBE, plus `ro.crypto.volume.contents_mode=adiantum`, `ro.crypto.volume.filenames_mode=adiantum`.
- Adiantum is recognized by libfscrypt as a valid mode (`{"adiantum"s, FSCRYPT_MODE_ADIANTUM}` for both contents and filenames):
  `system/extras/libfscrypt/fscrypt.cpp`, lines 53–64, 180–260.

### Is `encryptable=footer` / `forceencrypt=footer` (FDE) valid in Android 11?
**Yes — still parsed and still implemented**, despite being deprecated for *new* devices.
- `forceencrypt=` / `encryptable=` are still parsed (fs_mgr_fstab.cpp lines 185–235, above).
- `needs_block_encryption()` still returns true for `force_crypt` (fs_mgr.cpp line 954).
- The entire FDE implementation `system/vold/cryptfs.cpp` still ships in Android 11 (101 KB), and init.rc still has the FDE triggers (`on encrypt`, `on defaultcrypto`, `on property:vold.decrypt=…`; see `rootdir/init.rc` lines 941–973 and the FDE doc flows).
- The FDE doc (`https://source.android.com/docs/security/features/encryption/full-disk`) documents `forceencrypt` and the four boot flows; "Detect encrypted /data with no password: … one of the flags `encryptable` or `forceencrypt` is set."

So the handover's claim "FDE is unsupported in Android 11 (FBE only)" is **incorrect** — FDE support code remains in vold/fs_mgr/init. FDE is merely not allowed for *devices launching* with Android 10+.

### Bottom line for Q3
- On this exact hardware (kernel 3.18.71, no AES instructions, no Adiantum backport), **`fileencryption=adiantum` will NOT work** (no 3.18 kernel support), and **`fileencryption=aes-256-xts` will only work if the vendor kernel actually has `CONFIG_EXT4_ENCRYPTION`** (unlikely on a stock-FDE device) and will be slow.
- **`forceencrypt=footer` (FDE) is the most likely to actually work** on this old kernel, because dm-crypt + software AES is exactly what the stock Android 8 device already used. It is still valid in Android 11's vold/fs_mgr/init.
- But mixing stock Android 8 **init** (which does the FDE mount handshake via `vold.decrypt` properties) with Android 11 **vold/framework** (different Keymaster/Gatekeeper HAL expectations) is itself a compatibility risk; see Q7.

---

## Q4. If init directly mounts /data via `on fs` mount (NOT vold), does Android 11 get confused and trigger a reset?

**Short answer: yes — bypassing `mount_all` (fs_mgr) breaks the FBE/FDE state machine and leaves Android 11 vold/StorageManagerService in an inconsistent, but not necessarily directly "wipe-triggering", state.**

How `/data` is normally mounted in Android 11 (and Android 8, identically here):
- **FBE (`fileencryption=`):** init's `mount_all` → `fs_mgr_mount_all()` mounts `/data` **directly** (with the fscrypt/inlinecrypt mount options). `handle_encryptable()` returns `FS_MGR_MNTALL_DEV_FILE_ENCRYPTED`, init sets `ro.crypto.state=encrypted`, `ro.crypto.type=file`, and triggers `nonencrypted`. **vold does not mount /data**; its job is key management only (`vdc cryptfs enablefilecrypto`, `init_user0` → `fscrypt_init_user0()`).
- **FDE (`forceencrypt=`/`encryptable=footer`):** init's `mount_all` sees an unmountable/encrypted `/data`, mounts a **tmpfs** at `/data`, sets `ro.crypto.type=block`, triggers `defaultcrypto`/`encrypt`; **vold (cryptfs)** then sets up the dm-crypt mapping and mounts the real `/data` via `fs_mgr_do_mount()` (cryptfs.cpp line ~1627). Even here the actual mount goes through fs_mgr, not a raw `mount(2)`.

Sources: `system/core/init/rootdir/init.rc` (lines 414–443 `late-init`; 528+ `on post-fs-data`; `installkey /data` line 541; `init_user0` line 761); `system/core/fs_mgr/fs_mgr.cpp` (`fs_mgr_mount_all` line 1308, `handle_encryptable` 974–998); `system/vold/cryptfs.cpp` (`fs_mgr_do_mount` of the dm-crypt device).

Consequences of a direct `mount ext4 … /data` in `on fs`:
1. `mount_all` never runs for `/data`, so **`ro.crypto.state`/`ro.crypto.type` are never set correctly**. The Android 8 init's `queue_fs_event()` never sees a `/data` mount result, so it stays `unsupported`/unset.
2. Android 11 vold's `fscrypt_is_native()` is literally:
   ```cpp
   bool fscrypt_is_native() {
       char value[PROPERTY_VALUE_MAX];
       property_get("ro.crypto.type", value, "none");
       return !strcmp(value, "file");
   }
   ```
   `system/extras/libfscrypt/fscrypt.cpp`, lines 88–92.
   With no `fileencryption` flag in the fstab → `ro.crypto.type` never becomes `file` → **`fscrypt_is_native()` is false** → vold treats the device as FDE/non-native, and `fscrypt_init_user0()` skips installing DE keys and instead runs the FDE-legacy path `fscrypt_unlock_user_key(0,0,"!","!")` (FsCrypt.cpp lines 490–492).
3. `vold` also independently reads the same fstab in `get_data_file_encryption_options()`:
   ```cpp
   if (!ParseOptions(entry->encryption_options, options)) {
       LOG(ERROR) << "Unable to parse encryption options for " << DATA_MNT_POINT ": "
                  << entry->encryption_options;
       return false;
   }
   ```
   `FsCrypt.cpp` lines 232–250. With no `fileencryption=` flag, `encryption_options` is empty → parse fails → FBE key setup fails → `LOG(ERROR) << "Failed to prepare user 0 storage"`.
4. The `/data` filesystem ends up **mounted but with no encryption policy applied**, while the Android 11 framework still expects CE/DE directory structure (`/data/user_de/0/…`, `/data/system_ce`, etc.). The Bluetooth crash the handover observed (`SQLiteCantOpenDatabaseException … /data/user_de/0/com.android.bluetooth/databases … doesn't exist`) is the direct symptom: the `user_de` tree was never created because DE preparation failed or ran in the wrong mode.
5. Does this *directly* trigger a wipe? The init wipe paths are on the FBE/FDE *failure* path (and suppressed in a GSI — Q1). So the direct mount itself does **not** call `--wipe_data`. But it produces a half-initialized `/data` that can then feed **RescueParty** (persistent system/app crashes after `boot_completed`) or **StorageManagerService vold-reset loops**, which is the observed behavior.

**Conclusion:** the direct `on fs` mount is a workaround that boots the device but leaves `/data` outside the encryption state machine. It should be removed and replaced with a proper fstab `/data` entry + `mount_all` (see Q7).

---

## Q5. What causes zygote's "Error creating cache dir /data/dalvik-cache/arm64"? Mount ordering or FBE CE-unlock?

**It is a mount/ordering (and/or writability) issue — not a CE-user unlock issue.**

Mechanism: at zygote/ART startup, `Runtime::Init` → boot-image `ImageSpace` setup calls `GetDalvikCache("arm64", create_if_absent=true, …)`, which does:
```cpp
const std::string dalvik_cache_root = android_data + "/dalvik-cache";
*dalvik_cache = dalvik_cache_root + '/' + subdir;
…
if (create_if_absent && !*dalvik_cache_exists && !*is_global_cache) {
    *dalvik_cache_exists = ((mkdir(dalvik_cache_root.c_str(), 0700) == 0 || errno == EEXIST) &&
                            (mkdir(dalvik_cache->c_str(), 0700) == 0 || errno == EEXIST));
}
```
`art/libartbase/base/file_utils.cc`, `GetDalvikCache()` lines 295–327 (Android 11).
The caller in the zygote boot path:
`art/runtime/gc/space/image_space.cc`, `FindImageFilenameImpl()` lines 205–249 (calls `GetDalvikCache(…, /*create_if_absent=*/ true, …)`).

If `/data` is **not mounted** (or not writable) at that moment, `mkdir` fails and ART logs the fatal "Error creating cache dir /data/dalvik-cache/arm64". On this device that was precisely the ordering problem: the fstab used the ueventd `by-name` symlink which wasn't ready at `on fs`, so `/data` never mounted before zygote started (this is the problem the direct mount "fixed").

Why it is **not** a CE-unlock issue:
- `/data/dalvik-cache` lives in **device-encrypted (DE) / unencrypted** storage. DE storage is available immediately after `/data` mounts (Direct Boot), **before** any user CE key is unlocked. The CE/DE split is: CE = `credential encrypted` (locked until user PIN/password), DE = `device encrypted` (available at boot). (`https://source.android.com/docs/security/features/encryption/file-based`, "Direct Boot" section.)
- In `on post-fs-data`, `/data` dirs are created with `encryption=Require`/`encryption=Attempt`/`encryption=None`; `/data/dalvik-cache` is not a CE directory. So a CE-locked user does **not** block dalvik-cache creation.

Therefore the "Error creating cache dir" message is an ART-startup **mkdir failure = /data not mounted/writable yet**. It can also be caused by wrong SELinux labels on a freshly `mke2fs`'d `/data` (which is why init.rc runs `restorecon /data` in `on post-fs-data`, line 538, and `restorecon --recursive --skip-ce /data` line 770).

---

## Q6. RescueParty: what does it log, what triggers it post-boot, and can it cause a factory reset (not just app reset)?

**Yes — RescueParty level 4 is a real factory reset** (it reboots into recovery with `--prompt_and_wipe_data`), not merely an app reset.

Escalation levels (`core/java/com/android/server/RescueParty.java`, lines 80–92):
```
LEVEL_NONE = 0
LEVEL_RESET_SETTINGS_UNTRUSTED_DEFAULTS = 1
LEVEL_RESET_SETTINGS_UNTRUSTED_CHANGES = 2
LEVEL_RESET_SETTINGS_TRUSTED_DEFAULTS = 3
LEVEL_FACTORY_RESET = 4
```

Factory-reset action (lines 273–288):
```java
case LEVEL_FACTORY_RESET:
    Runnable runnable = new Runnable() {
        @Override public void run() {
            try { RecoverySystem.rebootPromptAndWipeUserData(context, TAG); }
            catch (Throwable t) { logRescueException(level, t); }
        }
    };
    Thread thread = new Thread(runnable); thread.start();
```
`RecoverySystem.rebootPromptAndWipeUserData(context, "RescueParty")` → `bootCommand(context, null, "--prompt_and_wipe_data", "--reason=RescueParty", "--locale=…")` → recovery wipes `/data`.
`core/java/android/os/RecoverySystem.java`, lines 1051–1088.

Logging (quote-able):
```java
private static void incrementRescueLevel(int triggerUid) {
    …
    EventLogTags.writeRescueLevel(level, triggerUid);
    logCriticalInfo(Log.WARN, "Incremented rescue level to "
            + levelToString(level) + " triggered by UID " + triggerUid);
}
private static void executeRescueLevel(Context context, @Nullable String failedPackage) {
    Slog.w(TAG, "Attempting rescue level " + levelToString(level));
    …
    logCriticalInfo(Log.DEBUG, "Finished rescue level " + levelToString(level));
}
private static void logRescueException(int level, Throwable t) {
    EventLogTags.writeRescueFailure(level, msg);
    logCriticalInfo(Log.ERROR, "Failed rescue level " + levelToString(level) + ": " + msg);
}
```
`RescueParty.java` lines 236–243, 245–258, 292–297.
On-device these appear as event-log tag `rescue_party` (`writeRescueLevel`/`writeRescueSuccess`/`writeRescueFailure`) plus logcat `RescueParty: Attempting rescue level …`.

Triggers:
- **Persistent app crash / ANR** (PackageWatchdog health check): `RescuePartyObserver.execute()` fires when a **persistent/system** package fails `DEFAULT_TRIGGER_FAILURE_COUNT = 5` times within `DEFAULT_TRIGGER_FAILURE_DURATION_MS = 30s` (PackageWatchdog lines 90, 116, 419–446 in RescueParty). The Bluetooth `SQLiteCantOpenDatabaseException` is an app crash — but it must repeat 5× in 30 s and be persistent/system to qualify.
- **Boot loop** (`noteBoot()` → `mBootThreshold.incrementAndTest()` → `onBootLoop()` → `executeBootLoopMitigation()`): `DEFAULT_BOOT_LOOP_TRIGGER_COUNT = 5` system-server boots within `DEFAULT_BOOT_LOOP_TRIGGER_WINDOW_MS = 10 min` (PackageWatchdog lines 112–124, 446–469; RescueParty lines 473–488, 481–488 for `executeBootLoopMitigation` which calls `incrementRescueLevel(Process.ROOT_UID)`).
- Re-execution after SettingsProvider publishes (`onSettingsProviderPublished` → `executeRescueLevel`, lines 160–167).

RescueParty is **disabled** on eng builds, on userdebug builds with active USB (ADB connected), and by `persist.sys.disable_rescue` / DeviceConfig flag (`isDisabled()`, lines 113–147). **This is the single most likely culprit for "boots to setup then wipes ~1 min later" on a GSI**, and it is also the easiest to test: boot with ADB attached (userdebug) or `setprop persist.sys.disable_rescue 1` and see whether the wipe stops.

Distinguishing log signatures:
- RescueParty wipe → recovery reason `--reason=RescueParty` (and event-log `rescue_party`, `RescueParty: Attempting rescue level FACTORY_RESET`).
- init fscrypt wipe → `--reason=set_policy_failed:…` / `enablefilecrypto_failed` / `init_user0_failed` / `fs_mgr_mount_all`.
- FDE cryptfs wipe → `--reason=cryptfs_enable_internal`.

---

## Q7. Recommended end-to-end /data setup for this device

### Step 0 — Determine what the kernel actually supports (do this first)
```sh
adb shell zcat /proc/config.gz 2>/dev/null | grep -E 'EXT4_ENCRYPTION|F2FS_FS_ENCRYPTION|CRYPTO_ADIANTUM|DM_CRYPT|CRYPTO_AES|CRYPTO_XTS|CRYPTO_AES_ARM'
adb shell cat /proc/filesystems            # look for fscrypt / ext4
adb shell "grep -i crypto /proc/crypto"    # look for xts(aes), adiantum, cbc(aes)
```
- If `EXT4_FS_ENCRYPTION` **is present** → FBE (`fileencryption=`) is possible; prefer `fileencryption=adiantum` only if `CRYPTO_ADIANTUM` exists (it won't on 3.18), otherwise `fileencryption=aes-256-xts` (slow but functional).
- If `EXT4_FS_ENCRYPTION` **is absent** (most likely) → **FBE is impossible** on this kernel; use FDE `forceencrypt=footer` (dm-crypt), which the stock kernel already has.

### Recommended fstab `/data` line (pick one; both are valid to both Android 8 and 11 parsers)

Prefer, only if kernel has fscrypt (FBE, Direct Boot):
```
/dev/block/mmcblk0p53    /data    ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,quota    wait,check,fileencryption=aes-256-xts:aes-256-cts
```
(Use the **direct node** `mmcblk0p53`, not the `by-name` symlink, because on this non-SAR device the ueventd symlink is not ready at `on fs` — this is the ordering bug the handover already diagnosed.)

Most likely to work on kernel 3.18 without fscrypt (FDE, matches the stock device):
```
/dev/block/mmcblk0p53    /data    ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,quota    wait,check,forceencrypt=footer
```

Then **remove the direct `mount` line from `init.target.rc`** and let the device's `mount_all /vendor/etc/fstab.qcom` handle `/data` again (it will run the FBE/FDE state machine and set `ro.crypto.type` correctly).

### mke2fs (format /data)
For **FBE** (only if kernel has fscrypt):
```sh
mke2fs -t ext4 -O encrypt,quota,project -E resize=… /dev/block/mmcblk0p53
```
For **FDE** (`forceencrypt=footer`), format WITHOUT the `encrypt` feature — dm-crypt encrypts the whole block device below the filesystem, and the `encrypt` ext4 feature belongs to fscrypt (FBE) only. A footer-based FDE volume must be a plain ext4 + a crypto footer written by vold's `cryptfs` on first boot:
```sh
mke2fs -t ext4 -O quota /dev/block/mmcblk0p53     # do NOT use -O encrypt
```
Then let vold write the crypto footer at first boot (FDE "encrypt on first boot" flow).

**Important:** do **not** mix them. `mke2fs -O encrypt` (fscrypt feature) on a partition that the init mounts as FDE is meaningless and, on a kernel without `EXT4_FS_ENCRYPTION`, an `encrypt`-featured ext4 may fail to mount entirely.

### SELinux (often overlooked; a freshly mke2fs'd /data has wrong labels)
```sh
restorecon -RF /data        # or rely on init.rc's restorecon /data + restorecon --recursive --skip-ce /data
```
init.rc already does `restorecon /data` (line 538) and `restorecon --recursive --skip-ce /data` (line 770), but if you ever mount `/data` manually in TWRP and write to it, re-run `restorecon`.

### Is a direct `on fs` mount acceptable?
**No — remove it.** A direct mount bypasses `fs_mgr_mount_all`/`handle_encryptable` and prevents `ro.crypto.state`/`ro.crypto.type` from being set, which breaks vold's `fscrypt_is_native()` and leaves `/data` mounted but unencrypted/unprepared (Q4). Use a correct fstab line + `mount_all` instead.

### Recommended test order to isolate the loop
1. Boot with **ADB attached on a userdebug build** (or `setprop persist.sys.disable_rescue 1`) to rule out RescueParty (Q6). If the wipe stops, RescueParty was the trigger; fix the underlying crash (the missing `/data/user_de` tree from Q2/Q4).
2. Capture a **long** logcat past `boot_completed=1` and grep for `RescueParty`, `rescue_party`, `vold`, `StorageManagerService`, `Fscrypt`, `recovery`, `--reason=`.
3. Confirm the kernel's fscrypt capability (Step 0), then commit to **one** of FBE or FDE and re-format consistently.

---

## Quick reference: key files & functions (all verified at `android-11.0.0_r48`)

| File | Function / line | Purpose |
|---|---|---|
| `system/vold/FsCrypt.cpp` | `fscrypt_init_user0` 467–502; `fscrypt_prepare_user_storage` 804+; `escape_empty` 113–115 | "volume null" = internal DE prep (Q2) |
| `system/vold/FsCrypt.cpp` | `get_data_file_encryption_options` 232–250 | fails when fstab lacks `fileencryption=` (Q4) |
| `system/extras/libfscrypt/fscrypt.cpp` | `fscrypt_is_native` 88–92; `ParseOptions` 176+; modes 53–64 | `ro.crypto.type=file` gate (Q3/Q4) |
| `system/core/fs_mgr/fs_mgr_fstab.cpp` | `ParseFsMgrFlags` 185–243; `ParseFileEncryption` 104–107 | flag parsing (Q3) |
| `system/core/fs_mgr/fs_mgr.cpp` | `handle_encryptable` 974–998; `fs_mgr_mount_all` 1308 | FBE/FDE decision (Q3/Q4) |
| `system/core/init/builtins.cpp` | `queue_fs_event` 553–620; `ExecVdcRebootOnFailure` 1138–1167; `do_installkey`/`do_init_user0` 1188–1200; `make_dir_with_options` 405–412 | wipe-reason paths (Q1) |
| `system/core/rootdir/init.rc` | `late-init` 414–443; `on post-fs-data` 528+; `installkey` 541; `init_user0` 761 | boot/mount ordering (Q4/Q5) |
| `frameworks/base/services/…/StorageManagerService.java` | `resetIfBootedAndConnected` 1095–1145; `mount` 2247–2283 | vold reset / "Scheduling reset in 10s" (Q1) |
| `frameworks/base/services/…/RescueParty.java` | levels 80–92; `incrementRescueLevel` 236–243; `executeRescueLevel` 245–258; factory reset 273–288 | RescueParty (Q6) |
| `frameworks/base/services/…/PackageWatchdog.java` | `noteBoot` 446–469; boot-loop trigger 112–124 | boot-loop detection (Q6) |
| `frameworks/base/core/java/android/os/RecoverySystem.java` | `rebootPromptAndWipeUserData` 1051–1088; `rebootWipeUserData` 823–865 | `--prompt_and_wipe_data` / `--wipe_data` (Q6) |
| `system/vold/cryptfs.cpp` | wipe-on-fail 2414–2433; checksum fail 2353 | FDE wipe (`--reason=cryptfs_enable_internal`) (Q1) |
| `packages/apps/Settings/res/values/strings.xml` | `crypt_keeper_data_corrupt_*` 1112/1124 | "Decryption unsuccessful" (Q1) |
| `art/libartbase/base/file_utils.cc` | `GetDalvikCache` 295–327 | dalvik-cache mkdir (Q5) |
| `art/runtime/gc/space/image_space.cc` | `FindImageFilenameImpl` 205–249 | zygote boot-image/dalvik-cache (Q5) |

## Authoritative doc URLs
- File-based encryption: `https://source.android.com/docs/security/features/encryption/file-based`
- Full-disk encryption: `https://source.android.com/docs/security/features/encryption/full-disk`
- Enable Adiantum: `https://source.android.com/docs/security/features/encryption/adiantum`
- Rescue Party (debug doc): `https://source.android.com/docs/core/tests/debug/rescue-party`

---

*Prepared from Android 11 (`android-11.0.0_r48`) source fetched from `android.googlesource.com`, plus Android 8.0 (`android-8.0.0_r4`) `init/builtins.cpp` for the on-device init comparison. All line numbers are exact for the cited revisions.*
