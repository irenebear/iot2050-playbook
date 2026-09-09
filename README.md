# iot2050-playbook

Hands-on guides for the **Siemens SIMATIC IOT2050** industrial IoT gateway — from building and customizing the Debian image, running it without real hardware, to wiring it up with S7-1200 PLCs, Node-RED, and cloud reporting.

## Guides

### Image Building & Customization
- [Build the IoT2050 Example Image with GitHub Actions](docs/Build_IoT2050_Image_with_GitHub_Actions.md) — No Docker needed: build `Image.wic` from a V-tag in the cloud.
- [Customize the Official Image with open62541 (OPC UA)](docs/IoT2050_Custom_Image_open62541.md) — Minimal-change integration of open62541 via a kas fragment.

### Running Without Hardware (QEMU / WSL2)
- [Run the Official Image on Win11 + WSL2 + QEMU](docs/IoT2050_WSL2_QEMU_Official_Script.md) — Official script, no real hardware required.
- [Enlarge the QEMU Disk Image (.wic)](docs/IoT2050_QEMU_WIC_Resize.md) — Two approaches: 4.9G → 8G / 20G.

### Industrial Control & Communication (CODESYS / OPC UA / S7-1200)
- [Turn Your IoT2050 into a SoftPLC with CODESYS](docs/IoT2050_CODESYS_SoftPLC.md) — Install CODESYS Control for Linux ARM64 SL: RT image build → one-click deploy → program running.
- [IoT2050 ↔ S7-1200 OPC UA Communication](docs/IoT2050_S7-1200_OPC_UA_Communication.md) — Full flow with an open62541 client.
- [S7-1200 Data to Cloud](docs/S7-1200_Data_to_Cloud.md) — 1000-point acquisition, SQLite archiving, MQTT reporting, with source code & 4 real-world findings.

### Node-RED
- [Install Node-RED Custom Nodes Offline](docs/IoT2050_Offline_Node-RED_Nodes.md) — Complete offline tutorial with common pitfalls.
- [Diagnose Node-RED on the IoT2050](docs/Diagnose_Node-RED_on_IoT2050.md) — Troubleshooting handbook with a real crash case.

### Python Development & Debugging
- [Create a Python venv on the IoT2050](docs/IoT2050_Python_venv_Setup.md) — Debian 12's PEP 668 pip ban and the right way forward.
- [Remote-Debug Python with VS Code Remote-SSH](docs/VS_Code_Remote_Debug_IoT2050_Python.md) — From connecting to setting breakpoints.
