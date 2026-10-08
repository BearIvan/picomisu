<p align="center"><img src="logo/picomisu.png" alt="Picomisu" width="360"></p>

<p align="center"><b>English</b> · <a href="README.ru.md">Русский</a></p>

# Picomisu

Picomisu is a system image (Source) for the PICO 4 Pro (PICOA8110), built from source on
CodeLinaro **LA.UM.8.12.c3-64900-sm8250.0** (Android 10, CAF). That tag is the base of the
factory PICO OS 5.13.7. Picomisu replaces only `system`, `vbmeta_system` and `vbmeta`. The
kernel, boot, vendor, odm and product partitions stay factory, so the image works only on
PICO OS **5.13.7**.

## Contents

- [Building](#building)
  - [Requirements](#requirements)
  - [Downloading the source](#downloading-the-source)
  - [Building the release](#building-the-release)
  - [Options](#options)
  - [Updating and stable releases](#updating-and-stable-releases)
- [Installing](#installing)
- [Factory files](#factory-files)
- [Repositories](#repositories)
- [Signing keys](#signing-keys)
- [Known limitations](#known-limitations)

## Building

### Requirements

- Linux x86_64. Tested on WSL2 with Ubuntu 24.04. The tree must be on ext4, not NTFS (`/mnt/c`).
- 32 GB of RAM or more, and about 200 GB of free disk space.
- The standard AOSP 10 build dependencies, plus `repo`.
- `python3-brotli`, `e2fsprogs`, `rsync` and `binutils`:

  ```bash
  sudo apt install python3-brotli e2fsprogs rsync binutils
  ```
- git access to github.com/BearIvan, because the repositories are private:

  ```bash
  gh auth login && gh auth setup-git
  ```

### Downloading the source

```bash
mkdir picomisu && cd picomisu
repo init -u https://github.com/BearIvan/picomisu.git -b main --depth=1
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch
```

`--depth=1` is recommended: the build does not need the CodeLinaro history, and it takes up
tens of GB. `repo` also checks out the build tools into `picomisu/`.

### Building the release

```bash
picomisu/build.sh
```

The script takes no arguments and does every step itself:

1. Downloads the factory PICO OS 5.13.7 OTA from the URL pinned in
   `picomisu/config/stock-firmware.lock.json`, checks its SHA-256 and reconstructs the
   factory partition images.
2. Extracts the factory files the device tree needs (`device/pico/PICOA8110/extract-files.py`)
   and checks each one against its SHA-256.
3. Builds the system image and the host tools.
4. Unpacks the factory system tree and checks the factory APKs.
5. Assembles the release image. The build is combined with the factory components it does
   not provide, the factory APKs are re-signed, and build.prop, init.rc, ld.config and VINTF
   are generated. The script then builds ext4 at the exact size of the factory partition,
   signs AVB and reads the image back to check it.
6. Runs offline checks: SELinux labels, init, native dependencies and VINTF.

The result is in `out/picomisu/outputs/source-<version>/`: `system.img`, `vbmeta_system.img`,
`vbmeta.img` and `SHA256SUMS.txt`. The version comes from `device/pico/PICOA8110/release.json`.

The steps that loop-mount images read-only run through `sudo`, so the script asks for your
password. Start it as a normal user, not as root.

### Options

Set these as environment variables:

| Variable | Meaning |
|---|---|
| `PICOMISU_BOOT=boot.img` | The boot image your headset runs, for example a Magisk boot read from it. `vbmeta` records its hash. The default is the factory boot from the OTA. |
| `PICOMISU_VARIANT=user` | Build a `user` image instead of `userdebug`. |
| `JOBS=16` | Number of parallel build jobs. The default is 8. |
| `OUT_DIR`, `PICOMISU_WORK` | Build output directory and work directory. The defaults are `out/` and `out/picomisu/`. |

### Updating and stable releases

To update the tree to the development branch:

```bash
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch
picomisu/build.sh
```

Each release is tagged with its version. The tag of this manifest pins the exact commit of
every project, so to rebuild a release:

```bash
repo init -b refs/tags/2.21
repo sync -c -j8 --no-tags --no-clone-bundle --optimized-fetch
picomisu/build.sh
```

`CHANGELOG.md` lists what changed in each release.

## Installing

> **Installing over the factory system erases userdata** (games, recordings and settings).

Requirements:

- A PICO 4 Pro on factory PICO OS 5.13.7.
- An unlocked bootloader.
- Root on the factory system (Magisk), used to write the updater recovery.
- Developer options and USB debugging enabled.

The 5.13.7 bootloader refuses to unlock without a PICO token. The author's headset is
unlocked with an older PICO-signed ABL (2022) in `abl`. Do not restore the 5.13.7 ABL: the
headset would lock again.

How the install works: the PICO 4 Pro has no A/B slots, so `system` is written from recovery.
`picomisu/tools/source-ota.py` first writes an updater recovery: the factory recovery with
root ADB and `device/pico/PICOA8110/source-updater.sh` added. From that recovery it maps
`system` inside `super` without changing the LP metadata, and streams `system`,
`vbmeta_system` and `vbmeta`. It checks the SHA-256 of each before and after writing. The
first install also wipes userdata.

After that, updates go over Wi-Fi with ADB, or through the in-headset **Source Update** app.
`switch factory-5.13.7` returns the headset to the factory system.

**Installation is not yet adapted to this repository layout.** `source-ota.py` still expects
the author's Windows workspace (`adb.exe` from the Android SDK, the per-release configs). A
single `picomisu/install.sh` is the next step.

## Factory files

**No PICO factory binaries are stored in these repositories.** `extract-files.py` pulls the 23
files the build needs from the factory images: 10 class path JARs, 10 PICO/QTI libraries,
`libcryptfs_hw.so` and 3 Smartisan XML lists. It checks each one against
`proprietary-files.json`. Factory APKs and other factory components are carried into the image
straight from the factory system image while it is assembled.

Picomisu does not modify any factory binary today: APKs are only re-signed, and build.prop,
init.rc, ld.config and VINTF are generated from the factory files. If a factory file ever has
to be changed, the change is stored as a binary patch (the `patch` field, applied with
`bspatch`), not as the modified file.

## Repositories

All repositories are private, at github.com/BearIvan:

| Repository | Path | Contents |
|---|---|---|
| `picomisu` | — | this manifest, `CHANGELOG.md` |
| `picomisu_tools` | `picomisu` | `build.sh`, release, OTA and check tools |
| `picomisu_device_pico_PICOA8110` | `device/pico/PICOA8110` | device tree, `release.json`, `extract-files.py` |
| `picomisu_external_picofacialdatadaemon` | `external/picofacialdatadaemon` | face/eye tracking data daemon (fork of thoricelli, MIT) |
| `picomisu_<path>` × 30 | `frameworks/base`, `art`, … | CAF projects with Picomisu changes, branch `picomisu` |
| `picomisu_PicoFacialDataModule` | not in the tree | VRCFaceTracking module for the PC (fork of thoricelli) |

In each CAF fork, the base is one commit with the CAF tag tree. The CodeLinaro history is
not copied; the commit message links to it. Picomisu commits sit on top of that base. All
other projects come straight from git.codelinaro.org.

## Signing keys

**Builds are currently signed with the public AOSP test keys:** `build/target/product/security`
for the factory APKs that are re-signed, and the AVB test key for `vbmeta`. That is fine for
development builds, but anyone can sign with these keys, so they are not acceptable for
distribution. Release keys are not set up yet.

## Known limitations

- The build and install have been tested on one headset (SEKO, INNOLUX5K panel).
- `userdebug` is the tested variant. `user` builds, but has not been tested on a headset.
- PICO telemetry is not ported, on purpose.
