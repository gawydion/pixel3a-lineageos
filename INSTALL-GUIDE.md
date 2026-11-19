# Installing LineageOS on a Google Pixel 3a (`sargo`)

Reference guide written from a real install completed 2026-11-19 (LineageOS 22.2).
Intended as context for an AI assistant or for a human repeating the job.

---

## Scope and expectations

- **Destructive.** Unlocking the bootloader wipes the device. So does the mandatory data format
  in recovery. Back up anything you care about first; `adb backup` is unreliable on Android 10+,
  so copy files off manually (`adb pull /sdcard/DCIM` etc.) or use a cloud service.
- **Not reversible back to a locked state safely.** Never relock the bootloader while LineageOS
  is installed — it will fail to boot. Relock only after restoring stock firmware.
- **Time:** ~30–60 min of attended work plus a 5–15 min first boot.
- **Network:** ~1.2 GB of downloads.
- **No root.** Official LineageOS is not rooted and this guide does not add root.

---

## Requirements checklist

| Requirement | How to verify |
|---|---|
| Model is **G020F / G020G / G020H** | `adb shell getprop ro.boot.hardware.sku` |
| Stock **Android 12** firmware installed | `adb shell getprop ro.build.version.release` → `12` |
| "OEM unlocking" enabled in Developer options | `adb shell getprop sys.oem_unlock_allowed` → `1` |
| USB debugging enabled and host authorized | `adb devices` → `device` |
| `adb` + `fastboot` on host | `adb version` (Arch: `pacman -S android-tools`) |

Android 12 is the **last** stock release for the Pixel 3a, so a stock device is normally already
on the correct firmware. If it is on an older release, update it through stock Android first.

Do **not** proceed on a carrier-locked unit (some Verizon models cannot unlock OEM). Check
`adb shell getprop ro.boot.carrier` — non-empty plus a locked `sys.oem_unlock_allowed` is a stop sign.

---

## Step 1 — Download

From `https://download.lineageos.org/devices/sargo` take the latest build directory, e.g.
`/full/sargo/<YYYYMMDD>/`:

| File | Purpose |
|---|---|
| `boot.img` | Lineage Recovery (this device is *recovery-as-boot*) |
| `lineage-<ver>-<date>-nightly-sargo-signed.zip` | the ROM (~1.05 GiB) |

`dtbo.img` and `vbmeta.img` also exist in that directory; the official procedure does **not**
require them. Verify every file against the published SHA-256 before flashing.

```bash
sha256sum boot.img lineage-*.zip
```

Store them somewhere with real disk space — not `/tmp` if it is a tmpfs.

---

## Step 2 — Unlock the bootloader

```bash
adb reboot bootloader
fastboot devices                 # must list the serial
fastboot flashing unlock         # wipes the device
```

The phone shows an on-screen confirmation; accept it with the volume and power keys. This is the
one step software cannot complete for you.

Afterwards the phone reboots into a freshly wiped stock Android. Confirm:

```bash
fastboot getvar unlocked         # → unlocked: yes
```

---

## Step 3 — Flash Lineage Recovery

```bash
fastboot --set-active=a          # pin the slot explicitly
fastboot flash boot boot.img     # writes boot_a
```

Optional but harmless: duplicate it (`fastboot flash boot_b boot.img`) so an A/B fallback cannot
land you on an old kernel.

Expected output: `Sending 'boot_a' ... OKAY`, `Writing 'boot_a' ... OKAY`.

---

## Step 4 — Boot into recovery  ⚠️ THE CRITICAL STEP

**Do not use `fastboot reboot`.** On `sargo` the recovery ramdisk lives in `boot` and contains a
`first_stage_ramdisk`. A normal reboot sets `androidboot.force_normal_boot=1`, so the recovery
ramdisk hands off to the **system** partition and you get stock Android. It looks exactly like a
failed flash.

`fastboot reboot recovery` is **not supported** by bootloader `b4s4-0.4-8048689`
(`FAILED (remote: 'Unsupported reboot option reboot-recovery')`). `fastboot boot boot.img` also
boots the system, for the same `force_normal_boot` reason.

Use either of these:

```bash
adb reboot recovery              # from booted Android with USB debugging on  ← recommended
```

or, with the phone in the bootloader menu, choose **Recovery Mode** using the volume keys and
press power to select.

You are in Lineage Recovery when you see the LineageOS logo and a menu:
`Reboot system now` / `Apply update` / `Factory reset` / `Advanced` / `View recovery logs`.

---

## Step 5 — Format data (mandatory)

In recovery: **Factory Reset → Format data / factory reset**, confirm.

This removes the old encryption keys. Skipping it produces an unbootable install or a bootloop.

---

## Step 6 — Sideload the ROM

On the phone: back to the main menu → **Apply update → Apply from ADB**.
The screen changes to "Now send the package you want to apply…".

On the host:

```bash
adb -d sideload lineage-22.2-<date>-nightly-sargo-signed.zip
```

- `adb devices` reports the phone as `sideload` in this mode.
- An `unauthorized` device state beforehand is irrelevant — sideload does not need the usual
  ADB authorization handshake.
- Progress reaching `Total xfer: 1.00x` is success.
- Stopping at **47%** with `adb: failed to read command: Success` / `No error` /
  `Undefined error: 0` is also **success**. Do not reinstall.

---

## Step 7 — Add-ons (optional, before first boot)

Only needed if you want Google services. Must be installed **now**, not after first boot.

- **MindTheGapps** — full Play Services + Play Store, `arm64` build.
- **microG** — open-source Play Services replacement, better privacy.
- **Nothing** — a genuinely Google-free phone. Install F-Droid afterwards; add Aurora Store if
  Play Store apps are needed.

When recovery asks "reboot to recovery to install add-ons?" answer **Yes** if you have add-ons to
flash, then `Apply update → Apply from ADB` again for each package (accept the signature warning —
add-ons are not signed with LineageOS keys). Answer **No** if you installed none.

---

## Step 8 — First boot

Back arrow → **Reboot system now**. First boot takes 5–15 minutes and may reboot once on its own.
The bootloader warning screen (unlocked device) appears on every power-on and is cosmetic.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Stock Android boots instead of recovery | `force_normal_boot` from a normal reboot | `adb reboot recovery`, or bootloader **Recovery Mode** |
| `Unsupported reboot option reboot-recovery` | bootloader does not implement it | use `adb reboot recovery` |
| `fastboot boot` returns OKAY but boots Android | same `force_normal_boot` behaviour | don't rely on RAM boot |
| `fastboot devices` empty, no USB device in `lsusb` | cable/port/phone socket | flip the USB-C plug, clean the socket, swap cable, rear USB-2 port |
| Kernel log: `Cannot enable. Maybe the USB cable is bad?` + full-speed link | physical fault | same as above |
| `adb: device unauthorized` in Android | new host key after wipe | tap **Allow USB debugging** on the phone |
| Sideload "fails" at 47% | known adb quirk | ignore, it succeeded |
| Bootloop after install | data not formatted | return to recovery, Format data, sideload again |

Useful bus watch while debugging connections:

```bash
watch -n1 'lsusb | grep 18d1'
```
`18d1:4ee0` = fastboot · `18d1:4ee1` = Android/recovery · `18d1:4ee7` = Android with ADB.

---

## After install

- **Never relock the bootloader** while LineageOS is installed.
- Updates come through the built-in **Updater** (nightlies are signed; no Google account needed).
- No root by design; `adb root` will not work.
- Google-dependent apps (banking, some games) may refuse to run without Play Services. That is
  inherent to a no-GApps setup.
- Want root later? Do it deliberately (Magisk patched `boot.img` + `fastboot flash boot`) and
  expect to re-patch after each OTA.

---

## Verified environment

- Device: Pixel 3a `sargo`, model `G020F`, A/B slots, bootloader `b4s4-0.4-8048689`
- Stock firmware at install: Android 12 `SP2A.220505.008`
- Installed: LineageOS 22.2, build `lineage-22.2-20261114-nightly-sargo-signed`
- Host: Omarchy (Arch Linux), `android-tools` 37.0.0.p1
