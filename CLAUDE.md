# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BLE CTF is a Bluetooth Low Energy Capture The Flag framework for ESP32. It implements a GATT server with 20 progressive flags that teach BLE concepts (reads, writes, notifications, indications, MAC spoofing, MTU negotiation, etc.). Flags are MD5 sums truncated to 20 characters.

## Build & Flash

### Docker Build (recommended)
```bash
docker build -t blectf .
docker run -it -v ./:/ble_ctf --name blectf blectf
# Inside container:
cd /ble_ctf
idf.py set-target esp32
idf.py build
```

### Native Build (requires ESP-IDF v5.5+)
```bash
idf.py set-target esp32  # or esp32s3, esp32c6
idf.py build
```

### Flash Precompiled Binaries
```bash
esptool.py -p /dev/ttyUSB0 -b 460800 --before default_reset --after hard_reset \
  --chip esp32 write_flash --flash_mode dio --flash_size 2MB --flash_freq 40m \
  0x1000 build/bootloader/bootloader.bin \
  0x8000 build/partition_table/partition-table.bin \
  0x10000 build/ble_ctf.bin
```

### Flash from Source Build
```bash
idf.py flash -p /dev/ttyUSB0
```

## Architecture

All CTF logic lives in a single file: `main/gatts_table_creat_demo.c` (~1130 lines).

**GATT structure**: 57 attributes in `gatt_db[]` define the service. The attribute index enum in `main/gatts_table_creat_demo.h` maps symbolic names (e.g., `IDX_CHAR_FLAG_SCORE`) to positions in the table.

**Runtime handle mapping**: `blectf_handle_table[]` maps attribute indices to runtime GATT handles, populated on `ESP_GATTS_CREAT_ATTR_TAB_EVT`.

**Flag state**: `flag_state[20]` tracks completion (T/F/H). Score is maintained in `score` and rendered to `score_read_value[]` as "Score: N/20".

**Event flow**: `gatts_profile_event_handler()` processes all GATT events (reads, writes, connections). Flag validation happens in the `ESP_GATTS_WRITE_EVT` handler — writes to the submission handle (`0x002c`, index 44) are compared against hardcoded MD5 strings.

**BLE config**: Device advertises as "BLECTF" on service UUID `0x00FF`, BLE-only mode, max 3 connections.

**Characteristic UUIDs**: Sequential 16-bit UUIDs from `0xFF01` (score) through `0xFF17` (flag 20), each corresponding to a specific challenge type.
