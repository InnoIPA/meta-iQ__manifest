- [Purpose of this document](#purpose-of-this-document)
- [Scope: baseline vs. ODM value-add](#scope-baseline-vs-odm-value-add)
- [1. Board bring-up (device enablement)](#1-board-bring-up-device-enablement)
- [2. Add-on card ecosystem (Innodisk-designed hardware)](#2-add-on-card-ecosystem-innodisk-designed-hardware)
- [3. Platform services \& connectivity](#3-platform-services--connectivity)
- [4. Test, verification \& traceability](#4-test-verification--traceability)
- [5. Build \& release infrastructure](#5-build--release-infrastructure)
- [Full package group](#full-package-group)
- [Related documents](#related-documents)

# Purpose of this document
This document supports **design-change review** for 2nd-development partners building on this layer. It answers one question: *what does this meta-layer (`meta-innodisk-iq`) add on top of the Qualcomm Yocto baseline, and why was each addition necessary?*

Each section below is one category of design change. Every item links to the recipe/file that implements it, so reviewers can trace a claim back to source.

# Scope: baseline vs. ODM value-add
The baseline is Qualcomm's own Yocto stack (`meta-qcom`, `meta-qcom-hwe`, `meta-qcom-distro`, `meta-qcom-qim-product-sdk`), which supports the reference EVKs (`qcs9075-iq-9075-evk`, `qcs8275-iq-8275-evk`) only. Everything in this layer exists because the Innodisk `exmp-q911` module (COM-HPC Mini, QCS9075) and its carrier-board ecosystem are **not** the reference design — Innodisk, acting as ODM, closes the gap between "Qualcomm reference EVK" and "shippable module + carrier board." Per [DEVELOPMENT.md](DEVELOPMENT.md#rules-of-meta-layers), changes to upstream recipes are restricted to `.bbappend` only (`recipes-bsp/`), so the baseline itself is never forked — every design change is layered on top and stays traceable.

> **TODO:** `exmp-q801` (QCS8275, EVT platform baseline since v2.5.0) and the `exma-q911` variants (`exma-q911-agx-orin`, `exma-q911-js-02`) are not yet documented in this file — all sections below describe `exmp-q911` only.

# 1. Board bring-up (device enablement)
Changes required because `exmp-q911` is a different physical board from the Qualcomm EVK (different DT, different boot config, different peripherals wired up).

| Design change | Why it's needed | Where |
|---|---|---|
| Custom machine definition for `exmp-q911` | Declares this board's DTBs/DTBOs, features (usbhost/usbgadget/alsa/wifi/bluetooth) and kernel cmdline separately from the EVK | [conf/machine/exmp-q911.conf](../conf/machine/exmp-q911.conf), [conf/machine/include/q911-extra-bootargs.inc](../conf/machine/include/q911-extra-bootargs.inc) |
| Custom device tree | The EVK devicetree doesn't describe Innodisk's module/carrier wiring | [recipes-bsp/device-tree/linux-qcom_%.bbappend](../recipes-bsp/device-tree/linux-qcom_%25.bbappend) |
| Custom boot binaries (`xbl.elf`, `xbl_config.elf`, `devcfg_iot.mbn`) | Required to expose `i2c@88c000` (not enabled by the EVK boot config) and to fix the PMIC LAN reset pin voltage (1.8V→1.9V) with a PWR hard-reset (2ms) workaround | [recipes-bsp/firmware-qcom-bootbins/firmware-qcom-boot-qcs9100_%.bbappend](../recipes-bsp/firmware-qcom-bootbins/firmware-qcom-boot-qcs9100_%25.bbappend) |
| Camera bring-up fixes | CCI reset/pinctrl/drive-strength patches and expander GPIO removal needed for the camera module actually wired on `exmp-q911`, now carried directly in the board device-tree sources | [recipes-bsp/device-tree/files](../recipes-bsp/device-tree/files) |
| Audio path calibration & routing | Calibration data (`acdb_cal.acdb`/`workspaceFileXml.qwsp`) and MI2S-LPAIF TERTIARY channel routing plus mclk-enable kernel patch for the `wm8904` codec, on Qualcomm's AudioReach stack (successor to ACDB/AGM) | [recipes-bsp/audioreach-conf](../recipes-bsp/audioreach-conf/audioreach-conf_git.bbappend), [recipes-bsp/audioreach-graphmgr](../recipes-bsp/audioreach-graphmgr/audioreach-graphmgr_git.bbappend), [recipes-modules/audioreach-kernel](../recipes-modules/audioreach-kernel/audioreach-kernel_git.bbappend) |
| Kernel config fragments for onboard/add-on peripherals | ~20 config fragments enabling drivers Qualcomm's default kernel config doesn't turn on (RTC, XFS, onboard USB hub, wm8904 codec, INA260, add-on card drivers, etc.), plus board-specific patches | [recipes-modules/inno-modules/linux-qcom_%.bbappend](../recipes-modules/inno-modules/linux-qcom_%25.bbappend) |
| Ethernet bring-up & naming | Pins `end0`/`end1` to their physical MAC controller via `.link` files (stmmac's generic `eth%d` claim order isn't stable across boots) and re-arms the PHY SERDES clock on admin-down so a later `ifconfig up` doesn't fail | [recipes-apps/inno-daemon/inno-daemon_1.0.bb](../recipes-apps/inno-daemon/inno-daemon_1.0.bb) |
| Fan control | Board-specific PWM fan control service — the EVK has no fan | [recipes-apps/inno-fan/inno-fan_1.0.bb](../recipes-apps/inno-fan/inno-fan_1.0.bb) |
| Boot logo & console handoff | Custom psplash boot logo, plus preventing weston from hijacking the serial debug console at boot | [recipes-bsp/psplash/psplash_%.bbappend](../recipes-bsp/psplash/psplash_%25.bbappend), [recipes-modules/inno-modules/files/boot-logo.cfg](../recipes-modules/inno-modules/files/boot-logo.cfg) |
| Login banner / build identification | Shows the exact BSP build (`git describe`) at console login, so a unit in the field can be identified by build | [recipes-bsp/base-files/base-files_%.bbappend](../recipes-bsp/base-files/base-files_%25.bbappend) |

# 2. Add-on card ecosystem (Innodisk-designed hardware)
`exmp-q911` supports a family of Innodisk expansion cards (serial, CAN, power monitoring). None of these exist in the Qualcomm baseline — the driver, the firmware, and the userspace tool are all Innodisk deliverables for each card.

| Card / device | Kernel driver | Userspace tool |
|---|---|---|
| EGP2-X401 (serial) | [recipes-modules/egp2-x401](../recipes-modules/egp2-x401/egp2-x401_1.0.bb) (f81601) | [recipes-apps/emp2-tool](../recipes-apps/emp2-tool/emp2-tool_1.0.bb) (`emp2cfg`/`emp2init`/`emp2reg`) |
| EGP2-X403 (serial) | [recipes-modules/egp2-x403](../recipes-modules/egp2-x403/egp2-x403_1.0.bb) (f81504_series) | [recipes-apps/serial-f81504-tool](../recipes-apps/serial-f81504-tool/serial-f81504-tool_1.0.bb) |
| EGPC-B4S1 (CAN) | [recipes-modules/egpc-b4s1](../recipes-modules/egpc-b4s1/egpc-b4s1_1.0.bb) (xr17v35x) | — |
| EMUC family — EGPC-B201, EMUC-B202, EMUC-B2S3 (USB-to-CAN) | [recipes-modules/emuc2socketcan](../recipes-modules/emuc2socketcan/emuc2socketcan_1.0.bb) (shared SocketCAN line discipline) | [recipes-apps/emuc-canutil](../recipes-apps/emuc-canutil/emuc-canutil_1.0.bb) (`emucd` daemon, udev rule, systemd template) |
| F81504A (serial) | [recipes-modules/f81504a](../recipes-modules/f81504a/f81504a_1.0.bb) (f81504a_u3) | — |
| INA260 (power/current monitor) | in-kernel `hwmon` driver, enabled via [recipes-modules/inno-modules/files/hwmon-ina260.cfg](../recipes-modules/inno-modules/files/hwmon-ina260.cfg) | — |
| Onboard library shared by EMUC tooling | — | [recipes-library/libiagt](../recipes-library/libiagt/libiagt_1.0.0.bb) |

This is the clearest evidence of ODM scope: a 2nd-dev partner gets a working driver + CLI for every add-on card out of the box, instead of having to write board support for each one.

# 3. Platform services & connectivity
| Design change | Why it's needed | Where |
|---|---|---|
| `inno-daemon` | Board management daemon running on the platform, with an `exmp-q911`-specific implementation | [recipes-apps/inno-daemon](../recipes-apps/inno-daemon/inno-daemon_1.0.bb) |
| `inno-version` | Reports the exact BSP build in-field, tied to the git tag it was built from | [recipes-apps/inno-version](../recipes-apps/inno-version/inno-version_1.0.bb) |
| `inno-ota` | OTA firmware update tool for the on-board NUVOTON MS51XB9BE MCU over serial | [recipes-apps/inno-ota](../recipes-apps/inno-ota/inno-ota.bb) |
| `libqcperf` | Remote CPU/GPU/NPU performance monitoring library for customer workload profiling | [recipes-apps/libqcperf](../recipes-apps/libqcperf) |
| WiFi/Bluetooth firmware packages (3165ngw, AX200, AX210, BE200) | The specific WiFi/BT modules Innodisk qualifies on this platform aren't bundled by the Qualcomm baseline | [recipes-apps/3165ngw-fw](../recipes-apps/3165ngw-fw/3165ngw-fw.bb), [ax200ngw-fw](../recipes-apps/ax200ngw-fw/ax200ngw-fw.bb), [ax210ngw-fw](../recipes-apps/ax210ngw-fw/ax210ngw-fw.bb), [be200-fw](../recipes-apps/be200-fw/be200-fw.bb) |
| DNN-enabled OpenCV | Enables the OpenCV DNN module, needed by customer vision workloads | [recipes-library/opencv/opencv_%.bbappend](../recipes-library/opencv/opencv_%25.bbappend) |
| TensorFlow Lite | Bundles tf-lite libraries into `packagegroup-innodisk` for on-device ML inference | [recipes-bsp/tensorflow-lite/tensorflow-lite_2.16.1.bbappend](../recipes-bsp/tensorflow-lite/tensorflow-lite_2.16.1.bbappend) |
| Locale support (en-US, zh-TW, zh-CN) | Required for Innodisk/regional deployment targets, not part of upstream defaults | [conf/machine/include/innodisk-locales.inc](../conf/machine/include/innodisk-locales.inc) |

# 4. Test, verification & traceability
These exist specifically to support design-change review, customer acceptance and field support — evidence that a build is correct, not just that it compiles.

| Design change | Why it's needed | Where |
|---|---|---|
| I/O function verification table | Per-machine record of which I/O (audio, CAN, USB, camera, RTC, TPM, etc.) has been verified and how | [stesting report](stesting-report_exmp-q911.html) |
| `stesting` | Automated/jig-based I/O test tool used to produce the verification results above and the release test report | [recipes-apps/stesting](../recipes-apps/stesting/stesting_git.bb) |
| `usb-test-mode` | USB2.0/3.0 compliance test-mode tool, needed for USB certification/compliance evidence | [recipes-apps/usb-test-mode](../recipes-apps/usb-test-mode/usb-test-mode_git.bb) |
| `sysmonapple` | Hexagon DSP profiling/monitoring, used for platform bring-up diagnostics | [recipes-apps/sysmonapple](../recipes-apps/sysmonapple/sysmonapple_git.bb) |
| `qprof` | Qualcomm platform profiling tool packaged for this platform | [recipes-apps/qprof](../recipes-apps/qprof/qprof_git.bb) |
| Build version scheme (Major.Minor.Patch = HW rev / QLI version / bugfix count) | Ties every shipped image back to a specific hardware revision and Qualcomm Linux Imagery baseline | [VERSION.md](VERSION.md), realized via [classes/inno-git-info.bbclass](../classes/inno-git-info.bbclass) |
| SBOM generation (SPDX 2.2 + CycloneDX) | Supply-chain/compliance deliverable for customer review, opt-in per build | [classes/sbom.bbclass](../classes/sbom.bbclass), [kas/sbom.yml](../kas/sbom.yml) |

# 5. Build & release infrastructure
| Design change | Why it's needed | Where |
|---|---|---|
| kas build configs | Reproducible, containerized build definition pinning every upstream layer (poky, meta-openembedded, meta-qcom*, etc.) plus this layer, per target machine | [ci/base.yml](../ci/base.yml), [kas/exmp-q911.yml](../kas/exmp-q911.yml), [kas/innodisk-distro-sota.yml](../kas/innodisk-distro-sota.yml) |
| Confidential release channel | Separate build overlay pulling in customer-restricted sources for controlled release builds | [kas/innodisk-distro-sota-confidential.yml](../kas/innodisk-distro-sota-confidential.yml) |
| Documented release flow | Defines the exact steps (tag → build → test → package → release) so every release is reproducible and auditable | [DEVELOPMENT.md — Release flow](DEVELOPMENT.md#release-flow) |
| UEFI capsule packaging | Packages `capsule.cap` as part of the release build, with `xbl_config.elf` pre-patched by default so a later OTA update can apply | [jenkinsfile](jenkinsfile) |
| Layer contribution rules | Keeps the ODM layer's changes scoped and reviewable (bbappend-only in `recipes-bsp/`, per-machine separation, oelint-adv style) | [DEVELOPMENT.md — Rules of meta-layers](DEVELOPMENT.md#rules-of-meta-layers) |

# Full package group
All of the packages above (plus supporting dev/runtime dependencies) are assembled into a single package group installed on the image:
[recipes-core/packagegroups/packagegroup-innodisk.bb](../recipes-core/packagegroups/packagegroup-innodisk.bb), pulled into the image via [recipes-core/qcom-image/qcom-multimedia-proprietary-image.bbappend](../recipes-core/qcom-image/qcom-multimedia-proprietary-image.bbappend).

# Related documents
- [README.md](../README.md) — build/flash instructions, supported machines, FAQ
- [DEVELOPMENT.md](DEVELOPMENT.md) — contribution rules, release process, known issues
- [VERSION.md](VERSION.md) — versioning scheme
- [stesting-report_exmp-q911.html](stesting-report_exmp-q911.html) — I/O verification status for `exmp-q911`
