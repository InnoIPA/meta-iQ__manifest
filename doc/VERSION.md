<!--
 Copyright (c) 2025 innodisk Crop.
 
 This software is released under the MIT License.
 https://opensource.org/licenses/MIT
-->
- [Overview](#overview)
- [Major](#major)
- [Minor](#minor)
- [Patch](#patch)
- [Release Notes](#release-notes)

# Overview
- This page shows the version rules.
- `Major`.`Minor`.`Patch`
- `Patch` will reset to 0 if Major or Minor changed.

# Major
- Related to hardware update.
- Since 2, only DVT/PVT release will count `Major`, EVT pre-release only count in `Patch`.

| Value | Description |
|-------|-------------|
| 0     | exmp-q911 EVT released |
| 1     | exma-q911 EVT released |
| 2     | exmp-q911 DVT released |

# Minor
- Related to QLI version.

| Value | Description |
|-------|-------------|
| 0     | QLI1.5      |
| 1     | QLI1.6      |
| 2     | QLI1.7      |
| 3     | QLI1.8      |

# Patch
- Bug fix or feature update.
- Plus one for each udpate.

# Release Notes

## v2.3.5 - 2026-08-21
- feat: qcom firmware for IQ9 (xbl.elf/xbl_config.elf/pm.dtsi) with PMIC PWR hard-reset 2ms workaround, for both r1.0_00114.0 and r1.0_00120.0 baselines
- feat: capsule package flow, default xbl_config.elf pre-patched for further OTA
- feat: innodisk customized desktop background on weston
- feat: camera device-tree patch for exma-q911-js02
- fix: eth0/eth1 naming now follows QLI2.0 convention, renamed to end0/end1
- fix: ethernet sequence issue causing retry-prevention to trigger incorrectly
- fix: remove unused exmp-ethup service, add sleep to exmp-eth to avoid race
- fix: required argument for packaging capsule.cap
- chore: remove exma-q911 machine, now maintained in its own repository
- chore: remove unused files
- doc: reset I/O function table to pending-verification for exmp-q911

## v2.3.4 - 2026-08-03
- feat: b2b FW for reset function, ensure exmp-q911 PMIC LAN reset pin is 1.8v to 1.9v
- feat: default i2s mclk function on exma-q911, audio nodes added to exma-q911
- feat: enhance GPIO handling for QLI1.8 & QLI2.0 in exmp-serial-ctrl for serial com port
- feat: detect audio card index dynamically instead of hardcoding card 0
- feat: cyclonedx to sbom, CVE flow for CRA (later removed for build time)
- feat: kas-container build flow moved to jenkins, machines added to kas yml files
- feat: eDP panel power sequence, force rebuild of regulator power chain
- feat: innodisk socketcan driver & utility support (EGPC-B201, EMUC-B202, EMUC-B2S3)
- feat: RTC handling for shutdown/reboot and boot initialization
- feat: built-in USB onboard hub for improved USB stability, preventing BT firmware load issue
- fix: KERNEL_TECH_DTBOS so camera dtb is applied correctly
- fix: cpu dai of sound, enable all dapm on PRIMARY/QUATERNARY MI2S RX/TX interfaces
- fix: eth down/up issue - open SERDES clock on eth down event, restore link mode on up
- fix: MAC bound to controller instead of interface name (prevents eth0/eth1 order swap)
- fix: js-02 carrier only supports usb0 in high-speed mode
- fix: i2c order issue and i2c4 aliases typo
- chore: refine archive flow, default version bumped to 2.3.4
- chore: remove github action and CVE flow (build time reduction)
- chore: refine jenkinsfile path, flow, and file naming
- chore: rename machine folders, refine kas yml include structure
- chore: remove ainvr machine (maintained in forked repository going forward)

## v2.3.3 - 2026-06-05
- feat: language zh (Chinese) support
- feat: prototype of inno-ota for MCU firmware update
- feat: firmware for power/reset button function
- feat: AVL support for AX200 and BE200 Wi-Fi & BT modules, AX210 E-key Wi-Fi/BT module firmware
- feat: enable mclk on i2s interface with exmp-q911, force-enable headphone mic jack with acdb replacement
- feat: f81504a support for AVL
- feat: auto version and update documents
- fix: serial control rs485 handling in C code, merge rs232/rs422 usage, add arg to select device for set
- chore: update github action and remove license (cleanup)

## v2.3.2 - 2026-05-18
- feat: add CONFIG_DEVMEM
- feat: add 3165NGW E-key Wi-Fi/BT module firmware
- fix: random weston blank in QLI1.8 when two mdss displays are enabled
- fix: better pinctrl for DP0 to avoid affecting HPD
- fix: typo of dtsi path
- chore: merge some dtsi into dts and separate external dts, better device-tree management
- chore: refine sdk package flow, sbom.tar.gz archived
- chore: refine jenkinsfile and release flow, params before env, better options/selection
- chore: refine version section and fork-repo update flow in README

## v2.3.1 - 2026-04-24
- feat: exma-q911 default support for AGX-Orin carrier and DP
- feat: enable Intel E610 & X710 driver
- feat: exma-q911 carrier mux setting, refine for 2nd dev
- feat: pcie0 pinctrl and iommu for pcie1001
- feat: new password required at first boot (default off)
- fix: exmp-q911 default both ttyHS1/ttyHS2 in RS485 mode with bias and low active
- fix: no more USB device-tree conflict between Ubuntu (x09) and QLI1.8
- fix: typo of intel_wifi_lan.cfg
- chore: update jenkinsfile and sub-repo naming for QLI1.8, add GitHub Action trigger for sub-repo update
- doc: add I/O function table, known issues & solution for slow eMMC boot

## v2.3.0 - 2026-04-08
- feat: QLI1.8 IQ9 base device-tree, update device-tree/patch to QLI1.8-Ver1.1
- feat: DP2/DP3 support on exmp-q911 (SA8775P controller & PHY, msm-dp-desc kernel patch, weston patch)
- feat: eDP (DP3) GPIO pinctrl to mdss1, default backlight setting for exma-q911 DVT
- feat: eyzz-0401 device-tree for v4l2 on 120-pin MIPI
- feat: i2s pinctrl for exmp-q911, systemd-analyze for better logging
- fix: new GPIO numbering from TCA6408, default disable uart17
- fix: exmp RGMII setting, TZ firmware update to prevent crash
- fix: sdhc1/sdhc2 device-tree compile issue
- doc: AP test flow and release flow md5sum, refine doc-folder formatting

## v2.1.0 - 2026-03-11
- feat: exmp-q911 DVT board, CANFD loopback mode test pass
- feat: enable i2s and audio example with max98090 codec, exma-q911 PWM fan control
- feat: fan-ctrl and RTC functions, PROCHOT/THERMTRIP wired to exmp-q911 inno-fan
- feat: MAC address from TPM (MP flow writes MAC), qps615-triggered reset for 10G LAN PHY after boot
- feat: enable CAN0/CAN1 on js-02, enable vreg_l8c 3v3 supply for low-speed SDIO
- feat: ntfs & exfat filesystem support, boot console logo
- fix: exma-q911 SDIO 1v8 max frequency limited to 25MHz (HW limitation, DVT fix pending)
- fix: uart alias conflicts (984000/98c000), disable uart4 on js-02
- doc: pre-release EVT no longer counts toward `Major` version; refine release flow

## v1.1.0 - 2026-01-13
- feat: new machine exma-q911 (B2B project), split device-tree by machine name
- feat: enable excc-q911 audio function (wm8904 codec dai, mclk, dapm routing, acdb i2s routing)
- feat: B2B low-speed IO I2C serial, default I2C changed to exmp-q911 dtsi
- feat: USB 5G card support with libmbim for WWAN utility, Intel Wi-Fi support
- feat: QPS615 eth0 UXGMII enabling 10G LAN, QPS615 firmware and iommu/PCIe BDF per JS02 topology
- feat: NCT7802 fan controller driver, 2.5V MICBIAS for microphone
- fix: seperate exma-q911/exmp-q911 machine-specific settings and pinctrl naming
- fix: disable 1G LAN temporarily due to boot issue

## v0.1.0 - 2025-11-17
- feat: upgrade device-tree to QLI1.6
- feat: qprof verified via dummy bitbake, sysmonapple
- feat: split carrier-card hardware into excc-q911, rename excc-q911 to exmp-q911 per PM
- feat: default GPIO status using dummy pinctrl rather than gpio-hog (verified on exmp-q911)
- fix: default no patch for chicdk, keep entry point for camera team
- fix: remove fastrpc bbappend (no more URL issue in QLI1.6)

## v0.0.3 - 2025-11-13
- feat: q911 camera and sound-card device-tree, MIPI device-tree
- feat: SPI-to-CAN chipset driver, mcp2517fd spi2can under spi12 (HW update pending in DVT)
- feat: INA260 power reader and hwmon driver, PMIC thermal resistor definition
- feat: innodisk LAN PHY LED definition, TPM function, libx11/libgpiod utilities
- feat: q911 default usb0 host mode
- fix: MSM_BE debris issue by replacing patch in meta-qcom-hwe
- fix: MDIO cmd moved into driver, avoiding Ethernet quirk
- fix: longer timeout to prevent large firmware load failure

## v0.0.2 - 2025-09-09
- fix: workaround LAN reset - remove GPIO reset pin from PMIC, reset PHY via PMIC GPIO instead

## v0.0.1 - 2025-09-03
- feat: initial release based on exmp-q911 EVT
- feat: packagegroup-innodisk with innodisk module drivers & utilities
- feat: EGP2-X401/X403, EGPC-B4S1 driver & utility, EMUC2SOCKETCAN driver, libiagt for innoagent
- feat: MIPI EV2M-OOM1 & EVDM-OOM1 firmware and kernel modules, ATK11 firmware
- feat: AIW-170BQ-001 Wi-Fi module driver, opencv DNN lib, gst lib and iCAP lib for PPE
- feat: rust-cross-canadian, cargo, cmake, pip & bc
- fix: SSH PTY problem, QCOM USB firmware update, mDNS name and avahi name updates
