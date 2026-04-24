<!--
 Copyright (c) 2026 innodisk Crop.
 
 This software is released under the MIT License.
 https://opensource.org/licenses/MIT
-->
- [Release flow](#release-flow)
- [Rules of meta-layer](#rules-of-meta-layer)
- [I/O Function Table](#io-function-table)
- [SBOM](#sbom)
- [Knowing issue](#knowing-issue)
- [Ubuntu porting](#ubuntu-porting)
- [Todo](#todo)

# Release flow
1. Update README.md & CHANGELOG.md & version files: 
    - `recipes-apps/inno-version/files/BSP-version`
    - `recipes-bsp/base-files/files/issue`
2. Commit then add tag and push to github.
3. Build Image with `meta-iQ-extra__confidential`.
    ```bash
    export EXTRALAYERS="meta-qcom-qim-product-sdk meta-innodisk-iq meta-iQ-extra__confidential" \
    MACHINE=${MACHINE} DISTRO=qcom-wayland QCOM_SELECTED_BSP=custom && \
    source setup-environment
    ```
4. Clean build and test functions with stesting(keep the report for github release).
5. Test run [iqs-streampipe](https://github.com/InnoIPA/iQ-Studio/tree/main/benchmarks/iqs-streampipe) for checking soc internal functions.
6. Copy images and `ostree_repo` to make following folder structure.
    ```bash
    mkdir -p $version_folder
    mkdir -p $version_folder/sdk
    cd $version_folder
    rsync -avPz $bsp_folder/build-qcom-wayland/tmp-glibc/deploy/images/exmp-q911/qcom-multimedia-image ./exmp-q911
    rsync -avPz $bsp_folder/build-qcom-wayland/tmp-glibc/deploy/images/exmp-q911/ostree_repo ./exmp-q911/
    rsync -avPz $bsp_folder/build-qcom-wayland/tmp-glibc/deploy/sdk ./sdk/
    sync
    ```
    ```bash
    $version_folder
    ├── exmp-q911
    └── sdk
    ```
7. Generate md5sum.txt for all files and compress.
    ```bash
    cd sdk
    find ./* -type f -exec md5sum '{}' + > md5sum.txt
    cd ../exmp-q911
    find ./* -type f -exec md5sum '{}' + > md5sum.txt
    cd ../
    tar -cf - exmp-q911 | pigz -p $(nproc) > exmp-q911.tar.gz
    tar -cf - sdk | pigz -p $(nproc) > sdk.tar.gz
    md5sum ./* > md5sum.txt
    ```
8. Upload image & sdk to `R://`.
9. Github release and upload stesting & iqs-streampipe test result.
    - Update [I/O Function Table](#io-function-table) in release note. 
10.  Sync forked gitub and update tags, for example:
        ```bash
        git remote add upstream https://github.com/aiotads/meta-iQ__confidential
        git fetch upstream --tags
        git push origin --tags
        ```
11. Manually update main branch of forked github and release the latest tag.
- Maybe jenkins these release flow in future.

# Rules of meta-layer
- Style & rule following [oelint-adv](https://marketplace.visualstudio.com/items?itemName=kweihmann.oelint-vscode).
- Only bbappend allowed in recipes-bsp.
- Seperate all hw / fw change by using machine name in bb, consider multiple applicaiton environment.
- Definition of recipes folder
    | Name            | Description |
    |-----------------|-------------|
    | recipes-apps    | utilities or service |
    | recipes-bsp     | bbappends for customize platform |
    | recipes-core    | packagegroup-innodisk & main image bbappend |
    | recipes-library | external libraries |
    | recipes-modules | external modules or kernel modification |

# I/O Function Table
- This section shows the I/O function table sample for release note.
    - 🔵 : Working properly at previous version, should be okay but not verified yet.
    - 🟢 : Verified at current version and working properly.
    - 🟡 : Function works but unstable or have some quirk.
    - 🔴 : Having some issues.
<details>
<summary>exmp-q911 I/O function table</summary>

| Name            | Status | Verify | Description |
|-----------------|--------|--------|-------------|
| CN_GPIO1        | 🟢     | Z-scan ||
| DP1             | 🟡     | Mannual | Unstable sometimes weston desktop not working. |
| EDP_Panel       | 🟡     | Mannual | Unstable sometimes weston desktop not working. |
| DP2             | 🟡     | Mannual | Unstable sometimes weston desktop not working. |
| USB3_2_UP       | 🟢     | Detect ||
| USB3_2_DOWN     | 🟢     | Detect ||
| USB3_1_UP       | 🟢     | Detect ||
| USB3_1_DOWN     | 🟢     | Detect ||
| USBC_1          | 🟢     | Detect ||
| USB2_M2E1       | 🟢     | Detect ||
| USB2_M2B1       | 🟢     | Detect ||
| CN_USB2_1       | 🟢     | Detect ||
| LAN0            | 🟢     | Ping IP ||
| LAN1            | 🟢     | Ping IP ||
| PCIE_M2E1       | 🟢     | Link status ||
| PCIE_M2M1       | 🟢     | Link status ||
| SPI_TPM         | 🟢     | Detect ||
| TPM_FUNC        | 🟢     | Utility ||
| SOM_COM         | 🟢     | Loopback ||
| CN_COM1         | 🟢     | Loopback ||
| JP_SPI_I2C1-SPI | 🟢     | Loopback ||
| FAN_CTL         | 🟢     | Mannual ||
| AMP             | 🔴     | Mannual | snd card driver updated makes mclk not supply. |
| AUDIO_JACK      | 🔴     | Mannual | snd card driver updated makes mclk not supply. |
| MIPI_CAMERA     | 🟢     | Mannual ||
| CAN+PCIECARD    | 🟢     | Loopback ||
| RTC             | 🟢     | Detect ||
| I2C_wm8904      | 🟢     | Detect ||
| I2C_tca6408     | 🟢     | Detect ||
| I2C_ina260      | 🟢     | Detect ||
| I2C_SOM         | 🟢     | Detect ||
| JP_SPI_I2C1-I2C | 🟢     | Detect ||
</details>

<details>

# SBOM
- Files be find under following path:
    ```bash
    build-qcom-wayland/tmp-glibc/deploy/images/<machine>/*spdx*
    ```

# Knowing issue
<details>
<summary><b>eMMC boot time too slow, takes about 2 min.</b></summary>

- Root cause is about TZ issue, default `firmware-qcom-partconf_1.0.bb` is pulling binary.zip from qualcomm upstream.

- Re-compile the binary files by using following workflow could slove this issue:
1. Modify `<image-path>/partition_emmc/partition.xml`, move `efi` & `persist` to the end after `system`.
    ```xml
    <?xml version="1.0"?>
    <configuration>
        <parser_instructions>
            <!-- NOTE: entries here are used by the parser when generating output -->
            <!-- NOTE: each filename must be on it's own line as in variable=value-->
            WRITE_PROTECT_BOUNDARY_IN_KB    = 65536
            GROW_LAST_PARTITION_TO_FILL_DISK=true
            ALIGN_PARTITIONS_TO_PERFORMANCE_BOUNDARY = true
            PERFORMANCE_BOUNDARY_IN_KB = 4
        </parser_instructions>

        <!-- NOTE: "physical_partition" are listed in order and apply to devices such as eMMC cards that have (for example) 3 physical partitions -->
        <!-- This is physical partition 0 -->
        <physical_partition>
            <!-- NOTE: Define information for each partition, which will be created in order listed here -->
            <!-- NOTE: Place all "readonly=true" partitions side by side for optimum space usage -->
            <!-- NOTE: If OPTIMIZE_READONLY_PARTITIONS=true, then partitions won't be in the order listed here -->
            <!--       they will instead be placed side by side at the beginning of the disk -->
            <partition label="xbl_a" 		size_in_kb="5120" 		type="DEA0BA2C-CBDD-4805-B4F9-F428251C3E98" 	 										system="true" dontautomount="true" filename="xbl.elf"/>
            <partition label="xbl_config_a" size_in_kb="512" 		type="5A325AE4-4276-B66D-0ADD-3494DF27706A" 	 										system="true" dontautomount="true" filename="xbl_config.elf"/>
            <partition label="shrm_a" 		size_in_kb="80" 		type="CB74CA22-2F0D-4B82-A1D6-C4213F348D73" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="shrm.elf"/>
            <partition label="xbl_b" 		size_in_kb="5120" 		type="7A3DF1A3-A31A-454D-BD78-DF259ED486BE" 	 										system="true" dontautomount="true" filename="xbl.elf"/>
            <partition label="xbl_config_b" size_in_kb="512" 		type="F462E0EA-A20E-4B10-867A-2D4455366548" 	 										system="true" dontautomount="true" filename="xbl_config.elf"/>
            <partition label="shrm_b" 		size_in_kb="80" 		type="39FD6C00-49EB-6BD1-6899-2FB849DD4F75" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="shrm.elf"/>
            <partition label="ddr_a" 		size_in_kb="1024" 		type="20A0C19C-286A-42FA-9CE7-F64C3226A794" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="ddr_b" 		size_in_kb="1024" 		type="325DEF02-1305-44A3-AA8D-AC82FEBE220E" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="aop_a" 		size_in_kb="256" 		type="D69E90A5-4CAB-0071-F6DF-AB977F141A7F" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="aop.mbn"/>
            <partition label="uefi_a" 		size_in_kb="5120" 		type="400FFDCD-22E0-47E7-9A23-F16ED9382388" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="uefi.elf"/>
            <partition label="uefi_b" 		size_in_kb="5120" 		type="9F234B5B-0EFB-4313-8E4C-0AF1F605536B" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="uefi.elf"/>
            <partition label="core_nhlos_a" size_in_kb="174080" 	type="6690B4CE-70E9-4817-B9F1-25D64D888357" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>	
            <partition label="core_nhlos_b" size_in_kb="174080" 	type="77036CD4-03D5-42BB-8ED1-37E5A88BAA34" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="uefisecapp_a" size_in_kb="2048" 		type="BE8A7E08-1B7A-4CAE-993A-D5B7FB55B3C2" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="uefi_sec.mbn"/>
            <partition label="uefisecapp_b" size_in_kb="2048" 		type="538CBDBA-D4A4-4438-A466-D7B356FAC165" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="uefi_sec.mbn"/>
            <partition label="xbl_ramdump_a" size_in_kb="2048" 		type="0382F197-E41F-4E84-B18B-0B564AEAD875" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="XblRamdump.elf"/>
            <partition label="xbl_ramdump_b" size_in_kb="2048" 		type="C3E58B09-ABCB-42EA-9F0C-3FA453FA892E" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="XblRamdump.elf"/>
            <partition label="dtb_a" 		size_in_kb="65536" 		type="2A1A52FC-AA0B-401C-A808-5EA0F91068F8" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="dtb.bin"/>
            <partition label="dtb_b" 		size_in_kb="65536" 		type="A166F11A-2B39-4FAA-B7E7-F8AA080D0587" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="dtb.bin"/>
            <partition label="tz_a" 		size_in_kb="4000" 		type="A053AA7F-40B8-4B1C-BA08-2F68AC71A4F4" 	bootable="false"	readonly="true" 	system="true" dontautomount="true" filename="tz.mbn"/>
            <partition label="hyp_a" 		size_in_kb="65536" 		type="E1A6A689-0C8D-4CC6-B4E8-55A4320FBD8A" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="hypvm.mbn"/>
            <partition label="TZAPPS" 		size_in_kb="320" 		type="14D11C40-2A3D-4F97-882D-103A1EC09333" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="mdtpsecapp_a" size_in_kb="4096" 		type="EA02D680-8712-4552-A3BE-E6087829C1E6" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="mdtp_a" 		size_in_kb="32768" 		type="3878408A-E263-4B67-B878-6340B35B11E3" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="keymaster_a" 	size_in_kb="512" 		type="A11D2A7C-D82A-4C2F-8A01-1805240E6626" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="devcfg_a" 	size_in_kb="128" 		type="F65D4B16-343D-4E25-AAFC-BE99B6556A6D" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="devcfg_iot.mbn"/>
            <partition label="qupfw_a" 		size_in_kb="128" 		type="21D1219F-2ED1-4AB4-930A-41A16AE75F7F" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="qupfw_b" 		size_in_kb="128" 		type="04BA8D53-5091-4958-9CA1-0FE0941D2CBC" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="cpucp_a" 		size_in_kb="1024" 		type="1E8615BD-6D8C-41AD-B3EA-50E8BF40E43F" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="cpucp.elf"/>
            <partition label="apdp_a" 		size_in_kb="64" 		type="E6E98DA2-E22A-4D12-AB33-169E7DEAA507" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="multiimgoem_a" size_in_kb="32" 		type="E126A436-757E-42D0-8D19-0F362F7A62B8" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="multi_image.mbn"/>
            <partition label="multiimgqti_a" size_in_kb="32" 		type="846C6F05-EB46-4C0A-A1A3-3648EF3F9D0E" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="multi_image_qti.mbn"/>
            <partition label="imagefv_a" 	size_in_kb="1024" 		type="17911177-C9E6-4372-933C-804B678E666F" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="imagefv.elf"/>
            <partition label="usb4fw_a" 	size_in_kb="61" 		type="3FA03C7A-9FDC-498B-A2A8-DE11EE339790" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>		
            <partition label="devinfo" 		size_in_kb="4" 			type="65ADDCF4-0C5C-4D9A-AC2D-D90B5CBFCD03" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="dip" 			size_in_kb="1024" 		type="4114B077-005D-4E12-AC8C-B493BDA684FB" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="spunvm" 		size_in_kb="8192" 		type="E42E2B4C-33B0-429B-B1EF-D341C547022C" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="splash" 		size_in_kb="33424" 		type="AD99F201-DC71-4E30-9630-E19EEF553D1B" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="limits" 		size_in_kb="4" 			type="10A0C19C-516A-5444-5CE3-664C3226A794" 	bootable="false" 	readonly="true"/>
            <partition label="logfs" 		size_in_kb="8192" 		type="BC0330EB-3410-4951-A617-03898DBE3372" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="cateloader" 	size_in_kb="2048" 		type="AA9A5C4C-4F1F-7D3A-014A-22BD33BF7191" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="emac" 		size_in_kb="512" 		type="E7E5EFF9-D224-4EB3-8F0B-1D2A4BE18665" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="uefivarstore" size_in_kb="512" 		type="165BD6BC-9250-4AC8-95A7-A93F4A440066" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="secdata" 		size_in_kb="128" 		type="76CFC7EF-039D-4E2C-B81E-4DD8C2CB2A93" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="quantumsdk" 	size_in_kb="40960" 		type="AA9A5C4C-4F1F-7D3A-014A-22BD33BF7191" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="quantumfv" 	size_in_kb="512" 		type="80C23C26-C3F9-4A19-BB38-1E457DACEB09" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="toolsfv" 		size_in_kb="1024" 		type="97745ABA-135A-44C3-9ADC-05616173C24C" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="tools.fv"/>
            <partition label="softsku" 		size_in_kb="8" 			type="69CFD37F-3D6B-48ED-9739-23015606BE65" 	bootable="false" 	readonly="false"	system="true" dontautomount="true" filename=""/>
            <partition label="gearvm_a" 	size_in_kb="16000" 		type="06EF844E-08FC-494E-89EB-396D4D6C5B27" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            
            <partition label="diag_log" 	size_in_kb="65536" 		type="3989AF30-5C02-4154-AD00-1D34C816CAC1" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="pvm_log" 		size_in_kb="65536" 		type="2889C942-FF80-4DA8-A5B8-3F32F285C0D8" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="gvm_log" 		size_in_kb="65536" 		type="78EBFD49-E8B1-4E75-ABC0-3F2DBC7428DD" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
            <partition label="aop_b" 		size_in_kb="256" 		type="B8B27C4C-4B5B-8AB2-502F-A792B590A896" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="aop.mbn"/>
            <partition label="tz_b" 		size_in_kb="4000" 		type="C832EA16-8B0D-4398-A67B-EBB30EF98E7E" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="tz.mbn"/>
            <partition label="hyp_b" 		size_in_kb="65536" 		type="3D3E3AD2-8FF3-4975-A7E7-0E8A10B69F0D" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="hypvm.mbn"/>
            <partition label="keymaster_b" 	size_in_kb="512" 		type="441EEF80-DE15-4522-9995-563398D94889" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="devcfg_b" 	size_in_kb="128" 		type="4E820A31-17E3-447D-B32D-FB339F7EA1A2" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="devcfg_iot.mbn"/>
            <partition label="cpucp_b" 		size_in_kb="1024" 		type="6C1111FB-5354-41DE-AC17-5B6E542BE836" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="cpucp.elf"/>
            <partition label="apdp_b" 		size_in_kb="64" 		type="110F198D-8174-4193-9AF1-5DA94CDC59C9" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="multiimgoem_b" size_in_kb="32" 		type="3E3E3ECD-C512-4F95-9144-6063826A8970" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="multi_image.mbn"/>
            <partition label="multiimgqti_b" size_in_kb="32" 		type="D30C8B21-DDD9-45B6-8DE0-3165D34395C9" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="multi_image_qti.mbn"/>
            <partition label="imagefv_b" 	size_in_kb="1024" 		type="920CFC3D-7285-4A47-9C1C-4A87590E0687" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="imagefv.elf"/>
            <partition label="gearvm_b" 	size_in_kb="16000" 		type="4D09E70E-F349-11ED-A05B-0242AC120003" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="logdump" 		size_in_kb="524288" 	type="5AF80809-AABB-4943-9168-CDFC38742598" 	 										system="true" dontautomount="true" filename=""/>				
            <partition label="recoveryinfo" size_in_kb="4"			type="7374B391-291C-49FA-ABC2-0463AB5F713F" 											system="true" dontautomount="true" filename=""/>
            <partition label="xbl_logs" 	size_in_kb="1024" 		type="F7EECB66-781A-439A-8955-70E12ED4A7A0" 	 										system="true" dontautomount="true" filename=""/>
            <partition label="SYSFW_VERSION" size_in_kb="4" 		type="3C44F88B-1878-4C29-B122-EE78766442A7" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename=""/>
            <partition label="system" 		size_in_kb="16777216" 	type="B921B045-1DF0-41C3-AF44-4C6F280D3FAE" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename="system.img" 	sparse="false"/>    </physical_partition>

            <partition label="efi" 			size_in_kb="524288" 	type="C12A7328-F81F-11D2-BA4B-00A0C93EC93B" 	bootable="false" 	readonly="true" 	system="true" dontautomount="true" filename="efi.bin"/>
            <partition label="persist" 		size_in_kb="30720" 		type="0FC63DAF-8483-4772-8E79-3D69D8477DE4" 	bootable="false" 	readonly="false" 	system="true" dontautomount="true" filename=""/>
        <physical_partition>
            <partition label="cdt" size_in_kb="4" type="A19F205F-CCD8-4B6D-8F1E-2D9BC24CFFB1" bootable="false" readonly="true" filename=""/>
            <partition label="last_parti" size_in_kb="0" type="00000000-0000-0000-0000-000000000000" system="true" readonly="true" filename=""/>
        </physical_partition>

    </configuration>
    ```
2. Compile binary file by using `ptool`.
    ```bash
    cd <tool-path>
    git clone https://github.com/qualcomm-linux/qcom-ptool.git
    cd <image-path>/partition_emmc/
    python3 <tool-path>/qcom-ptool/ptool.py -x ./partition.xml
    ```
3. All the binary files in `<image-path>/partition_emmc/` will be replaced
4. Use following command to flash image into eMMC.
    ```bash
    cd <image-path>
    cp ./partition_emmc/* ./
    qdl --storage emmc \
        --include <image-path>/ \
        <image-path>/prog_firehose_ddr.elf \
        <image-path>/rawprogram0.xml \
        <image-path>/rawprogram1.xml \
        <image-path>/patch0.xml \
        <image-path>/patch1.xml
    ```
</details>

# Ubuntu porting
- Current [released ubuntu](https://ubuntu.com/download/qualcomm-iot) from Canonical are updated dynamically.
- Should becareful about some little difference in device-tree, caused issue and solution are describe at [here](../recipes-bsp/q911/files/ubuntu.dtsi).
- [This repository](https://github.com/aiotads/iQ-ubuntu__confidential) provides debian pkg for fixing qurik between realtek phy and qcom mac(`exmp-q911`).
<details>
<summary>Workflow for porting ubuntu:</summary>

1. Download [released ubuntu](https://ubuntu.com/download/qualcomm-iot) which for qcs9075.
2. Update `dtb.bin` from yocto which sould fix issue about [here](../recipes-bsp/q911/files/ubuntu.dtsi).
3. Update firmware files from yocto if in need.
4. Flash images.
5. Install `iq-ubuntu.deb` from [here](https://github.com/aiotads/iQ-ubuntu__confidential) after system boot if in need.
6. Check if every thing is working fine, if no we could only modify following parts:
    - firmware files
    - `iq-ubuntu.deb`
    - `dtb.bin`
</details>

# Todo
- [ ] Jenkins whole release flow.

