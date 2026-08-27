<!--
 Copyright (c) 2025 innodisk Crop.
 
 This software is released under the MIT License.
 https://opensource.org/licenses/MIT
-->

- [Overview](#overview)
- [Latest release](#latest-release)
- [Requirement](#requirement)
- [Build Image](#build-image)
- [Flash Image](#flash-image)
- [Development](#development)
- [FAQ](#faq)
- [Reference](#reference)

# Overview
This repository provide the bsp for following platfroms which base on [Qualcomm yocto](https://github.com/quic-yocto/qcom-manifest) :

| Machine | Platform Description | Current Position |
|---|---|---|
| `exmp-q911` | QCS9075 COM-HPC Mini module | Main DVT platform baseline |
| `qcs9075-iq-9075-evk` | RB8 / QCS9075 EVK | Reference evaluation baseline |

# Latest release 
| [Version](doc/VERSION.md) | Date | Status | Description |
|---------|--------------|-----------|-------------|
| v2.3.5  | 2026-08-21   | Released  | Qcom fw for IQ9 with PMIC PWR hard-reset, capsule OTA flow, exmp eth0/eth1 naming fix. |

<details>
<summary>Release history</summary>

| [Version](doc/VERSION.md) | Date | Status | Description |
|---------|--------------|-----------|-------------|
| v2.3.4  | 2026-07-31   | Released  | Feat kas, refine meta-layer, fix I/O issues, support sbom. |
| v2.3.3  | 2026-06-04   | Released  | Fix audio and rs232/422/485 function, support language zh. |
| v2.3.2  | 2026-05-18   | Released  | Fix wayland issue with multiple screen. |
| v2.3.1  | 2026-04-23   | Released  | Github action to InnoIPA. |
| v2.3.0  | 2026-04-08   | Released  | Upgrade to QLI1.8-1.1. |
| v2.1.0  | 2026-03-12   | Released  | exmp-q911 DVT & some issue fixed. |
| v1.1.0  | 2026-01-13   | Released  | New machine exma-q911 EVT & enable excc-q911 audio function. |
| v0.1.0  | 2025-11-14   | Released  | Upgrade to QLI1.6-1.2.1. |
| v0.0.3  | 2025-11-13   | Released  | Enable mipi, pmic thermal, lan phy led, ina260 hwmon, ethernet1. Add tpm utility, libraries for i-cap. |
| v0.0.2  | 2025-09-08   | Released  | Enable ethernet0. |
| v0.0.1  | 2025-09-03   | Released  | Initial release based on exec-q911 EVT. |
</details>

# Requirement
- Recommend build machine:
  - CPU x86 over 10 threads
  - RAM over 32GB
  - Ubuntu over 22.04
- Utilities in need:
  - kas 5.1

# Build Image
1. Extract or download repository into directory which named `meta-innodisk-iq`.
    > [!NOTE]NOTICE
    > This step expects the customer contact innodisk to obtain a snapshot of the meta layer. [meta-iQ__manifest](https://github.com/InnoIPA/meta-iQ__manifest.git) is for viewing only.
    - If you received a zip file, unzip it and move the extracted folder into `layers`, renaming it to `meta-innodisk-iq` if needed.
        ```bash
        unzip meta-innodisk-iq.zip
        mv meta-innodisk-iq layers/meta-innodisk-iq
        ```
    - Or, if you have repository access, clone it and optionally check out a specific tag.
        ```bash
        cd layers
        git clone <this-repository> meta-innodisk-iq
        git -C meta-innodisk-iq checkout <tag>   # optional
        cd ..
        ```
2. Build image by using kas, for example target machine is `exmp-q911`.
    > [!NOTE]NOTICE
    [kas](https://github.com/siemens/kas) encapsulates the BitBake build process within a container, minimizing host environment differences and simplifying Yocto project management.

    Swap `kas/exmp-q911.yml` to target another machine from the table above.
    ```bash
    # Container build image
    kas-container build meta-innodisk-iq/kas/exmp-q911.yml:meta-innodisk-iq/kas/innodisk-distro.yml
    ```
    ```bash
    # Container build image and generate SBOM (SPDX + CycloneDX)
    kas-container build meta-innodisk-iq/kas/exmp-q911.yml:meta-innodisk-iq/kas/innodisk-distro.yml:meta-innodisk-iq/kas/sbom.yml
    ```
    ```bash
    # Container build sdk
    kas-container build meta-innodisk-iq/kas/exmp-q911.yml:meta-innodisk-iq/kas/innodisk-distro.yml \
        -c populate_sdk
    ```
    ```bash
    # Open container with console
    kas-container shell meta-innodisk-iq/kas/exmp-q911.yml:meta-innodisk-iq/kas/innodisk-distro.yml
    ```
    ```bash
    # optional : for shared download & sstate-cache folder
    export DL_DIR="../downloads"
    export SSTATE_DIR="../sstate-cache"
    ```
    ```bash
    # optional : with less cpu used
    kas-container --runtime-args "--cpus=8" build
    ```

- Results under `tmp-glibc/deploy/`:
    | Results | Path |
    |--------|------|
    | Image | `deploy/images/<MACHINE>/qcom-multimedia-image` |
    | SDK | `deploy/sdk` |
    | SBOM (SPDX 2.2) | `deploy/spdx/2.2/` |
    | SBOM (CycloneDX) | `deploy/cyclonedx-export/<image-recipe>/` |

# Flash Image
- Follow [this page](https://github.com/InnoIPA/iQ-Studio/blob/main/tutorials/starting-guides/flash-image/README.md).
- Workflow are based on [tutorial](https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-254/flash_images.html) from Qualcomm.

# Development
- For more information about development, please refer to [DEVELOPMENT.md](doc/DEVELOPMENT.md).

# FAQ
<details>
<summary><b>Fetch failed during bitbake building</b></summary>
<br>

**Issue：**
Download failed because about internet issue or poor connection with download server.

**Solution：**
Copy the fetch cmd & manually fetch or apply following cmd preventing git timeout. :
```bash
git config --global http.lowSpeedLimit 0
git config --global http.lowSpeedTime 999999
```
</details>

<details> <summary><b>Ubuntu 24.04 namespaces not usable issue</b></summary> <br>


**Issue：**
Error log after bitbake/kas-container cmd.
```
ERROR: User namespaces are not usable by BitBake, possibly due to AppArmor.
See https://discourse.ubuntu.com/t/ubuntu-24-04-lts-noble-numbat-release-notes/39890#unprivileged-user-namespace-restrictions
```

**Solution：**
```bash
# Fix temporarily
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
# Fix permanently
echo "kernel.apparmor_restrict_unprivileged_userns = 0" | sudo tee /etc/sysctl.d/60-apparmor-userns.conf
sudo sysctl --system
```
</details>

<details> <summary><b>kas-container failed that yml file not found.</b></summary> <br>


**Issue：**
All meta-layers need git management for kas-container, this issue of snapshot.zip will be fixed since 2.3.4.

**Solution：**
- Make sure this meta-layer have git management if not exist use following command.
  ```bash
  cd meta-innodisk-iq
  git init
  ```
- Clean all exist meta-layers then run the kas-container again.
</details>

# Reference
- https://github.com/qualcomm-linux/qcom-manifest
- https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-254/build_landing_page.html