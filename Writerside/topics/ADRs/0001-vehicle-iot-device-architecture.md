# ADR-0001: Vehicle IoT Device Architecture

> **Purpose:** Living reference document capturing hardware, firmware, and product
> architecture decisions made during the design phase.  
> **Last Updated:** 2026-03-01 (rev 4 — added data transport and messaging strategy, NTN payload scope policy)  
> **Status:** Active — update as decisions evolve or are superseded.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Core Hardware Stack](#2-core-hardware-stack)
3. [Communications Architecture](#3-communications-architecture)
4. [Data Transport & Messaging Strategy](#4-data-transport--messaging-strategy)
5. [CAN Bus Interface](#5-can-bus-interface)
6. [Power Architecture](#6-power-architecture)
7. [Relay Output](#7-relay-output)
8. [Product Variants](#8-product-variants)
9. [PCB Design Strategy](#9-pcb-design-strategy)
10. [Release & Versioning Strategy](#10-release--versioning-strategy)
11. [Market & Connectivity Coverage](#11-market--connectivity-coverage)
12. [Enclosure Architecture](#12-enclosure-architecture)
13. [Antenna Strategy](#13-antenna-strategy)
14. [Open Questions & Next Steps](#14-open-questions--next-steps)

---

## 1. Project Overview

A vehicle-mounted IoT device that:

- Reads vehicle telemetry (engine state, fuel level, GPS, etc.) from the CAN bus
- Transmits data over LTE with automatic satellite fallback when terrestrial coverage is unavailable
- Controls an isolated 12V relay for user-defined ancillary functions (e.g. fuel pump disable for asset protection)
- Operates continuously, including when the vehicle is turned off
- Supports global deployment with minimal hardware variants

---

## 2. Core Hardware Stack

### 2.1 Microcontroller — ESP32-S3-DevKitC-1

| Property          | Detail                                                       |
|-------------------|--------------------------------------------------------------|
| Chip              | Espressif ESP32-S3                                           |
| Dev Board         | ESP32-S3-DevKitC-1 (Espressif official reference board)      |
| Key features used | TWAI (CAN) controller, UART, GPIO, deep sleep, OTA, Wi-Fi/BT |
| Toolchain         | ESP-IDF (primary), Arduino-ESP32 (secondary/prototyping)     |
| Language          | Rust (via esp-rs / esp-idf-hal)                              |

**Decision rationale:** The ESP32-S3 includes a native TWAI (Two-Wire Automotive Interface) CAN 2.0B compatible hardware controller, eliminating the need for an external CAN controller IC. Deep sleep current (~8–20µA) enables battery-backed always-on operation.

### 2.2 Communications Module — Quectel BG95-S5

| Property              | Detail                                            |
|-----------------------|---------------------------------------------------|
| Module                | Quectel BG95-S5-TE-A                              |
| Part number confirmed | BG95S5LATEA-64-SGNS                               |
| Variant               | LATEA — global band SKU including Band 28         |
| Dev kit in hand       | Yes — Quectel UMTS&LTE-EVB-KIT with BG95-S5 fitted |
| Interface to ESP32    | UART (AT commands)                                |

### 2.3 CAN Bus Transceiver — Adafruit CAN Pal (TJA1051T/3)

| Property           | Detail                                                       |
|--------------------|--------------------------------------------------------------|
| Board              | Adafruit CAN Pal — ADA5708                                   |
| Transceiver IC     | NXP TJA1051T/3                                               |
| Interface to ESP32 | Direct TX/RX to ESP32-S3 TWAI peripheral                     |
| Termination        | Switchable 120Ω onboard                                      |
| Power              | 3.3V input, onboard 5V charge pump — no separate 5V required |
| Automotive rating  | Yes — ISO 11898-2:2003, rated for 12V/24V systems            |

**Decision rationale:** The TJA1051T/3 is a pure transceiver — it works directly with the ESP32-S3's native TWAI hardware over simple TX/RX pins, requiring no SPI or external CAN controller. The Adafruit CAN Pal breakout was chosen over the Seeed Studio CAN Bus board (which includes an MCP2515 SPI controller) specifically to preserve use of the ESP32-S3's native TWAI hardware.

---

## 3. Communications Architecture

### 3.1 TN + NTN Strategy

A single Quectel BG95-S5 module handles both terrestrial (LTE) and non-terrestrial (satellite) communications. There is no separate satellite modem.

```
ESP32-S3  ──UART──►  Quectel BG95-S5
                          │
                          ├── LTE Cat M1 / NB-IoT (primary)
                          │     └── via terrestrial cell towers
                          │
                          └── NTN Satellite (fallback)
                                └── via Skylo network (GEO)
```

Failover logic is handled in firmware on the ESP32-S3 via AT commands to the BG95-S5.

### 3.2 LTE Band Coverage

| Market        | Priority  | Critical Bands        | BG95-S5 Support |
|---------------|-----------|-----------------------|-----------------|
| Australia     | Primary   | B28 (700MHz APT), B3  | ✅               |
| New Zealand   | Primary   | B28, B3               | ✅               |
| USA           | Secondary | B2, B4, B12, B13, B66 | ✅               |
| EU            | Secondary | B3, B8, B20           | ✅               |
| Rest of world | Tertiary  | B1, B5, B8            | ✅               |

**Band 28 (700MHz APT) is non-negotiable** for Australian/NZ rural coverage.

### 3.3 Satellite Coverage — Skylo NTN

| Property      | Detail                                        |
|---------------|-----------------------------------------------|
| Network       | Skylo (GEO satellite NTN)                     |
| Standard      | 3GPP Rel-17 NTN                               |
| AU/NZ status  | Commercially live                             |
| US/EU status  | Available                                     |
| Integration   | Standards-based, same AT command stack as LTE |

**SIM strategy — emnify SuperNetwork SatPlus (recommended)**

| Property                     | Detail                                                                       |
|------------------------------|------------------------------------------------------------------------------|
| Provider                     | emnify                                                                       |
| Product                      | SuperNetwork SatPlus                                                         |
| TN coverage                  | Global multi-network LTE (AU, NZ, US, EU and more)                           |
| NTN coverage                 | Skylo GEO satellite — AU, NZ, US, EU live                                    |
| SIM form factor (production) | MFF2 (solderable, no physical SIM slot)                                      |
| SIM form factor (dev/proto)  | 4FF (nano SIM) or 3FF (micro SIM)                                            |
| Standard                     | M2M eUICC — SGP.01, SGP.02, SGP.016 compliant                                |
| Billing model                | Postpaid, monthly per active SIM, pooled data across fleet                   |
| Satellite MTU                | 256 bytes max per message, 20 messages/minute max throughput                 |
| Satellite billing unit       | 100-byte increments                                                          |

**MFF2 form factor rationale:** Permanently soldered to the PCB — improved ingress protection, better vibration resistance, reduced failure points.

> **Action required — BG95-S5 compatibility:** emnify SatPlus compatibility with the Quectel BG95-S5 must be confirmed directly with emnify before committing to this platform. This is a pre-production requirement.

### 3.4 GNSS

The BG95-S5 includes integrated GNSS (GPS / GLONASS / BeiDou / Galileo / QZSS). No separate GPS module is required.

---

## 4. Data Transport & Messaging Strategy

### 4.1 Dual-Path Transport Architecture

The device uses different transport protocols depending on which network path is active. MQTT is used exclusively on the LTE path. The NTN satellite path uses CoAP with binary-encoded minimal payloads.

```
ESP32-S3 firmware
    │
    ├── LTE path active?
    │       └── MQTT over TCP
    │             Topic-based routing, QoS 1
    │             Full telemetry payload (CAN data, GNSS, status)
    │             JSON or CBOR encoding
    │
    └── NTN satellite path active?
            └── CoAP over UDP
                  CBOR binary encoding
                  Minimal payload only (see 4.3)
                  Single datagram — no session handshake
                  Backend CoAP→MQTT bridge for pipeline consistency
```

### 4.2 Why Standard MQTT Is Not Suitable for NTN

| Constraint            | Value                | Impact                                          |
|-----------------------|----------------------|-------------------------------------------------|
| Max message size      | 256 bytes            | MQTT topic strings alone consume 30–50 bytes    |
| Max throughput        | 20 messages / minute | QoS 1/2 ACK round trips consume message budget  |
| GEO satellite latency | 600ms–1.5s RTT       | TCP + MQTT session handshake = 3–4s per message |
| Billing unit          | Per 100 bytes        | Every unnecessary byte has a direct cost        |

### 4.3 NTN Payload Policy — Strict Scope Limit

On the NTN satellite path, only the following data categories will be transmitted. All other data is queued locally and transmitted when LTE connectivity is restored.

| Data Category           | Transmitted via NTN? | Notes                                       |
|-------------------------|----------------------|---------------------------------------------|
| Device location (GNSS)  | ✅ Yes                | Lat/long, accuracy                          |
| Essential device status | ✅ Yes                | Online, relay state, battery %, 12V present |
| CAN bus telemetry       | ❌ No                 | Queued for LTE transmission                 |
| OBD-II / vehicle data   | ❌ No                 | Queued for LTE transmission                 |
| Enhanced diagnostics    | ❌ No                 | Queued for LTE transmission                 |
| OTA firmware updates    | ❌ No                 | LTE only — payload far exceeds NTN limits   |

### 4.4 NTN Payload Design

Target NTN payload using CBOR binary encoding:

| Field                        | Type    | Bytes      | Notes                                  |
|------------------------------|---------|------------|----------------------------------------|
| Device ID (hash)             | uint32  | 4          | Truncated hash of full UUID            |
| Timestamp (Unix epoch)       | uint32  | 4          |                                        |
| Latitude                     | float32 | 4          |                                        |
| Longitude                    | float32 | 4          |                                        |
| GNSS accuracy (m)            | uint8   | 1          | Capped at 255m                         |
| Battery level %              | uint8   | 1          |                                        |
| 12V present (ignition proxy) | bool    | 1          |                                        |
| Relay state                  | uint8   | 1          | Bitmask if multiple relays added later |
| Network path flag            | uint8   | 1          | Indicates message sent via NTN         |
| CBOR framing overhead        | —       | ~8         |                                        |
| CoAP header                  | —       | 4          | Fixed                                  |
| **Total**                    |         | **~29 bytes** | Well within 256-byte MTU            |

### 4.5 LTE Payload Design

On the LTE path, payloads are not MTU-constrained. Full telemetry is transmitted including all CAN bus data, OBD-II PIDs, GNSS, and device status. CBOR is recommended for production to reduce data costs.

MQTT topic structure (indicative):

```
devices/{device-id}/telemetry/gnss
devices/{device-id}/telemetry/can
devices/{device-id}/telemetry/status
devices/{device-id}/events/relay
devices/{device-id}/events/power
devices/{device-id}/commands/relay   ← inbound command topic
```

### 4.6 Local Queuing During NTN Operation

When the device is operating on the NTN satellite path, CAN bus and vehicle telemetry data must be queued locally on the ESP32-S3 (in NVS or external flash) for transmission once LTE is restored.

Queue design considerations:

- Define maximum queue depth to prevent unbounded storage growth
- Apply a FIFO or priority eviction policy — recent data is more valuable than old data
- Timestamps must be preserved accurately — data transmitted after LTE restoration must reflect the time it was captured, not the time it was sent
- Queue must survive deep sleep cycles and power loss (stored in NVS, not RAM)

### 4.7 Backend Protocol Bridge

```
Device (NTN → CoAP/UDP) ──► CoAP Gateway / Bridge ──┐
                                                      ├──► MQTT Broker ──► Application
Device (LTE → MQTT/TCP) ──────────────────────────────┘
```

Both AWS IoT Core and HiveMQ support CoAP-to-MQTT bridging. A message metadata field (network path flag) identifies which path was used.

### 4.8 CoAP Protocol Rationale

| Property          | CoAP                               | MQTT                              |
|-------------------|------------------------------------|-----------------------------------|
| Transport         | UDP (no handshake)                 | TCP (3-way handshake)             |
| Header overhead   | 4 bytes fixed                      | Variable, 2+ bytes + topic string |
| Reliable delivery | Confirmable message (1 round trip) | QoS 1 (multiple round trips)      |
| Session state     | Stateless                          | Stateful session                  |
| Latency tolerance | Designed for it                    | Assumes low latency               |
| ESP-IDF support   | Yes (libcoap)                      | Yes (esp-mqtt)                    |

---

## 5. CAN Bus Interface

### 5.1 Protocol Support

| Protocol                   | Scope                       | Notes                                 |
|----------------------------|-----------------------------|---------------------------------------|
| CAN 2.0B (11-bit)          | Passenger vehicles          | Standard OBD-II, ISO 15765-4          |
| CAN 2.0B (29-bit extended) | Commercial / heavy vehicles | J1939 standard                        |
| ISO 15765-4                | OBD-II layer                | Mandatory post-2008 AU/US/EU          |
| ISO 14229 (UDS)            | Modern EU vehicles          | Optional deeper diagnostics           |
| J1939                      | Trucks / commercial         | Future — firmware only, same hardware |
| OEM proprietary PIDs       | Vehicle-specific            | Best effort, user contributed         |

### 5.2 Initial Capability

**Phase 1 (current):** Read-only CAN bus listening. Target data:

- Engine on/off state
- Fuel level
- Vehicle speed
- GPS (via CAN OEM PIDs where available)
- Battery voltage

**Phase 2 (future):** Write capability to CAN bus. The TJA1051T/3 supports full bidirectional communication.

### 5.3 OBD-II Port Connection

For the consumer variant and all development work, a SparkFun OBD-II connector (DEV-09911) provides:

- CANH / CANL access (pins 6 and 14)
- 12V constant power (pin 16)
- Ground (pins 4 and 5)

---

## 6. Power Architecture

### 6.1 Requirements

| Requirement            | Detail                                                |
|------------------------|-------------------------------------------------------|
| Always-on operation    | Device must remain operational when vehicle is off    |
| Vehicle power input    | 12V (constant and/or ignition-switched)               |
| Input voltage range    | 6V min (cold crank) to 40V max (load dump)            |
| Battery backup         | Onboard cell for vehicle-off operation                |
| Target backup duration | 2–4 hours minimum, 8–12 hours preferred               |
| Thermal environment    | Up to 80°C (AU dashboard summer)                      |
| Tamper detection       | Detect sudden 12V loss, alert before battery failover |

### 6.2 Battery Chemistry — LiFePO4

**Decision:** LiFePO4 (lithium iron phosphate), not LiPo.

| Property               | LiPo                                 | LiFePO4                             |
|------------------------|--------------------------------------|-------------------------------------|
| Thermal stability      | Degrades >60°C, thermal runaway risk | Stable to 70°C+, no thermal runaway |
| Energy density         | Higher                               | Lower                               |
| Cycle life             | ~500 cycles                          | ~2000+ cycles                       |
| Automotive suitability | Poor                                 | Good                                |

LiFePO4 is mandatory for an unattended in-vehicle device in Australian conditions.

### 6.3 Power Architecture Block Diagram

```
Vehicle 12V (constant) ──────────────────────────────────────────────────────┐
                                                                              │
                         ┌─────────────────────────────────────────────┐     │
                         │  Automotive Buck Converter                  │     │
                         │  (e.g. LM5165 / LM53635 — TI)              │     │
                         │  Input: 6–40V    Output: 5V regulated       │     │
                         └───────────────────┬─────────────────────────┘     │
                                             │                                │
                         ┌───────────────────▼─────────────────────────┐     │
                         │  Charger IC with Power Path                 │◄────┘
                         │  (e.g. BQ25895 / BQ25798 — TI)             │
                         │  Manages: charging, power path, protection  │
                         └───────┬───────────────────┬─────────────────┘
                                 │                   │
                    ┌────────────▼───┐     ┌─────────▼──────────────────┐
                    │  LiFePO4 Cell  │     │  System Power Rail (3.3V)  │
                    │  (backup)      │     │  ESP32-S3, BG95-S5, etc.   │
                    └────────────────┘     └────────────────────────────┘
```

### 6.4 Key ICs Under Consideration

| Function                | IC Options                 | Interface | Notes                                                                      |
|-------------------------|----------------------------|-----------|----------------------------------------------------------------------------|
| Charger / power path    | BQ25895, BQ25798 (TI)      | I2C       | Power path critical — system runs from input, not battery, when vehicle on |
| Buck converter (12V→5V) | LM5165, LM53635 (TI)       | —         | Automotive grade, handles 6–40V input                                      |
| Fuel gauge              | BQ27441, BQ27427, MAX17048 | I2C       | State of charge reporting to ESP32                                         |

### 6.5 Low Power Strategy

| Mode             | ESP32-S3 Current | BG95-S5 Current  | Trigger                             |
|------------------|------------------|------------------|-------------------------------------|
| Active           | ~240mA           | ~300mA (TX peak) | Vehicle on / active comms           |
| Modem sleep      | ~20mA            | ~1mA             | Short idle periods                  |
| Deep sleep + PSM | ~20µA            | ~15µA (3GPP PSM) | Vehicle off, between wake intervals |

3GPP Power Saving Mode (PSM) must be configured on the BG95-S5 for extended battery operation.

### 6.6 Relay State on Power Loss

**Decision:** On complete power loss, relay must default to **last known state** stored in ESP32-S3 NVS (non-volatile storage), not a fixed open/closed state.

**Rationale:** For the fuel pump disable use case, a power interruption (including deliberate battery disconnection by a thief) must not automatically re-enable the fuel pump.

---

## 7. Relay Output

### 7.1 Specification

| Property                    | Detail                                                   |
|-----------------------------|----------------------------------------------------------|
| Output voltage              | 12V                                                      |
| Purpose                     | User-defined ancillary device control                    |
| Primary use case            | Fuel pump disable for asset protection                   |
| Isolation                   | Fully electrically isolated from ESP32-S3 (optoisolated) |
| Control                     | Single GPIO from ESP32-S3 via optoisolator driver        |
| Default state on power loss | Last known state (see 6.6)                               |

### 7.2 Isolation Rationale

The vehicle electrical system is a hostile environment — load dump spikes, inductive kickback from motors, and ground offsets are common. Optoisolation protects the ESP32-S3 from 12V-side events. This is mandatory.

---

## 8. Product Variants

### 8.1 Overview

| Property              | Consumer                              | Prosumer                                    |
|-----------------------|---------------------------------------|---------------------------------------------|
| Vehicle connection    | OBD-II port (plug-in)                 | Direct wire tap to CAN bus                  |
| Power source          | OBD-II pin 16 (12V) + onboard LiFePO4 | Hardwired to vehicle 12V + onboard LiFePO4  |
| Installation          | Self-install, no tools                | Installer / technically capable user        |
| Enclosure             | Compact, fits under dash              | Weatherproof, flexible mounting             |
| Battery capacity      | ~1000–2000mAh (size constrained)      | ~5000–10000mAh (unconstrained)              |
| Target market         | Residential / consumer                | Commercial / fleet / asset protection       |

### 8.2 Shared Core

Both variants share:

- Identical core PCB (ESP32-S3, BG95-S5, power management, CAN transceiver, fuel gauge)
- Identical firmware codebase
- Same OTA update infrastructure
- Variant detected at boot via hardware ID pin or carrier board register

### 8.3 OBD-II Passthrough (Consumer Variant)

The consumer variant should implement an **OBD-II passthrough** — a female OBD-II port that exposes all pins from the vehicle port, so the device can remain plugged in while a workshop tool or dealer scan tool is also connected.

---

## 9. PCB Design Strategy

### 9.1 Phased Approach

1. **Phase 1 — Breadboard prototype:** Validate all subsystems using dev boards and jumper wires before committing to PCB layout.
2. **Phase 2 — First PCB (single board):** Design a single 4-layer PCB containing all subsystems.
3. **Phase 3 — Modular split:** Once Phase 2 is proven, split into core module + variant-specific carrier boards.

### 9.2 Layer Stack-up (Minimum)

4-layer board is the minimum for this design:

| Layer | Purpose                                        |
|-------|------------------------------------------------|
| L1    | Components and signal routing                  |
| L2    | Solid ground plane (critical for RF and noise) |
| L3    | Power planes                                   |
| L4    | Signal routing                                 |

### 9.3 Subsystem Separation Guidelines

| Subsystem                    | Sensitivity                          | Placement Guidance                                                            |
|------------------------------|--------------------------------------|-------------------------------------------------------------------------------|
| BG95-S5 RF module            | High — both emits and is susceptible | Edge of board, antenna connector facing outward, away from switching supplies |
| Power input / buck converter | Noisy — switching transients         | Opposite edge from RF, good thermal copper pour                               |
| Relay and 12V IO             | Very noisy — inductive switching     | Isolated ground zone, TVS on all 12V lines                                    |
| ESP32-S3                     | Moderate                             | Central, between RF and power zones                                           |
| CAN transceiver              | Moderate                             | Near OBD-II / wiring connector                                                |

### 9.4 Tooling

| Tool    | Recommendation              |
|---------|-----------------------------|
| KiCad   | **Primary recommendation** — free, open source, professional grade |
| EasyEDA | Secondary / learning — browser-based, integrated with JLCPCB/LCSC  |

### 9.5 Hardware Versioning

From the first prototype, implement:

- A visible PCB version silkscreen (e.g. `HW_REV: 1.0`)
- A hardware version register readable by firmware at boot (GPIO strap pins or I2C EEPROM)
- A changelog tracking hardware revisions and firmware compatibility

---

## 10. Release & Versioning Strategy

### 10.1 Customer Tiers

| Tier        | Description                                                 | Firmware Track  |
|-------------|-------------------------------------------------------------|-----------------|
| Residential | Early adopters, bleeding edge, higher tolerance for updates | Latest stable   |
| Commercial  | Fleet / asset protection, maximum stability                 | n-1, LTS-style  |

### 10.2 Firmware Release Pipeline

```
Development
    │
    ▼
Residential (latest stable — auto-pushed OTA)
    │
    │  ← stabilisation window / defined soak period
    ▼
Commercial (n-1, change-controlled, on-demand or scheduled OTA)
```

### 10.3 OTA Update Requirements

OTA firmware updates are **mandatory from day one**, not a future feature.

- ESP32-S3 OTA via ESP-IDF dual partition (A/B) with automatic rollback on failure
- Update delivery via BG95-S5 LTE/satellite data channel
- Tier-based update scheduling
- Failed update must roll back to previous known-good firmware automatically
- OTA must function on both LTE and satellite (satellite bandwidth constraints mean update packages should be delta/incremental where possible)

---

## 11. Market & Connectivity Coverage

### 11.1 Target Markets

| Market        | Priority  | LTE             | Satellite         | Notes                                |
|---------------|-----------|-----------------|-------------------|--------------------------------------|
| Australia     | Primary   | ✅ B28 + B3      | ✅ Skylo live      | Band 28 mandatory for rural coverage |
| New Zealand   | Primary   | ✅ B28 + B3      | ✅ Skylo live      | —                                    |
| USA           | Secondary | ✅ B2/4/12/13/66 | ✅ Skylo available | —                                    |
| EU            | Secondary | ✅ B3/8/20       | ✅ Skylo available | —                                    |
| Rest of world | Tertiary  | ✅ B1/5/8        | TBC               | Skylo expanding                      |

### 11.2 Hardware SKU Strategy

**Goal: One hardware SKU for all markets.**

The BG95-S5 LATEA variant covers all target LTE bands globally. No regional hardware variants are required for the communications stack. Variant differentiation is Consumer vs Prosumer only, not regional.

---

## 12. Enclosure Architecture

### 12.1 Modular Core + Carrier Concept

The device is physically split into two parts:

- **Core module** — a self-contained enclosure housing all active electronics (ESP32-S3, BG95-S5, power management, CAN transceiver, fuel gauge). This is the certified radio device.
- **Carrier enclosure** — a variant-specific outer enclosure. Contains only passive connectors, wiring, antenna elements, and mechanical housing.

```
┌─────────────────────────────────────────────┐
│              Carrier Enclosure              │
│   (variant-specific — Consumer or           │
│    Prosumer/Commercial)                     │
│                                             │
│   ┌─────────────────────────────────────┐   │
│   │          Core Module                │   │
│   │  ESP32-S3 │ BG95-S5 │ Power mgmt   │   │
│   │  CAN transceiver │ Fuel gauge       │   │
│   └─────────────────────────────────────┘   │
│                                             │
│   [ Variant-specific connectors,            │
│     antenna, power input, relay IO ]        │
└─────────────────────────────────────────────┘
```

### 12.2 Certification Advantage

Separating the radio (BG95-S5) into the core module means:

- Only the core module requires radio certification (RCM / ACMA in AU/NZ, FCC in US, CE in EU)
- The BG95-S5 itself carries Quectel's own modular certifications
- Carrier enclosures contain no active radio components and do not independently require radio certification
- Hardware revisions to the carrier do not trigger re-certification of the radio

> **Action required:** Engage an Australian RCM / EMC consultant before finalising the core module PCB design.

### 12.3 Enclosure Manufacturing Path

| Phase                 | Approach                                                  |
|-----------------------|-----------------------------------------------------------|
| Prototyping           | Modified off-the-shelf ABS enclosures (Hammond / Bopla)   |
| Pilot (10–100 units)  | SLS nylon 3D printing                                     |
| Production            | Injection moulding (economical above ~500–1000 units)     |

---

## 13. Antenna Strategy

### 13.1 Overview

Antenna type is determined by the **carrier enclosure variant**, not the core module. The core module exposes U.FL / MHF4 coaxial ports for LTE and GNSS.

| Property              | Consumer Carrier             | Prosumer / Commercial Carrier                               |
|-----------------------|------------------------------|-------------------------------------------------------------|
| Antenna type          | Internal (inside enclosure)  | External via bulkhead connector                             |
| Connector on carrier  | U.FL / MHF4 internal pigtail | SMA (development/prosumer) or FAKRA (commercial production) |

### 13.2 Consumer Variant — Internal Antenna

**Recommended:** Combined LTE + GNSS FPC (Flexible Printed Circuit) antenna, mounted adhesively on inside of enclosure lid.

| Property            | Detail                                                   |
|---------------------|----------------------------------------------------------|
| Type                | FPC / flex adhesive antenna                              |
| Coverage            | Combined LTE (Cat M1 / NB-IoT) + GNSS in a single unit  |
| Suggested suppliers | Taoglas, Molex, Abracon                                  |
| Example part        | Taoglas FXP73 (combined LTE+GNSS FPC) or equivalent      |

### 13.3 Prosumer / Commercial Variant — External Antenna

**Connector standard: FAKRA for production, SMA for development/early prosumer**

**FAKRA colour coding:**

| Band           | FAKRA Code | Colour          |
|----------------|------------|-----------------|
| GNSS / GPS     | FAKRA Z    | Black           |
| Cellular (LTE) | FAKRA D    | Grey or FAKRA C |

---

## 14. Open Questions & Next Steps

### 14.1 Immediate (Prototyping Phase)

- [ ] Procure OBD-II connector (SparkFun DEV-09911 from Core Electronics AU)
- [ ] Procure Adafruit CAN Pal transceiver (ADA5708 from Core Electronics AU)
- [ ] Wire ESP32-S3 DevKitC-1 ↔ BG95-S5 EVB via UART and test AT commands
- [ ] Wire ESP32-S3 ↔ CAN Pal ↔ OBD-II connector and validate TWAI receive

### 14.2 Near Term (Firmware)

- [ ] Implement basic TWAI receive and log raw CAN frames
- [ ] Identify target vehicle PIDs (fuel level, ignition state, speed)
- [ ] Implement BG95-S5 AT command driver in Rust (esp-idf-hal / esp-rs)
- [ ] Implement LTE → satellite failover logic with path detection
- [ ] Implement MQTT client for LTE path (esp-mqtt)
- [ ] Implement CoAP client for NTN satellite path (libcoap via esp-idf)
- [ ] Implement CBOR binary encoding for NTN payloads
- [ ] Implement local telemetry queue (NVS-backed) for CAN data during NTN operation
- [ ] Implement 3GPP PSM configuration on BG95-S5
- [ ] Implement ESP32-S3 deep sleep with timed wake
- [ ] Implement OTA update partition scheme
- [ ] Design and implement backend CoAP→MQTT bridge (evaluate AWS IoT Core or HiveMQ)

### 14.3 Commercial / Procurement

- [ ] **Confirm BG95-S5 compatibility with emnify SatPlus** — hard dependency before finalising PCB design
- [ ] Evaluate emnify SatPlus plan tiers against expected per-device data consumption
- [ ] Design emnify REST API integration for per-SIM chargeback
- [ ] Confirm BG95-S5 production availability and pricing at target volumes
- [ ] Investigate Australian RCM requirements for a vehicle-connected radio device

### 14.4 Hardware Design (PCB Phase)

- [ ] Complete breadboard prototype validation before starting PCB design
- [ ] Select and evaluate BQ25895 or BQ25798 charger IC
- [ ] Select and evaluate automotive buck converter (LM5165 / LM53635)
- [ ] Select LiFePO4 cell size for each variant
- [ ] Design 4-layer PCB in KiCad (single board first)
- [ ] Finalise core PCB dimensions before enclosure design begins

### 14.5 Enclosure & Antenna

- [ ] Engage RCM / EMC consultant to confirm certification strategy for core module
- [ ] Select combined LTE+GNSS FPC antenna for consumer carrier (evaluate Taoglas FXP73 or equivalent)
- [ ] Define core-to-carrier interface connector (evaluate Molex Micro-Fit 3.0)
- [ ] Define IP rating target for prosumer/commercial carrier enclosure (IP65 minimum)

---

## Appendix A — Component Reference

| Component                     | Part                                      | Supplier             | Approx. Cost (AUD)       |
|-------------------------------|-------------------------------------------|----------------------|--------------------------|
| MCU Dev Board                 | ESP32-S3-DevKitC-1                        | Various              | ~$15–20                  |
| Comms Module                  | Quectel BG95-S5 (BG95S5LATEA-64-SGNS)    | Quectel / Mouser / DigiKey | TBC               |
| Comms Dev Kit                 | Quectel UMTS&LTE-EVB-KIT                  | —                    | In hand                  |
| CAN Transceiver               | Adafruit CAN Pal ADA5708 (TJA1051T/3)    | Core Electronics AU  | $7.15                    |
| OBD-II Connector              | SparkFun DEV-09911                        | Core Electronics AU  | $4.20                    |
| Jumper Wires + Breadboard Kit | Jaycar PB8819                             | Jaycar (in store)    | $29.95                   |
| eSIM (development)            | emnify SatPlus 4FF / 3FF nano or micro SIM | emnify.com          | Per plan                 |
| eSIM (production)             | emnify SatPlus MFF2 solderable            | emnify.com           | Per plan                 |

## Appendix B — Key References

- [Espressif TWAI documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/twai.html)
- [Quectel BG95-S5 product page](https://www.quectel.com/product/lpwa-bg95-s5)
- [CoAP RFC 7252](https://datatracker.ietf.org/doc/html/rfc7252)
- [CBOR RFC 8949](https://datatracker.ietf.org/doc/html/rfc8949)
- [emnify SuperNetwork SatPlus](https://www.emnify.com/iot-supernetwork/satellite)
- [Skylo NTN](https://skylo.tech)
- [TJA1051T/3 datasheet](https://www.nxp.com/products/interfaces/can-transceivers/can-with-flexible-data-rate/high-speed-can-transceiver:TJA1051)
- [KiCad EDA](https://www.kicad.org)
