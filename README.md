<div align="center">

# iOS 16 on the Redmi 10C

**Apple's XNU kernel and the unmodified iPhone X 16.7.16 userland on a Xiaomi `rain` (Snapdragon 680 / SM6225).**

[Status](#status) · [Hardware](#hardware) · [How it boots](#how-it-boots) · [Build](#build-it-yourself) · [Roadmap](#roadmap) · [Credits](#credits)

</div>

> **2026-10-24: first light.** XNU reaches UFS, mounts the HFS+ iOS root, starts `launchd`, `backboardd` and `SpringBoard`, and draws the iOS 16 lock screen. The clock advances. Touch is not enabled yet, so this is a bring-up demo, not a usable phone.

This is a port of [MakrSas/ios-topaz](https://github.com/MakrSas/ios-topaz), which brought the iPhone X userland to the Redmi Note 12 4G. The original topaz README, including the full engineering history of the XNU port, is preserved as [README.topaz.md](README.topaz.md).

## Status

| Area | State |
| --- | --- |
| Boot | Stock Xiaomi ABL → `fastboot boot` of a boot-image v4 shim → XNU. Boot slots and firmware are not written |
| CPU | Four Cortex-A73 cores; the A53 cluster is parked because XNU and the iOS userland use 16K pages |
| Memory | 1.75 GiB at `0x80000000`, below the firmware carve-out |
| Storage | UFS host driver; iOS root is a journaled HFS+ volume in `userdata`, read/write |
| iOS userland | Real iPhone X 16.7.16 (20H392) root and shared cache; `launchd` and system daemons run |
| Display | 720×1650 `ft8006s` DSI panel, presented to iOS as an iPhone X 375×812-point display (750×1624 @2x), with a 15 px side crop and 13 px letterbox |
| Brightness | Panel DBV set to `0x7ff` at boot for bring-up; CoreBrightness integration is still to come |
| GUI | `backboardd` and `SpringBoard`; iOS 16 lock screen with a running clock |
| Touch | **Not yet.** FocalTech SPI bring-up is the next step |
| GPU | No hardware acceleration; frames are composed on the CPU at 30 Hz |
| RTC | No driver yet, so the date starts at 31 December |
| Audio, Wi-Fi, cellular, camera, sensors | Not ported |
| Security | Permissive AMFI/sandbox, stub AppleKeyStore, no SEP |

## Hardware

| Part | Detail |
| --- | --- |
| Phone | Xiaomi Redmi 10C (`rain`) |
| SoC | Qualcomm SM6225, 4× Cortex-A73 + 4× Cortex-A53, Adreno 610 |
| RAM | 4 GB LPDDR4 |
| Storage | UFS 2.2 |
| Panel | `ft8006s`, 720×1650, DSI video mode, 60 Hz, ~269 ppi |
| Touch | FocalTech `fts_ts`, SPI on QUP0 SE0 (`0x4a80000`), reset GPIO86, IRQ GPIO80 |
| Bootloader | Unlocked Xiaomi ABL; slot `_b` |

The kernel runs on the four A73 cores. The A53 cores cannot translate 16K pages, which XNU and the iOS 16 userland require.

## How it boots

```text
Xiaomi ABL (stock, unlocked)
  └─ fastboot boot shim-boot-rain.img          boot image v4, RAM only
       └─ shim on the boot CPU
            ├─ prints MMFR0 / MIDR, prepares the GIC
            ├─ wakes a Gold CPU and parks the boot CPU
            └─ finds the XNU payload behind _end
                 └─ XNU
                      ├─ UFS driver
                      ├─ HFS+ root in userdata (read/write)
                      ├─ dyld + shared cache
                      └─ launchd → backboardd → SpringBoard → lock screen
```

There is no serial console. The shim and kernel draw their console into the framebuffer left by ABL. The kernel also keeps a persistent log ring at the end of `userdata`.

## Build it yourself

You need an unlocked Redmi 10C, a Linux PC with `fastboot`, a GitHub account for the kernel build, and the iPhone X iOS 16.7.16 IPSW. **Everything in `userdata` is erased.** Nothing is written to boot, vendor_boot, modem, persist or the partition table.

### 1. Kernel

The kernel builds on GitHub's macOS runners with [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build):

```sh
gh workflow run xnu.yml -R movka2/ios-rain --ref main
gh run download RUN_ID -R movka2/ios-rain -n xnu-kernel -D xnu/out
```

The result is `xnu/out/kernel.development.sweet`.

### 2. iOS root image

Use the iPhone X (iPhone10,3) iOS 16.7.16 (20H392) IPSW. On Linux, with `apfs-fuse`, `hfsprogs` and sudo:

```sh
sudo bash tools/mkiosroot.sh OS.dmg SystemOS.dmg AppOS.dmg ios16-root-rain.img 16G
sudo mount -t hfsplus -o loop,force ios16-root-rain.img /mnt/iosroot
sudo python3 tools/sweet-root-tweaks.py /mnt/iosroot
sudo umount /mnt/iosroot
img2simg ios16-root-rain.img ios16-root-rain.simg
```

The device-tree builder needs `/product` from the real iPhone X device tree. Take `Firmware/all_flash/DeviceTree.d22ap.im4p` from the IPSW, extract its LZFSE payload, and save it as `research/ipsw-x/d22ap.adt`.

### 3. ADT, payload and shim

```sh
python3 tools/mkadt.py shim/out/rain.adt --rain

python3 tools/mkpayload.py xnu/out/kernel.development.sweet \
    shim/out/rain.adt shim/out/payload-rain-debug.bin \
    --cmdline 'debug=0xc serial=3 cpus=4 keepsyms=1 -noprogress rd=ufsroot0 unentitled_ios_sim_launch=1 ignition_level=1 wdt_test=no_user_checkin ufsrw=1 rootrw=1 launchd_unsecure_cache=1 wdt=-1 sweetfb_hz=30 sweettouch=0'

MKBOOTIMG=/usr/share/android-tools/mkbootimg/mkbootimg.py \
    shim/build.sh ../shim/out/payload-rain-debug.bin
cp shim/out/shim-boot.img shim/out/shim-boot-rain-debug.img
```

The ADT describes the rain memory map, the 750×1624 canvas and the iPhone X identity. The shim uses the 720×1650 panel geometry from `shim/console.h`.

### 4. Flash and boot

```sh
fastboot -S 512M flash userdata ios16-root-rain.simg   # destructive; ~5 minutes
fastboot boot shim/out/shim-boot-rain-debug.img
```

The first boot spends a while on the Apple logo while `launchd` builds its caches; the lock screen follows. To return to fastboot, hold Power + Volume Down.

| Boot argument | Effect |
| --- | --- |
| `rd=ufsroot0 ufsrw=1 rootrw=1` | Root on `userdata`, writable |
| `cpus=4` | All four A73 cores; `cpus=1` is the safe fallback |
| `sweetfb_hz=30` | CPU frame-copy rate |
| `sweettouch=0` | Touch driver disabled during bring-up |
| `sweetfb_native=1` | Present the raw 720×1650 panel instead of the 375×812-point display (debugging only; the red UIKit status bar returns) |
| `sweetfb_noconsole=1` | Stop drawing kernel text after the first iOS frame; omit it while debugging |
| `launchd_unsecure_cache=1`, `wdt=-1 wdt_test=no_user_checkin` | Launchd without AMFI; no hardware watchdog |

## Roadmap

1. **FocalTech SPI touch.** QUP0 SE0, reset GPIO86, IRQ GPIO80, 720×1650 coordinates.
2. **Display polish.** CoreBrightness, DSI/MDP timing, panel power, dot pitch, and a proper solution for the 720×1650 presentation.
3. **Real-time clock** from the PMIC RTC or passed in by the shim.
4. **GPU / 60 Hz.** CPU composition is the current limit.
5. **Audio, Wi-Fi, USB and the rest of the phone.**

## Repository layout

| Path | What |
| --- | --- |
| `shim/` | Bare-metal boot shim, boot image v4, Gold hand-off, GIC setup, payload loader |
| `xnu/patches/` | XNU patch series, applied in order by CI; rain-specific patches start at `0951` |
| `.github/workflows/xnu.yml` | Kernel CI build |
| `tools/mkadt.py` | Apple Device Tree for rain (`--rain`) and topaz |
| `tools/mkpayload.py` | Packs kernel + ADT + boot arguments into the shim payload |
| `tools/mkiosroot.sh`, `tools/sweet-root-tweaks.py` | Build and adjust the HFS+ iOS root |
| `NOTES.md` | Engineering log of the topaz port |
| `README.topaz.md` | Original topaz README |
| `CREDITS.md` | Licences and upstream credits |

Not in git: partition backups, live device trees, Apple firmware, root images and build trees.

## Credits

- [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet): the original XNU-on-Qualcomm port.
- [MakrSas/ios-topaz](https://github.com/MakrSas/ios-topaz): the Redmi Note 12 4G port this tree is based on.
- [HackiOS](https://github.com/oosexgang/hackios-xnu-kernel) by OOSEX GANG: GPU, display, audio, Wi-Fi and sandbox work (APSL 2.0 / MIT; see [CREDITS.md](CREDITS.md)).
- [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build) and [blacktop/ipsw](https://github.com/blacktop/ipsw).
- [topaz-woa](https://github.com/MakrSas/topaz-woa): hardware documentation and drivers for SM6225.
- Apple's open-source XNU, IOHIDFamily, libpthread, hfs and the iPhone X firmware.

iOS itself is Apple's and is not distributed here. This is an experimental project; do not expect a daily driver.
