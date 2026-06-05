<!--
 Copyright (c) 2026 innodisk Crop.
 
 This software is released under the MIT License.
 https://opensource.org/licenses/MIT
-->
# exmp-q911 I/O Function Table
- 🔵 : Working properly at previous version, should be okay but not verified yet.
- 🟢 : Verified at current version and working properly.
- 🟡 : Function works but unstable or have some quirk.
- 🔴 : Having some issues.

| Name            | Status | Verify | Description |
|-----------------|--------|--------|-------------|
| CN_GPIO1        | 🔵     | Z-scan ||
| DP1             | 🔵     | Mannual ||
| EDP_Panel       | 🔵     | Mannual ||
| DP2             | 🔵     | Mannual ||
| USB3_2_UP       | 🔵     | Detect ||
| USB3_2_DOWN     | 🔵     | Detect ||
| USB3_1_UP       | 🔵     | Detect ||
| USB3_1_DOWN     | 🔵     | Detect ||
| USBC_1          | 🔵     | Detect ||
| USB2_M2E1       | 🔵     | Detect ||
| USB2_M2B1       | 🔵     | Detect ||
| CN_USB2_1       | 🔵     | Detect ||
| LAN0            | 🔵     | Ping IP ||
| LAN1            | 🔵     | Ping IP ||
| PCIE_M2E1       | 🔵     | Link status ||
| PCIE_M2M1       | 🔵     | Link status ||
| SPI_TPM         | 🔵     | Detect ||
| TPM_FUNC        | 🔵     | Utility ||
| SOM_COM         | 🔵     | Loopback ||
| CN_COM1         | 🔵     | Loopback ||
| JP_SPI_I2C1-SPI | 🔵     | Loopback ||
| FAN_CTL         | 🔵     | Mannual ||
| AMP             | 🟢     | Mannual ||
| AUDIO_JACK      | 🟢     | Mannual ||
| MIPI_CAMERA     | 🟢     | Mannual ||
| CAN+PCIECARD    | 🔵     | Loopback ||
| RTC             | 🔵     | Detect ||
| I2C_wm8904      | 🔵     | Detect ||
| I2C_tca6408     | 🔵     | Detect ||
| I2C_ina260      | 🔵     | Detect ||
| I2C_SOM         | 🔵     | Detect ||
| JP_SPI_I2C1-I2C | 🔵     | Detect ||
