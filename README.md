# PS4 Linux kernel

Linux **7.3.4** for jailbroken PlayStation 4 consoles - Aeolia, Belize and Baikal
(fat, Slim, Pro) - with the AMD Liverpool/Gladius GPU, the MediaTek and Marvell
Wi-Fi chips, and the PS4 storage path (UFS + dm-crypt + AES-XTS) built in.

This repository is the kernel behind [PS4 Linux](https://github.com/shanerm420/ps4-linux-desktop):
the exact tree, the exact configuration, the build script, and the checksums of
the binary that ships in the installer package. It exists so the kernel can be
found, audited and rebuilt on its own, without cloning the whole distribution.

> **Nothing here has been tested on a console by this project.** The build is
> reproducible and the options are verified after configuration, but no PS4 was
> available to boot it. Treat it as a build, not a proven boot.

## The binary

| | |
|---|---|
| File | `bzImage` |
| Size | 14,251,008 bytes |
| sha256 | `a70b093511ed5c58bf03df56646b5f85c770a92c8c9545b608f5aefee291b889` |
| Release string | `7.3.4-PS4Linux-baikal+` |
| Config | `PS4Linux-kernel-7.3.4-baikal.config`, sha256 `e513ca400802c70f3d9809fd0963753f00233bd788bec245a95b748c48b9de58` |

That exact file is what sits at `assets/ps4linux/bzImage` inside `PS4Linux.pkg`,
next to `initramfs.cpio.gz`. If you already have the package, you already have the
kernel - no download needed.

## Where it comes from

```
https://gitlab.com/rmuxnet/linux        branch ps4/7.3.4
```

One branch covers every console. The tree's own config handles Aeolia, Belize and
Baikal, and the loader picks the southbridge at runtime - `SB=` in the build is a
label for the output, not a different kernel.

Earlier notes around the internet name branches like `baikal/7.0.8-Stable` or a
GitHub mirror. Those do not exist; the tree is on GitLab, on the branch above.

## Build it yourself

Needs an x86-64 Linux host with ~15 GB free, clang/LLD (LLVM=1 build), `bc`,
`flex`, `bison`, and the OpenSSL/ELF/zstd headers:

```sh
sudo apt install build-essential clang lld llvm bc flex bison \
                 libssl-dev libelf-dev libzstd-dev

git clone --depth 1 -b ps4/7.3.4 https://gitlab.com/rmuxnet/linux.git src/linux
SB=baikal bash build.sh
```

The result lands in `out/kbuild-baikal/out/bzImage`, together with the `.config`
that produced it. Build time is roughly 20-40 minutes on a modern machine; the
link step is the slow part and wants several GB of RAM.

`build.sh` is the same script the distribution uses, kept here so this repository
stands on its own. Point it at an existing checkout with `SRC=/path/to/linux`.

## What the build does, and why

- **`-march=btver2 -mtune=btver2`** - the PS4 CPU is a Jaguar core: SSE4.2, AVX,
  F16C, BMI1, but **no** AVX2, FMA or BMI2. Generic x86-64 code is safe; a
  `-march=native` build from a desktop is not.
- **GPU firmware built in** via `CONFIG_EXTRA_FIRMWARE` - `amdgpu` needs the
  Liverpool/Gladius blobs before it can bring up HDMI. Without them the screen
  stays dark and it looks like a hardware fault.
- **Wi-Fi and Bluetooth firmware built in** - which chip is present is decided by
  the SDIO ids at runtime (88W8797, 88W8897, MT7668), so all supported sets ship.
- **The storage path** - `EXT4_FS`, `BLK_DEV_LOOP` (the install is one image file
  on the PS4 drive), `DM_CRYPT` + AES-XTS (that drive is encrypted), `ZSTD_DECOMPRESS`
  (the rootfs tarball), `SQUASHFS`, `OVERLAY_FS`, `BLK_DEV_NVME`, `USB_STORAGE`, `UAS`.
- **Controllers and sound** - `HID_PLAYSTATION` for the DualShock 4 over USB and
  Bluetooth, `SND_HDA_INTEL` for HDMI audio.
- **Every requested option is checked after `olddefconfig`** and the build fails
  loudly if one silently did not stick. The same goes for the firmware list.

`CONFIG_NETCONSOLE`, `CONFIG_MAGIC_SYSRQ` and `CONFIG_KEXEC` stay on because the
upstream config has them, and `CONFIG_KEXEC` is how the loader hands the console
to Linux at all. They are reported, not rejected.

## Verify

```sh
sha256sum bzImage
# a70b093511ed5c58bf03df56646b5f85c770a92c8c9545b608f5aefee291b889  bzImage

strings bzImage | grep -m1 PS4Linux
# 7.3.4-PS4Linux-baikal+
```

A rebuild on a different compiler version will not be byte-identical - kernel
builds embed the compiler and timestamps. Compare the release string and the
config instead, and treat the sha256 above as identifying *this* artifact.

## How PS4 Linux uses it

The loader reads `bzImage` and `initramfs.cpio.gz` from `/data/linux/boot` (USB
first, then internal storage) and `kexec`s them. Two things next to the kernel
matter:

- **`vram.txt`** - a plain number in MB. The loader gives that much of the 8 GB
  unified pool to the GPU as VRAM and the rest stays system memory. It overrides
  whatever the payload was built with, which is why one kernel serves all sizes.
- **`bootargs.txt`** - optional extra kernel arguments.

The initramfs opens the encrypted PS4 drive, finds the ext4 image, mounts it and
switches root. See the main project for that side.

## Hardware

| Console | Southbridge | Status |
|---|---|---|
| PS4 fat | Aeolia | untested |
| PS4 Slim | Belize | untested |
| PS4 Pro | Baikal | untested (development target) |

The tree itself supports all three. What has been established on a real Baikal
console is that the drive key, AES-XTS and UFS2 work - the ChromeOS-for-PS4
project booted on it. This kernel has not.

## Licensing

GPL-2.0, like Linux itself. The source is the upstream tree above plus the
configuration in `build.sh`; no patches are applied here, so publishing this
script and the config is the complete source side of the binary.

**Sony-extracted blobs are not in this repository** - no `liverpool_*.bin`, no
`gladius_*.bin`, no EAP HDD key. The firmware that is built in comes from the
upstream tree's `extra_firmware/`, which the loader-side projects also rely on.

## Related

- [PS4 Linux](https://github.com/shanerm420/ps4-linux-desktop) - the installer,
  the Debian + XFCE rootfs, the initramfs and the docs.
