# Pixel 3a (sargo) — LineageOS 22.2 — Install Log

**Status: ✅ SUCCESS.** Clean LineageOS 22.2 installed, no Google services.
Completed 2026-11-19.

---

## Result

| Item | Value |
|---|---|
| Device | Google Pixel 3a, model **G020F**, codename `sargo` |
| OS | **LineageOS 22.2** (Android 15) |
| Build | `lineage-22.2-20261114-nightly-sargo-signed` |
| Google services | **none** — no Play Services, no Play Store (by request) |
| Bootloader | unlocked (stays unlocked; warning screen on boot is cosmetic) |
| Root | none (stock LineageOS is not rooted; `adb root` will not work) |
| Data | wiped once by the bootloader unlock, once more by the required format |
| USB debugging | off after install; re-enable in Developer options when needed |

Single Google account(s) that were on the device were removed by the wipe, so there was no
factory-reset-protection prompt.

---

## The one thing that matters (for future attempts)

**Do not use `fastboot reboot` to get into recovery on this device.**

`sargo` uses a *recovery-as-boot* image (the recovery ramdisk lives in the `boot` partition and
contains a `first_stage_ramdisk`). On a normal reboot the bootloader passes
`androidboot.force_normal_boot=1`, so that same ramdisk hands control straight to the **system**
partition — you boot stock Android, not recovery, and it looks like the flash failed.

Use one of these instead:

```bash
adb reboot recovery          # from a booted Android, USB debugging on  ← what worked
```
or from the bootloader menu select **Recovery Mode** with the volume/power keys.

This single mistake caused several hours of false "the flash didn't take" debugging.

---

## Chronology of what actually happened

1. Device confirmed over ADB (`18d1:4ee7` = MTP+ADB), shell user authorized.
2. Verified prerequisites: model `G020F` ✅, firmware Android 12 `SP2A.220505.008` ✅ (final
   Android 12 for sargo, exactly what LineageOS 22.2 requires), `sys.oem_unlock_allowed=1` ✅,
   not carrier-locked ✅.
3. `fastboot flashing unlock` → bootloader unlocked, **userdata wiped**.
4. `fastboot flash boot boot.img` → Lineage Recovery written to `boot_a`.
5. ❌ `fastboot reboot recovery` → `FAILED (remote: 'Unsupported reboot option reboot-recovery')`.
6. ❌ `fastboot reboot` → booted **stock Android** (the `force_normal_boot` trap above).
7. Flash was then repeated to *both* `boot_a` and `boot_b`, slot pinned with `--set-active=a`.
   Still booted stock Android — same trap, not a write failure.
8. ❌ `fastboot boot boot.img` (RAM boot) → `OKAY`, still landed in stock Android, because RAM
   booting also sets `force_normal_boot`.
   Proof it was our kernel, not stock: `/proc/version` showed
   `4.9.337-... #0 SMP PREEMPT Sat Nov 14 12:15:13 UTC 2026` (the LineageOS build date) while
   `ro.build.description` still read `sargo-user 12 SP2A.220505.008`.
9. ✅ `adb reboot recovery` → **LineageOS Recovery 22.2** finally came up.
10. On-device: *Factory Reset → Format data / factory reset*, then *Apply update → Apply from ADB*.
11. `adb sideload lineage-22.2-20261114-nightly-sargo-signed.zip` → installed.
12. Recovery offered "reboot to recovery to install add-ons?" → answered **No** (no add-ons).
13. *Reboot system now* → LineageOS boot animation → setup wizard. ✅

---

## Side quest: the USB 2.0 flakiness

Around step 6–8 the phone stopped enumerating at all. Kernel log:

```
usb usb1-port6: Cannot enable. Maybe the USB cable is bad?
usb 1-6: new full-speed USB device number 23 using xhci_hcd
usb usb1-port6: unable to enumerate USB device
```

Full-speed (12 Mbps) instead of high-speed (480 Mbps) plus repeated `Cannot enable` is the
signature of a **physical** fault, not software. It was intermittent — the same phone worked on
`usb 1-1` earlier and later. Practical fix list, in order of likelihood:

1. Re-seat / **flip the USB-C plug 180°** (worn ports often only work one orientation).
2. Clean lint out of the phone's USB-C socket (very common on a 2019 handset).
3. Try a known-good **data** cable (charge-only cables enumerate nothing).
4. Try a rear USB-A 2.0 port.

Handy bus watch:

```bash
watch -n1 'lsusb | grep 18d1'
```
- `18d1:4ee0` = fastboot
- `18d1:4ee1` = Android / recovery (MTP)
- `18d1:4ee7` = Android with MTP + ADB

---

## Files

`images/` (not committed to git — 1.2 GB) from
`https://mirrorbits.lineageos.org/full/sargo/20261114/`

| File | SHA-256 |
|---|---|
| `lineage-22.2-20261114-nightly-sargo-signed.zip` | `954e95a682ebc2c8e4560c62445997aa8c1e1f1da8d5c097313a78fe1cdb5a10` ✅ |
| `boot.img` (Lineage Recovery) | `aa306000695edfee2b0ce85150b43f7d7bc11a56105af17072cace473bd53898` ✅ |
| `dtbo.img` | `a6e4cde7c3f3262dceb238f441092ffc2ae8f77208884bb72bdeec0ebd0678e5` ✅ |
| `vbmeta.img` | `18235ab6fb3524b8509ac4ef34576367e13f7f18942435229719f45422028f1f` ✅ |

`dtbo.img` / `vbmeta.img` are not used by the official path; kept as spares.

---

## Post-install notes

- **Do not relock the bootloader** (LineageOS FAQ). Relocking bricks a non-stock system.
- Updates arrive via the built-in Updater (nightlies are signed; no Google account needed).
- App sources without Google: **F-Droid** (free/open-source), **Aurora Store** (Play Store front-end).
- Google-dependent apps (banking, some games) may refuse to run — that's inherent to a
  no-GApps install, not a fault.
- Battery/storage: 50 GB total; the ROM plus factory reset left internal storage effectively empty.

---

## Hardware facts captured before the wipe

| Item | Value |
|---|---|
| Serial | `9CSAY1QW7C` |
| USB product IDs | `4ee7` (MTP+ADB), `4ee1` (MTP), `4ee0` (fastboot) |
| Firmware at start | Android 12, `SP2A.220505.008` |
| Bootloader | `b4s4-0.4-8048689` |
| Slot layout | A/B, `slot-count:2`, was active `a` |
| Pre-wipe usage | ~17 GB of 50 GB |
| Host | Omarchy (Arch), `android-tools` 37.0.0.p1 |
