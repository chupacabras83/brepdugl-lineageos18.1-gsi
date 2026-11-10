# HTC Desire 12+ (brepdugl) — Recovery & Fix Plan (post-FBE-hang)

## State right now
- Device is soft-bricked: the FBE attempt (`fileencryption=aes-256-xts`) made the Android 8.0 `init`
  hang at first-stage `mount_all`, **before** adbd, so there is no adb window and no USB enumeration.
- Root cause of the ORIGINAL loop (already confirmed) is unchanged:
  vold `fscrypt_prepare_user_storage` (CE) fails at `mkdir /data/vendor_ce/0` because
  `/data/vendor_ce` was never created -> "Destroying user 0" -> `--prompt_and_wipe_data` -> reset.

## To recover the device (USER ACTION REQUIRED)
1. Hold **Power ~15-20 s** to force power-off.
2. With phone off, hold **Vol-Down + Power ~15 s** to enter bootloader/fastboot.
   - fallback: Vol-Up + Power (recovery); Vol-Down + Vol-Up + Power (EDL).
3. Tell the agent what the screen shows. The moment `fastboot devices` shows `33c6ba65`,
   run: `fastboot boot C:\Users\iMac2019\brepdugl\twrp-3.7.0_9-0-brepdugl.img`

## Once back in TWRP — PRIMARY fix (lowest risk: band-aid, keep the KNOWN-bootable config)
The device previously booted to `boot_completed=1` with **unencrypted /data + direct `on fs` mount**;
the only thing that reset it was the missing `/data/vendor_ce`. So: restore that config, then just
create the missing directories (no encryption handshake at all).

1. Mount vendor rw and restore the ORIGINAL files (from `analysis/device-backup/orig/`):
   - `/vendor/etc/fstab.qcom`  -> original (unencrypted, by-name `userdata`, `wait` only)
   - `/vendor/etc/init/hw/init.target.rc` -> original (with the direct `on fs` mount)
2. Reformat `/data` plain ext4 (no `-O encrypt`): `mke2fs -t ext4 /dev/block/mmcblk0p53`
3. Pre-create the missing dirs (plain, matching the dirs the boot already creates):
   - `mkdir -p /data/vendor_ce` ; `chmod 0771 /data/vendor_ce` ; `chown system:system /data/vendor_ce`
   - `mkdir -p /data/media/0` ; `chown media_rw:media_rw /data/media/0`
   - (set SELinux labels matching `/data/system_ce` and `/data/media` if needed)
4. `adb reboot` and capture a fresh boot; verify `boot_completed=1` holds with **no** reset.

## FALLBACK (if band-aid still resets): FDE
- fstab `/data` -> `/dev/block/mmcblk0p53 /data ext4 noatime,nosuid,nodev,barrier=1,noauto_da_alloc  wait,forceencrypt=footer`
- remove the direct `on fs` mount
- `mke2fs -t ext4 /dev/block/mmcblk0p53` (plain, NO `-O encrypt`); reboot (vold writes the crypto footer).
- NOTE: `forceencrypt=footer` (FDE/dm-crypt) is the device's native scheme; the earlier
  "Decryption unsuccessful" was because /data was still stock-FDE-encrypted, not a fresh setup.

## Files already on disk
- `analysis/device-backup/orig/fstab.qcom.orig` + `init.target.rc.orig` (true originals)
- `analysis/device-backup/vendor-fstab.qcom` (currently holds the FDE variant)
- `analysis/device-backup/vendor-init.target.rc` (direct mount removed)
- `analysis/fresh-boot-prev-*/logcat_03.txt` (the definitive reset-trigger capture)
