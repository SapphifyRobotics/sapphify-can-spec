# SAPPHIFY CAN protocol specification

**Status: DRAFT v0.9 — not frozen.**
**Manufacturer ID pending assignment by FIRST. Beta builds use the "Team Use" manufacturer ID 8.**

Document owner: SAPPHIFY LLC. Licence: CC BY 4.0. Implementations: MIT.

This document is normative for **every SAPPHIFY FRC CAN device**, not for one product.
Firmware, the WPILib vendor library, the host tooling and the published test vectors all derive
from it. Version 1.0 is not cut until the manufacturer ID is assigned and the shipped firmware
matches this document exactly.

---

## 0. The device family

SAPPHIFY FRC devices share one manufacturer ID, one protocol, one vendor library
(`SapphifyLib`), one configuration tool and one log format. A team that learns one of them has
learned all of them.

| Device | FRC device type | Status | Hardware |
|---|---|---|---|
| **ROTEM** — CAN FD attitude and heading reference | 4 (Gyro Sensor) | routed rev B, first article | ICM-45686 + MMC5983MA + STM32U575 |
| absolute encoder (unnamed) | 7 (Encoder) | in design | MT6826S |
| USB-CAN FD bridge (unnamed) | 10 (Miscellaneous) | in design | — |

**This is the reason the series matters technically, not just commercially:**

- **One manufacturer ID** is requested from FIRST once and covers the whole family. Device Type
  and Device Number separate the products inside it. Requesting an ID per product would be wrong
  and would be refused.
- **One vendordep.** `SapphifyLib` exposes every device. A team adds one library entry and gets
  the encoder for free when it buys one. This is what Phoenix and ReduxLib do, and a
  library-per-product would be a strategic error we cannot undo after publication.
- **One configuration tool** discovers, identifies, calibrates and updates any SAPPHIFY device on
  the bus or over USB.
- **One shared frame vocabulary.** Health flags, identity, configuration persistence, time
  synchronisation and firmware update are defined once in sections 2 to 3.4 and are identical on
  every device. Only the measurement frames differ per product.

Sections 1 through 3.4 are the series contract. Product-specific frames live in section 5,
which currently defines the AHRS only.

### 0.1 Naming

**The vendor name carries the library; each product keeps a distinct name of its own.** The
library is `SapphifyLib`; the AHRS is `ROTEM`. This follows REV (`REVLib` + SPARK MAX) and CTRE
(Phoenix + Pigeon 2) rather than a shared product prefix, for a practical reason: a library named
after one product reads as broken the moment a differently-named second product ships inside it,
and renaming a vendordep after teams have installed it forces every one of them to uninstall and
reinstall.

Names built on "compass" or "north" were rejected on substance rather than taste. They would
advertise magnetic heading as the primary output, and magnetometer fusion is **off by default** in
the FRC profile. A name promising what the default configuration deliberately refuses to do is a
support cost forever.

---

## 1. Scope and design rules

- Any team may implement a driver from this document alone, with no SAPPHIFY software.
- Every field below is fully specified: width, byte order, scaling, unit and invalid encoding.
- Values are **little-endian**, matching the STM32 and the WPILib host. Bit 0 is the LSB.
- Signed values are two's complement.
- Every frame that carries an estimator output also carries a timestamp and a sequence number.
  A frame without a fresh timestamp is not usable for pose estimation and must be treated as stale.
- Frames are self-describing enough to be replayed offline. Nothing requires vendor tooling.

### 1.1 Rule: the manufacturer ID is a named constant

The manufacturer ID is embedded in every arbitration ID on the bus. It is defined **once**, as
`SAPPHIFY_CAN_MANUFACTURER_ID`, and referenced by firmware, library, tooling, this document and the
test vectors. It never appears as a literal. When FIRST assigns the production value, one constant
changes and the whole stack follows.

Until assignment:

```
SAPPHIFY_CAN_MANUFACTURER_ID = 8    // "Team Use" — BETA ONLY, must not ship in a release build
```

---

## 2. Addressing

Every SAPPHIFY device uses the standard FRC CAN arbitration-ID layout (29-bit extended identifier).

| Field | Width | ROTEM value |
|---|---|---|
| Device Type | 5 bits | per product — see the series table in section 0 |
| Manufacturer | 8 bits | `SAPPHIFY_CAN_MANUFACTURER_ID` (8 during beta; 21–255 once assigned) — **the same value for every device** |
| API Class | 6 bits | see section 3 |
| API Index | 4 bits | see section 3 |
| Device Number | 6 bits | 0–62; **default 0**; 63 reserved |

Device Type values are taken from the FIRST device-type table. The AHRS uses 4, "Gyro Sensor": it
is an AHRS rather than a bare gyro, but 4 is the closest assigned type and the reserved range
14–30 is not ours to claim. This choice is raised with FIRST alongside the manufacturer-ID
request.

Device Number is scoped per Device Type, so an AHRS and an encoder may both be device 0 without
conflicting. `FLAG_ID_CONFLICT` is therefore evaluated within a Device Type, never across the
family.

### 2.1 Device number and conflicts

- Factory default device number is 0.
- The device number is stored in the configuration flash and survives power cycles.
- On boot and continuously thereafter, ROTEM listens for any frame carrying its own Device
  Type + Manufacturer + Device Number that it did not transmit. On detection it sets
  `FLAG_ID_CONFLICT`, reports it through the health frame and the WPILib Alert API, and drives the
  CAN LED to the fault pattern. **It does not silently renumber itself.**
- Device number is assignable over USB with no bus and no robot code, and over CAN by directed
  frame.

### 2.2 CAN FD and bus compatibility

- ROTEM operates on **classic CAN 2.0B at 1 Mbps** and on **CAN FD**. Both are supported from the
  first firmware release; the mode is auto-detected and also settable explicitly.
- On CAN FD, ROTEM packs a full estimator state into a single large frame rather than several
  8-byte frames. This is a deliberate bandwidth choice: the Systemcore CAN interfaces share SPI
  controllers and the practical limit is **frames per second**, not bits per second.
- The **broadcast/robot-state heartbeat from the robot controller is always a CAN 2.0 frame,
  never an FD frame**, and is forwarded across Motioncore to all buses. Firmware must accept it in
  classic form regardless of the configured FD mode.
- Motioncore is transparent to addressing. No ROTEM behaviour changes when it is present.

### 2.3 Broadcast and enable state

ROTEM honours the standard broadcast messages, including disable. Behaviour:

- **Disabled**: ROTEM keeps estimating and keeps its bias/ZUPT policy running. It does **not**
  stop, freeze or drift-park the estimator. It continues publishing on the bus at the configured
  rates. Losing heading while the robot sits disabled is a documented failure of another product;
  it is not one we reproduce.
- **Bus-off**: recover per the standard automatic recovery sequence, increment
  `busOffCount`, and never lose stored configuration.
- **No heartbeat for the configured timeout**: set `FLAG_NO_HOST`, drive the CAN LED to the
  "host software absent" pattern, keep estimating, keep logging.

---

## 3. API map

API Class 0–15 are periodic status frames, 16–31 are on-demand control, 32–47 are configuration,
48–63 are reserved for the bulk/black-box and firmware paths.

API classes 2 (`STATUS_HEALTH`, `STATUS_CALIBRATION`, `STATUS_CAN`, `STATUS_IDENTITY`), 16
(commands) and 32 (configuration) are **series-common**: identical fields, identical semantics on
every ROTEM device. Classes 0, 1 and 3 are the measurement frames and are product-specific; the
tables below define them for the ROTEM AHRS.

### 3.1 Periodic status frames — ROTEM

| Class | Index | Name | Default rate | Payload |
|---|---|---|---|---|
| 0 | 0 | `STATUS_ORIENTATION` | 100 Hz | quaternion + timestamp + sequence |
| 0 | 1 | `STATUS_RATES` | 100 Hz | calibrated angular rate, 3 axes |
| 0 | 2 | `STATUS_ACCEL` | 100 Hz | calibrated acceleration, 3 axes |
| 0 | 3 | `STATUS_EULER` | 50 Hz | roll/pitch/yaw convenience frame |
| 1 | 0 | `STATUS_QUALITY` | 10 Hz | yaw 1σ uncertainty, accumulated drift estimate, bias magnitude |
| 1 | 1 | `STATUS_BIAS` | 4 Hz | gyro bias estimate, 3 axes, plus die temperature |
| 1 | 2 | `STATUS_VIBRATION` | 4 Hz | per-axis vibration level, clip counters, dominant frequency |
| 1 | 3 | `STATUS_MAG` | 4 Hz | field magnitude, disturbance flag, confidence, drift-audit heading error |
| 2 | 0 | `STATUS_HEALTH` | 4 Hz | flag word, estimator state, self-test result, fault code |
| 2 | 1 | `STATUS_CALIBRATION` | 1 Hz | calibration validity, age, mount-pose state, calibration revision |
| 2 | 2 | `STATUS_CAN` | 1 Hz | frames sent/dropped, bus-off count, rx errors, utilisation contribution |
| 2 | 3 | `STATUS_IDENTITY` | 1 Hz | serial number, hardware revision, firmware version, protocol version |
| 3 | 0 | `STATUS_FD_COMPOSITE` | 250 Hz (FD only) | orientation + rates + accel + quality + flags in one FD frame |

Every rate in this table is individually configurable, including to zero. `setUpdateFrequency()`
in the vendor library maps directly onto these.

**Rate policy.** Defaults are chosen to be safe on a shared 1 Mbps bus with a full robot's worth
of devices. Teams running pose estimation at 250–500 Hz raise `STATUS_ORIENTATION`, or switch to
`STATUS_FD_COMPOSITE` on an FD bus, and the library reports the resulting bus utilisation.

### 3.2 Frame layouts

#### `STATUS_ORIENTATION` — class 0, index 0

| Offset | Width | Field | Encoding |
|---|---|---|---|
| 0 | int16 | `qw` | value / 32767, unit quaternion component |
| 2 | int16 | `qx` | value / 32767 |
| 4 | int16 | `qy` | value / 32767 |
| 6 | int16 | `qz` | value / 32767 |

The 8-byte classic frame carries the quaternion only. The device timestamp and sequence number
travel in the paired FD composite frame, or — on classic CAN — in `STATUS_EULER`, which reserves
its upper bytes for them. A separate high-rate timestamped sample stream is available over the FD
composite frame and over USB; this is the mechanism the library uses for latency compensation.

> Draft note: the classic-CAN timestamp carriage is the one open design point in this document.
> Two candidates are on the table — a shared 16-bit rolling device timestamp appended to each
> status frame at the cost of quaternion resolution, or a separate `STATUS_TIME` frame disciplined
> against the robot heartbeat. Resolve this against measured latency on real hardware before v1.0.
> Do not implement library latency compensation until it is resolved.

#### `STATUS_RATES` / `STATUS_ACCEL` — class 0, index 1 / 2

| Offset | Width | Field | Encoding |
|---|---|---|---|
| 0 | int16 | x | rates: value × 0.02 °/s. accel: value × 0.002 g |
| 2 | int16 | y | as above |
| 4 | int16 | z | as above |
| 6 | uint16 | `seq` | free-running sample sequence, wraps at 65535 |

Full-scale at these codings is ±655 °/s and ±65 g, both beyond the configured sensor ranges, so
the encoding never clips before the sensor does. The saturation flags in `STATUS_HEALTH` report
sensor clipping, which is the event that actually matters.

#### `STATUS_QUALITY` — class 1, index 0

| Offset | Width | Field | Encoding |
|---|---|---|---|
| 0 | uint16 | `yawSigma` | 1σ yaw uncertainty from the filter covariance, value × 0.001 ° |
| 2 | uint16 | `driftSinceZero` | estimated accumulated yaw error since last zero, value × 0.001 ° |
| 4 | uint16 | `biasMagnitude` | ‖gyro bias‖, value × 0.0001 °/s |
| 6 | uint8 | `zuptState` | 0 moving, 1 stationary candidate, 2 ZUPT applied, 3 inhibited |
| 7 | uint8 | `estimatorState` | 0 init, 1 converging, 2 converged, 3 degraded, 4 fault |

This frame is the product. No other FRC IMU publishes its own uncertainty, and it is what turns
"our odometry is broken" into a readable diagnosis.

#### `STATUS_HEALTH` — class 2, index 0

| Offset | Width | Field |
|---|---|---|
| 0 | uint32 | `flags` — bit field below |
| 4 | uint8 | `selfTestResult` — 0 pass, non-zero = failed subtest index |
| 5 | uint8 | `faultCode` |
| 6 | uint8 | `dieTemperatureC` — signed, °C |
| 7 | uint8 | `uptimeMinutes` — saturating |

Flag bits:

| Bit | Name | Meaning |
|---|---|---|
| 0 | `FLAG_CAL_VALID` | factory calibration present and CRC-valid |
| 1 | `FLAG_CAL_STALE` | calibration older than the configured age limit |
| 2 | `FLAG_MOUNT_SET` | mount pose configured and persisted |
| 3 | `FLAG_MAG_ENABLED` | magnetometer fusion is on (off by default) |
| 4 | `FLAG_MAG_DISTURBED` | magnetic disturbance detected, corrections suspended |
| 5 | `FLAG_GYRO_SAT` | gyro saturation since last report |
| 6 | `FLAG_ACCEL_SAT` | accelerometer saturation since last report |
| 7 | `FLAG_HIGH_VIBRATION` | vibration above the configured threshold |
| 8 | `FLAG_FIFO_OVERRUN` | sensor FIFO overrun |
| 9 | `FLAG_TIME_DISCONTINUITY` | rejected or implausible sample timestamp |
| 10 | `FLAG_NO_HOST` | no robot-controller heartbeat within timeout |
| 11 | `FLAG_ID_CONFLICT` | another device answers on our address |
| 12 | `FLAG_BUS_OFF_RECOVERED` | bus-off occurred and recovery completed |
| 13 | `FLAG_LOG_ACTIVE` | black-box recorder is writing |
| 14 | `FLAG_LOG_FULL` | black-box flash wrapped |
| 15 | `FLAG_TEMP_OUT_OF_CAL` | die temperature outside the calibrated range |
| 16 | `FLAG_NUMERICAL_FAULT` | estimator numerical fault, output not trustworthy |
| 17 | `FLAG_FIRMWARE_MISMATCH` | firmware/protocol version mismatch with the host library |
| 18–31 | reserved | must be transmitted as zero |

Every flag above maps to exactly one WPILib Alert string in the vendor library. That mapping is
part of the library's public test suite.

### 3.3 Control and configuration

| Class | Index | Name | Direction |
|---|---|---|---|
| 16 | 0 | `CMD_ZERO_YAW` | to device |
| 16 | 1 | `CMD_SET_YAW` | to device, int32 milli-degrees |
| 16 | 2 | `CMD_RESET_ESTIMATOR` | to device |
| 16 | 3 | `CMD_IDENTIFY` | to device, blinks the LEDs for N seconds |
| 16 | 4 | `CMD_SELF_TEST` | to device |
| 32 | 0 | `CFG_SET_DEVICE_NUMBER` | to device, persisted |
| 32 | 1 | `CFG_SET_RATE` | to device, per API class/index |
| 32 | 2 | `CFG_SET_MOUNT_POSE` | to device, persisted |
| 32 | 3 | `CFG_SET_MAG_ENABLE` | to device, persisted |
| 32 | 4 | `CFG_COMMIT` | to device, atomic commit of the staged set |
| 32 | 5 | `CFG_READ` | to device, replies with the current value |

**Persistence contract.** Configuration is staged, range-checked, then committed atomically to
wear-levelled flash with a schema version, a monotonic revision counter and a CRC. A brown-out
during commit leaves the previous valid configuration intact. Configuration that does not survive
a power cycle is a defect, not a limitation.

### 3.4 Time synchronisation

ROTEM timestamps every sample on-device from a monotonic hardware timer and publishes the
timestamp with the data. It additionally accepts and emits a bus-time discipline message so that
several ROTEM units — and future SAPPHIFY devices — share one time base.

This works on a plain 1 Mbps bus. It requires no companion hardware and no licence. The achievable
synchronisation precision will be **measured and published**, not asserted; the specification
reserves the frame and defines the fields, and the number goes in when the bench data exists.

---

## 4. What is deliberately not in this document yet

Honest gaps, to be closed before v1.0:

1. Production manufacturer ID (blocked on FIRST).
2. Classic-CAN timestamp carriage — see the draft note in 3.2.
3. Measured time-synchronisation precision.
4. Black-box download and firmware-update frame formats over CAN (class 48–63). The USB path is
   the primary one for both; the CAN path is defined once the USB protocol is stable.
5. Test vectors: a machine-readable file of encoded frames and their decoded values, published
   with the specification so any implementer can verify a decoder without hardware.

## 5. Change log

- **v0.9, 2026-08-18** — first public draft. Written as a **vendor-wide** specification: one
  manufacturer ID, one common frame vocabulary, product-specific measurement frames. ROTEM is the
  first device defined under it.
  Addressing, AHRS status frame map, health flags, configuration and persistence contract
  defined. Manufacturer ID pending.
