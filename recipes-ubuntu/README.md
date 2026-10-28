<!--
 Copyright (c) 2026 innodisk Corp.
 
 This software is released under the MIT License.
 https://opensource.org/licenses/MIT
-->

- [Overview](#overview)
- [Quick start](#quick-start)
- [How the image is built](#how-the-image-is-built)
- [What goes into the image](#what-goes-into-the-image)
  - [Packages of this build](#packages-of-this-build)
  - [Kernel modules](#kernel-modules)
  - [Ubuntu packages (apt)](#ubuntu-packages-apt)
- [Boot firmware and device tree](#boot-firmware-and-device-tree)
- [Moving to another Ubuntu release](#moving-to-another-ubuntu-release)

# Overview
The Canonical Ubuntu image is not built from source: Canonical's prebuilt image is downloaded and our packages are written into its rootfs, then the result is deployed together with boot firmware and `dtb.bin` as a flashable folder. Everything for that is in this folder, nothing in it is used by the Yocto images.

| Piece | Role |
|---|---|
| [`conf/ubuntu-release.conf`](../conf/ubuntu-release.conf) | The release: image + sha256, boot firmware, kernel, glibc. Everything which has to change together is here. |
| [`classes/ubuntu-image.bbclass`](../classes/ubuntu-image.bbclass) | The mechanism, see [below](#how-the-image-is-built). |
| [`images`](images) | `qcom-ubuntu-image`: says what goes into the image (`UBUNTU_PKGS`, `UBUNTU_MODULES_AUTOLOAD`, `UBUNTU_UNITS`, `UBUNTU_OVERWRITE`) and the board's boot files. Helper files are in [images/files](images/files). |
| [`device-tree`](device-tree) | `ubuntu-dtb`: builds `dtb.bin` from the device-tree sources of the Ubuntu kernel. |
| [`kernel`](kernel) | `ubuntu-kernel-devsrc`: the build tree of the Ubuntu kernel, for out-of-tree modules. |
| [`scripts`](scripts) | `ubuntu-debs-lock.py`: resolves the Ubuntu package manifest into the lock. |

# Quick start
Run from the folder which holds `meta-innodisk-iq` (see the main [README](../README.md) for the checkout):
```bash
# build the flashable Ubuntu folder
kas-container build meta-innodisk-iq/kas/exmp-q911.yml:meta-innodisk-iq/kas/distro-ubuntu.yml
```
- `distro-ubuntu.yml` includes `distro-sota.yml` (same layers and pins), switches to `package_deb` and uses its own `TMPDIR` (`tmp-ubuntu`), so it does not mix with the Yocto build.
- Result: `tmp-ubuntu/deploy/images/exmp-q911/qcom-ubuntu-image/`. Flashing is the same as for the Yocto image.

# How the image is built
Recipe `qcom-ubuntu-image` (the CI relies on this name for every release) inherits `ubuntu-image.bbclass`:

| Task | Does |
|---|---|
| `do_ubuntu_base` | Unpacks Canonical's image, checks that its kernel is the one of the release. Repeated only when the image or release changes. |
| `do_ubuntu_stage` | Collects the `.deb` files: those of this build (`UBUNTU_PKGS`) and Ubuntu's (from the lock). QA for glibc and module vermagic. |
| `do_ubuntu_inject` | Installs the packages into the image with `debugfs` (no mount, no root), runs depmod, `e2fsck`. |
| `do_deploy` | Writes the flashable folder: image, boot firmware, `rawprogram0.xml`, `dtb.bin`. |

A changed package only repeats stage and inject.

The class stops when the pieces of a release do not belong together: the lock's snapshot is older than the image, the base package list is not the one of the image, the image has another kernel than the modules are built for, a binary needs a newer glibc, a module has another vermagic.

# What goes into the image
## Packages of this build
The `.deb` files of `UBUNTU_PKGS` (Yocto `package_deb`) go through the same installer as Ubuntu's packages, so `dpkg -l`, `dpkg -L`, `dpkg -S` and `apt remove` know them. What differs from an Ubuntu package:
- **Maintainer scripts are not stored.** They are Yocto's, written for its rootfs construction. A script which creates users or groups stops the build. Units are enabled at build time from `UBUNTU_UNITS`.
- **Architecture** is written as `arm64`.
- **Dependencies** on packages which are not in the image (Yocto names, `kernel-module-*`, ...) and all version constraints are dropped from the dpkg entry, the installer prints them. `md5sums` are generated.
- A file of the Ubuntu image is **never silently replaced**: name it in `UBUNTU_OVERWRITE`.
- A package which has the name of a package of the image stops the build.
- Only the newest `.deb` of a recipe in `DEPLOY_DIR_DEB` is taken.

## Kernel modules
Out-of-tree modules are built for Canonical's kernel (`6.8.0-1089-qcom`), not the Yocto kernel. No module recipe is changed: `conf/ubuntu-release.conf` (required by [`kas/distro-ubuntu.yml`](../kas/distro-ubuntu.yml)) makes [`ubuntu-kernel-devsrc`](kernel/ubuntu-kernel-devsrc_6.8.0.bb) the provider of `virtual/kernel`, so recipes with `inherit module` build against it. The Yocto builds of this layer are not affected.

To put a module into the image, add `<recipe>:kernel-module-<name>-<abi>-qcom` to `UBUNTU_PKGS` in [`qcom-ubuntu-image`](images/qcom-ubuntu-image_24.04.bb), and the name to `UBUNTU_MODULES_AUTOLOAD` if it has to be loaded at boot. [`ubuntu-depmod.sh`](images/files/ubuntu-depmod.sh) then rebuilds `modules.dep` and the other indexes.

## Ubuntu packages (apt)
The utilities which Ubuntu ships as packages are installed **at build time**, the image needs no first boot service and no network. The result is the same as `dpkg -i`.

All files are in [images/files](images/files):

| File | Purpose |
|---|---|
| `ubuntu-packages.txt` | The manifest, the packages you want. Edit this. |
| `ubuntu-base-packages.txt` | What the unmodified Ubuntu image already has. Generated from the image. |
| `ubuntu-debs.lock` | Every `.deb` to download, with sha256, pinned to a [snapshot.ubuntu.com](https://snapshot.ubuntu.com) snapshot. Generated. |
| `ubuntu-scripts.ack` | The reviewed maintainer scripts. |
| `ubuntu-install-debs.py` | The installer, runs in `do_compile` of `qcom-ubuntu-image`. |

**Add or remove a package**
1. Edit `ubuntu-packages.txt`.
2. Resolve the dependencies and write the lock:
    ```bash
    python3 recipes-ubuntu/scripts/ubuntu-debs-lock.py
    ```
    It keeps the snapshot of the current lock, prints what was added, removed or changed, and stops if a package would need an upgrade of something the image already has.
3. Build. If the build asks for it, review the new maintainer scripts (below).
4. Commit the manifest, the lock and the `.ack` together.

**Take newer versions** (security updates, ...): `ubuntu-debs-lock.py --snapshot 20260601T000000Z`. Use a snapshot close to the date of the Ubuntu image (the default): newer packages may need a newer `libc6` and the installer does not upgrade what the image ships.

**Maintainer scripts** (`preinst`, `postinst`) are **not run**, that would need a chroot and so root. Instead every script is reviewed once and `ubuntu-scripts.ack` says what to do, keyed by the sha256 of the script:

| Action | Meaning |
|---|---|
| `ignore` | Nothing has to exist before the first boot, e.g. `py3compile`, `systemd-tmpfiles --create`. |
| `systemd-enable` | Create the symlinks of `deb-systemd-helper enable`. |
| `addgroup:<name>` | Create the system group in `/etc/group` and `/etc/gshadow`. |
| `copy:<src>:<dst>` | Copy a file of the package. |

A script which is not in the file, or changed in a new version, **stops the build**. To review:
```bash
python3 recipes-ubuntu/images/files/ubuntu-install-debs.py --review \
    --image <ubuntu image copy> --offset 569376768 \
    --lock recipes-ubuntu/images/files/ubuntu-debs.lock --debs <dir of the .deb files> \
    --ack recipes-ubuntu/images/files/ubuntu-scripts.ack
```
It prints every unacknowledged script with its hash. Read it, then add a line: `<sha256> <package> <script> <action>[,<action>...]`. If none of the actions covers what a script does, extend the installer instead of ignoring it.

**What the installer does** (all through `debugfs`):
1. Writes the files with the owner and mode of the deb (paths below `/lib`, `/bin`, `/sbin` go to `/usr/...`, Ubuntu is usr-merged).
2. Updates the dpkg database (`/var/lib/dpkg/status`, `info/<package>.*`) and marks packages which are not in the manifest as automatically installed.
3. Does what the reviewed scripts ask for.
4. Reads every written file back and compares it.

**Limits:** a file which already exists in the image stops the build; so does a package which adds `/etc/ld.so.conf.d/*.conf` or has other triggers than `ldconfig`; packages which need a user (`adduser`) or something else not listed above need a new action in the installer.

# Boot firmware and device tree
`do_deploy` takes the CodeLinaro boot firmware (`QLI.1.8-Ver.1.1`, QCS9100) from `conf/ubuntu-release.conf` as-is, and adds:
- **`rawprogram0.xml`** of Canonical's image (writes the whole `.img` as one raw disk). The one in the firmware tarball does not match this image and is not used. Rename it (e.g. `_rawprogram0.xml`) to skip the rootfs write on a re-flash when only firmware changed.
- **`rawprogram[1-4].xml`, `patch*.xml`, GPT bins** of `partition_ufs/`, copied up next to `prog_firehose_ddr.elf` (the board boots from UFS).
- **`dtb.bin`** built by `ubuntu-dtb`, never Canonical's: it has the device-tree differences between Canonical's and Qualcomm's kernel fixed, see [ubuntu.dtsi](../recipes-bsp/device-tree/files/ubuntu.dtsi).
- **`xbl.elf`, `xbl_config.elf`** from [`images/files/firmware/r1.0_00120.0/inno-workaround-PWR-2ms`](images/files/firmware/r1.0_00120.0/inno-workaround-PWR-2ms), over the CodeLinaro ones. This set (not the `exmp-q911/` one of the Yocto `.bbappend`) boots Ubuntu correctly (reset button, QUP SPI). Keep `xbl.elf`, `xbl_config.elf` and `devcfg_iot.mbn` of CodeLinaro together as a matched set: mixing in the ones built for the Yocto image caused an early-boot loop.

The build fails if `prog_firehose_ddr.elf`, `rawprogram0.xml`, `rawprogram4.xml`, `dtb.bin` or the `.img` is missing.

# Moving to another Ubuntu release
For a newer Ubuntu release (x16, ...), the recipe stays `qcom-ubuntu-image`, only `conf/ubuntu-release.conf` changes:
1. Pick the release from [Canonical](https://people.canonical.com/~platform/images/qualcomm-iot/ubuntu-24.04/) (`ubuntu-24.04-xNN/`). The `*-rb8.yaml` in `ubuntu-desktop-24.04/` shows the image/firmware pairing Canonical validated for QCS9100. Take the matching firmware from [CodeLinaro](https://artifacts.codelinaro.org/ui/native/qli-ci/flashable-binaries/ubuntu-fw/QCS9100/) for the QLI version of this project.
2. Update the image, its sha256, the firmware and the kernel (with the headers packages and their sha256) in `conf/ubuntu-release.conf`. The device tree and the modules must come from the kernel the image runs.
3. Refresh the base package list and resolve the lock again:
    ```bash
    python3 recipes-ubuntu/scripts/ubuntu-debs-lock.py --refresh-base <ubuntu image .img>
    python3 recipes-ubuntu/scripts/ubuntu-debs-lock.py --snapshot <date of the image>
    ```
4. Build, and check the device-tree differences in [ubuntu.dtsi](../recipes-bsp/device-tree/files/ubuntu.dtsi) against the new kernel.

Packages which need updating later can be delivered as Debian packages, like this [repository](https://github.com/aiotads/iQ-ubuntu__confidential).
