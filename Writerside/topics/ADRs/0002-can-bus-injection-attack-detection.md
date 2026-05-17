# ADR-0002: CAN Bus Injection Attack Detection and Countermeasure

| Field             | Value                                             |
|-------------------|---------------------------------------------------|
| **Status**        | Proposed                                          |
| **Date**          | 2026-03-19                                        |
| **Migrated from** | Project-wide ADR-006                              |

---

## Context

The prosumer direct-wire variant of the CEREBRAL STRATUM tracker device includes CAN bus access via the ESP32-S3's TWAI (Two-Wire Automotive Interface) controller and an Adafruit CAN Pal (TJA1051T/3) transceiver. This hardware capability creates an opportunity to deliver proactive vehicle protection services beyond passive GPS tracking.

A well-documented class of vehicle theft attack — commonly referred to as a CAN injection or "CAN Invader" attack — exploits the absence of source authentication on the ISO 11898 CAN bus. An attacker physically splices a rogue injector device into an accessible CAN bus node (most commonly the headlight wiring harness, accessible from under the front bumper), then floods the bus with forged frames impersonating a legitimate key fob. These frames target the immobiliser and engine start arbitration IDs, causing the immobiliser ECU to disarm and the engine to start without a physical key present. The attack bypasses the vehicle's cryptographic smart key system entirely, as it operates below that layer at the raw CAN frame level.

This attack has a consistent and observable forensic signature:

- A sudden, sustained high-rate flood of frames targeting known immobiliser and powertrain arbitration IDs while the vehicle ignition state is off.
- Near-simultaneous loss of expected CAN communication from the headlight ECU node, caused by the attacker physically disconnecting it to splice in their device.
- The compound condition of ECU loss plus immobiliser command flooding has a very low false-positive rate compared to either signal in isolation.

This ADR captures the detection strategy, countermeasure design, system-level event flow, and the policy decisions governing automatic versus user-triggered countermeasure activation.

---

## Decision

### Scope by Hardware Variant

| Capability                       | OBD-II Consumer Variant                                    | Direct-Wire Prosumer Variant                 |
|----------------------------------|------------------------------------------------------------|----------------------------------------------|
| CAN bus read access              | Via OBD-II port (gateway-bridged)                          | Direct bus access (body/comfort bus)         |
| Attack detection                 | Limited — gateway filtering may suppress raw bus anomalies | Full — direct visibility of body bus traffic |
| Countermeasure (error injection) | Not supported                                              | Supported (opt-in)                           |

The OBD-II consumer variant is **detection and alerting only**. Installation documentation must clearly communicate this distinction to end users.

### Detection Logic (Firmware)

Detection runs as a dedicated FreeRTOS task on the ESP32-S3, consuming frames from the TWAI hardware receive queue. Four complementary detection layers are applied:

**Layer 1 — Frame Rate Anomaly**
A rolling per-cluster frame rate counter tracks the rate of frames targeting known immobiliser and engine start arbitration ID clusters. If the sustained rate for any protected cluster exceeds a configurable threshold (default: 50 frames/second over a 500 ms window) while the vehicle ignition state is off, Layer 1 fires.

**Layer 2 — Ignition and Key State Context**
Ignition state is derived from CAN bus frames. Layer 1 alone can produce false positives during normal key-insert sequences; gating on ignition state eliminates the majority of legitimate high-rate scenarios. Detection is only active when ignition state is "off" or "accessory."

**Layer 3 — ECU Heartbeat Loss**
Each ECU on the body bus transmits periodic heartbeat or status frames at a predictable interval. The detection task maintains a watchdog per monitored node. If the headlight ECU's expected heartbeat is absent for longer than a configurable timeout (default: 3 seconds), an ECU loss flag is raised.

**Layer 4 — Compound Condition**
A CRITICAL security event is raised only when the compound condition is true: Layer 1 (frame flood) AND Layer 3 (ECU loss) are both active simultaneously. This compound gate materially reduces false positives. Layer 1 or Layer 3 individually may raise a WARNING-level event for investigation purposes but do not trigger the full alert and countermeasure flow.

Arbitration ID clusters for Toyota-specific immobiliser and powertrain frames are configured per vehicle profile, loaded from NVS at boot. Detection thresholds are tunable via the device's local web UI and remotely via the device command channel.

### Countermeasure — CAN Error Frame Injection

The CAN protocol error confinement mechanism can be exploited defensively: by transmitting a dominant bit during the data field of an attacker's injected frames, CEREBRAL STRATUM can force the attacker's device to increment its Transmit Error Counter (TEC), progressively degrading and ultimately silencing the injector.

This countermeasure is implemented as a high-priority FreeRTOS task that, when activated, monitors the TWAI receive queue for frames matching the attack signature and asserts a transmit interrupt timed to the attacker's active frame.

**Known limitations and risks:**

- Error injection is not perfectly selective; other nodes on the same bus segment may also increment their error counters.
- The countermeasure should be time-bounded. A single activation window is capped at 30 seconds by default, after which the device re-evaluates whether the attack signature is still present.
- Results are not guaranteed. An attacker device with modified error handling may not respond as expected.

### Countermeasure Activation Policy

**Default: User-triggered.** On detection of a CRITICAL compound event, the device immediately publishes a security event to Kafka and the Notification Dispatch service delivers a CRITICAL priority push notification to all registered push tokens for the device owner. The notification presents two actions: **View Details** and **Activate Countermeasure**. The countermeasure is only executed after the user explicitly selects the activation action.

**Optional: Auto-countermeasure mode.** This is an opt-in per-device setting. When enabled, the device activates error injection immediately on CRITICAL detection without waiting for user confirmation. Auto-mode still publishes the security event and push notification.

Auto-countermeasure mode is disabled by default and must be explicitly acknowledged at the point of enablement, with clear disclosure that the feature may affect legitimate vehicle CAN bus communication.

### Event Schema

Two new Kafka topics under the `device.security` namespace:

**`device.security.events`** — published by the device firmware via Eclipse Hono:

```json
{
  "eventType": "SECURITY_CAN_INJECTION_DETECTED",
  "deviceId": "string",
  "tenantId": "string",
  "timestamp": "ISO-8601",
  "severity": "CRITICAL | WARNING",
  "details": {
    "frameRateHz": "number",
    "targetArbIdClusters": ["string"],
    "ecuLossDetected": "boolean",
    "ignitionState": "OFF | ACCESSORY | ON",
    "detectionLayersFired": ["L1", "L3", "L4"]
  }
}
```

**`device.security.countermeasure`** — published by the device firmware on activation and completion:

```json
{
  "eventType": "SECURITY_CAN_COUNTERMEASURE_ACTIVATED | SECURITY_CAN_COUNTERMEASURE_RESULT",
  "deviceId": "string",
  "tenantId": "string",
  "timestamp": "ISO-8601",
  "triggeredBy": "USER | AUTO",
  "result": "IN_PROGRESS | ATTACK_SUBSIDED | TIMEOUT | INCONCLUSIVE"
}
```

The Primary Backend consumes both topics and persists events to `security_events` and `security_countermeasure_log` tables in the tenant schema.

### NTN Compatibility

The countermeasure command may need to be delivered over NTN (Skylo satellite) if the vehicle is in a location without LTE coverage. The command payload is minimal and fits within NTN payload constraints. The round-trip latency over NTN means user-triggered countermeasure in a no-LTE environment may have a delay of tens of seconds; this is acceptable and should be communicated in the UX.

---

## Consequences

### Positive

- Addresses a real, documented vehicle theft vector with a technically grounded detection and response mechanism.
- Detection is fully on-device and low-latency; the server notification path is asynchronous and not in the detection critical path.
- Layered detection (compound gate) provides a high signal-to-noise ratio and reduces notification fatigue from false positives.
- The opt-in countermeasure default protects BlueGuardian from liability exposure while giving technically sophisticated users an active protection option.
- NTN compatibility ensures the security event and command flow works even in low-connectivity environments.

### Negative / Risks

- CAN bus error injection is inherently not perfectly selective. Sustained activation may disrupt other ECUs on the same bus segment. This must be clearly disclosed in product documentation and the auto-mode opt-in flow.
- Arbitration ID clusters are vehicle-make/model specific. Maintaining a vehicle profile database introduces ongoing maintenance overhead. Initial release should scope to explicitly validated vehicle profiles (Toyota Hilux, LandCruiser, RAV4 as primary targets given known attack prevalence).
- The OBD-II variant's detection gap (gateway filtering) needs to be clearly communicated at the point of sale and during onboarding.
- The countermeasure is a best-effort mechanism. The primary value is deterrence and rapid user notification, not guaranteed prevention.

---

## Alternatives Considered

### Physical Bus Isolation via Hardware Relay

A relay wired in-line on the bus segment could physically disconnect the attacker's splice point. This would be highly effective but requires additional hardware on the PCB and disconnecting the bus segment would take down all legitimate ECUs for the duration. Rejected for v1; may be revisited for a future hardware revision.

### Frame Flooding Counter

Flooding the bus with higher-priority frames to starve the attacker's frames. Rejected — this would guarantee bus saturation and is more disruptive than error injection.

### Server-Side Detection via Telemetry Analysis

Running anomaly detection on the server using CAN telemetry streamed from the device. Rejected as the primary detection mechanism — the Kafka/cloud round-trip latency (5–30 seconds) is too slow for a real-time theft-in-progress scenario. Server-side analysis remains useful for retrospective forensics and model training.

---

## Related Decisions

- ADR-0001: Vehicle IoT Device Architecture — hardware variant scope (OBD-II vs direct-wire) governs detection and countermeasure capability.
- Backend ADR-0002: Observability and Fleet Dashboard Embedding — security events share the Notification Dispatch pipeline.
