# CEREBRAL STRATUM Firmware

The CEREBRAL STRATUM firmware runs on the **ESP32-S3** and is written in **Rust** (via `esp-rs` / `esp-idf-hal`).

## Overview

The firmware is responsible for:

- Reading vehicle telemetry from the CAN bus (via ESP32-S3 TWAI controller and Adafruit CAN Pal TJA1051T/3 transceiver)
- Transmitting data over LTE (MQTT) with automatic satellite fallback (CoAP over NTN via Quectel BG95-S5)
- Managing an optoisolated 12V relay output for user-defined ancillary functions
- Operating continuously when the vehicle is off (LiFePO4 battery backup, deep sleep, 3GPP PSM)
- Receiving OTA firmware updates over LTE or satellite

## Hardware

| Component         | Part                                  |
|-------------------|---------------------------------------|
| MCU               | Espressif ESP32-S3-DevKitC-1          |
| Communications    | Quectel BG95-S5 (LTE Cat M1 + NTN)   |
| CAN Transceiver   | Adafruit CAN Pal ADA5708 (TJA1051T/3) |
| Connectivity (SIM)| emnify SuperNetwork SatPlus (eUICC)   |

## Product Variants

| Variant  | Vehicle Interface | Installation    |
|----------|------------------|-----------------|
| Consumer | OBD-II port      | Self-install    |
| Prosumer | Direct wire tap  | Qualified installer |

## Architecture Decision Records

See the [ADRs](ADRs.md) section for hardware, firmware, and product architecture decisions.
