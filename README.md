<!--
 Copyright (c) 2025 innodisk Crop.
 
 This software is released under the MIT License.
 https://opensource.org/licenses/MIT
-->

- [Overview](#overview)
- [Release table](#release-table)
- [Requirement](#requirement)
- [Build Image \& SDK from BSP](#build-image--sdk-from-bsp)
- [Flash Image](#flash-image)
- [FAQ](#faq)
- [Reference](#reference)

# Overview
This repository provide the bsp of platfroms which base on [Qualcomm yocto](https://github.com/quic-yocto/qcom-manifest) :
- exmp-q911: qcs9075 COMHPC mini module.
- qcs9075-iq-9075-evk: rb8 for evaluation.

# Release table
| Version | Date         | Status    | Description |
|---------|--------------|-----------|-------------|
| v0.0.1  | 2025-09-03   | Released  | Initial release based on exec-q911 EVT. |
| v0.0.2  | 2025-09-08   | Released  | Enable ethernet0. |
| v0.0.3  | 2025-11-13   | Released  | Enable mipi, pmic thermal, lan phy led, ina260 hwmon, ethernet1. Add tpm utility, libraries for i-cap. |
| v0.1.0  | 2025-11-14   | Released  | Upgrade to QLI1.6-1.2.1. |
| v1.1.0  | 2026-01-13   | Released  | New machine exma-q911 EVT & enable excc-q911 audio function. |
| v2.1.0  | 2026-03-12   | Released  | exmp-q911 DVT & some issue fixed. |
| v2.3.0  | 2026-04-08   | Released  | Upgrade to QLI1.8-1.1. |
| v2.3.1  | 2026-04-23   | Released  | Github action to InnoIPA. |
- Check [version rules](doc/VERSION.md).

# Requirement
- Recommend build machine:
  - CPU x86 over 10 threads
  - RAM over 32GB
  - Ubuntu over 22.04
- Utilities in need:
    ```bash
    sudo apt update
    sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential \
        chrpath socat cpio python3 python3-pip python3-pexpect xz-utils \
        debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa \
        libsdl1.2-dev pylint xterm python3-subunit mesa-common-dev zstd \
        liblz4-tool locales tar python-is-python3 file libxml-opml-simplegen-perl \
        vim whiptail repo
    ```

# Build Image & SDK from BSP
1. Get Qualcomm QLI source according to `Minor` in version.
    ```bash
    repo init -u https://github.com/quic-yocto/qcom-manifest -b qcom-linux-scarthgap -m qcom-6.6.119-QLI.1.8-Ver.1.1_qim-product-sdk-2.3.1.xml
    ```
2. Pull BSP files.
    ```bash
    repo sync
    ```
3. Download this meta layer into BSP & check out to specific tag if in need.
    ```bash
    cd layers
    git clone <this-repository> meta-innodisk-iq
    git checkout <tag>
    cd ..
    ```
4. Setup environment.
    - Currently support `MACHINE`: `exmp-q911`, `qcs9075-iq-9075-evk`
    - General build:
        ```bash
        export EXTRALAYERS="meta-qcom-qim-product-sdk meta-innodisk-iq" \
        MACHINE=exmp-q911 DISTRO=qcom-wayland QCOM_SELECTED_BSP=custom && \
        source setup-environment
        ```
    - Build with firmware source:
        ```bash
        export EXTRALAYERS="meta-qcom-qim-product-sdk meta-qcom-extras meta-innodisk-iq" \
        CUST_ID=213195 FWZIP_PATH=<path> MACHINE=exmp-q911 DISTRO=qcom-wayland QCOM_SELECTED_BSP=custom && \
        source setup-environment
        ```
5. Build 
    - Image:
        ```bash
        bitbake qcom-multimedia-image
        ```
    - SDK:
        ```bash
        bitbake -c do_populate_sdk qcom-multimedia-image
        ```
- The image and sdk result will be under following path.
    ```bash
    ./tmp-glibc/deploy/images/<MACHINE>/qcom-multimedia-image
    ./tmp-glibc/deploy/sdk
    ```

# Flash Image
- Follow [this page](https://github.com/InnoIPA/iQ-Studio/blob/main/tutorials/starting-guides/flash-image/README.md).
- Workflow are based on [tutorial](https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-254/flash_images.html) from Qualcomm.


# FAQ
- Fetch failed during bitbake building.
  - Copy the fetch cmd & manually fetch or apply following cmd preventing git timeout because of poor connection.
  ```bash
  git config --global http.lowSpeedLimit 0
  git config --global http.lowSpeedTime 999999
  ```
- Ubuntu 24.04 namespaces not usable issue.
  - Error log after bitbake cmd.
    ```
    ERROR: User namespaces are not usable by BitBake, possibly due to AppArmor.
    See https://discourse.ubuntu.com/t/ubuntu-24-04-lts-noble-numbat-release-notes/39890#unprivileged-user-namespace-restrictions
    ```
  - Solution:
    ```bash
    # Fix temporarily
    sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
    # Fix permanently
    echo "kernel.apparmor_restrict_unprivileged_userns = 0" | sudo tee /etc/sysctl.d/60-apparmor-userns.conf
    sudo sysctl --system
    ```
- Developer note:
    - See [DEVELOPMENT.md](doc/DEVELOPMENT.md).

# Reference
- https://github.com/qualcomm-linux/qcom-manifest
- https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-115/qualcomm-linux-docs-home.html
- https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-254/build_landing_page.html