# ESP32 Setup Guide

Building from source is not required for ESP32. You can simply flash the pre compiled firmware binary.

## Installation Methods

### Method 1: PlatformIO (Recommended for Developers)

If you are using **VS Code + platformIO**, the easiest way to flash the pre-compiled binary is using the platformIO CLI.

1. Download the latest `firmware.bin` from the [Releases](../../releases) page.
2. Open your terminal in VS Code and run the following command:

```bash
pio run --target upload --upload-port /dev/ttyUSB0 --firmware firmware.bin
```

### Method 2: esptool.py (CLI)

1. Download the latest `firmware.bin` from the [Releases](../../releases) page.
2. Flash the binary using esptool:

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 write_flash -z 0x1000 firmware.bin
```

### Method 3: Web Browser (No Installation Required)

For GUI users or quick testing without installing local tools:

1. Open the [ESP Web Flasher](https://espressif.github.io/esptool-js/) in chrome or edge.
2. Connect your ESP32 via USB, select your port, and upload `firmware.bin` at offset 0x1000.
