# Software Requirements Specification
## MAX30101/MAX30102 Sensor Driver — Zephyr RTOS

| Field | Value |
|---|---|
| Document ID | SRS-MAX30101-001 |
| Version | 1.0 |
| Author | Md Shofiqul Islam |
| Date | 2026-06-10 |
| Status | Released |
| Standard | IEC 62304:2006/Amd 1:2015 Clause 5.2 |
| Device context | Wearable heart rate / SpO2 monitor (Class II medical device) |

---

## 1. Purpose and Scope

This document specifies the software requirements for the MAX30101/MAX30102
photoplethysmography (PPG) sensor driver implemented for the Zephyr RTOS platform.

These requirements govern the driver software unit responsible for:
- Initialising and configuring the MAX30101/MAX30102 sensor hardware
- Acquiring raw PPG optical data from the sensor
- Delivering data to the application layer via the Zephyr Sensor API
- Detecting and reporting hardware faults

This document is the authoritative requirements source for the Software Design
Specification (SDS-MAX30101-001) and the Failure Mode and Effects Analysis
(FMEA-MAX30101-001).

**Out of scope:** Heart rate calculation algorithms; SpO2 computation;
clinical interpretation of sensor data; display or storage of results.

---

## 2. Definitions

| Term | Definition |
|---|---|
| PPG | Photoplethysmography — optical technique measuring blood volume changes |
| SpO2 | Peripheral oxygen saturation via pulse oximetry |
| FIFO | First-In-First-Out hardware sample buffer inside the MAX30101 |
| ISR | Interrupt Service Routine — executes in interrupt context, not thread context |
| Shall | Indicates a mandatory requirement |
| Should | Indicates a recommended but non-mandatory behaviour |
| Application layer | Software above the driver that consumes PPG data |

---

## 3. System Context

The driver operates as a software unit within a Class II medical device. The
device acquires PPG waveforms to calculate heart rate and peripheral oxygen
saturation. The driver sits between the MAX30101 hardware and the application
algorithm layer.

```
[Application: HR / SpO2 Algorithms]
           ↕  Zephyr Sensor API
  [MAX30101 Driver — this unit]
           ↕  I2C bus + GPIO INT
    [MAX30101 Hardware Sensor]
```

The driver does not perform any clinical calculation. It provides raw ADC values
only. Clinical safety obligations are owned by the application layer.

---

## 4. Functional Requirements

### 4.1 Initialisation

**REQ-003** — The driver shall verify sensor identity by reading the PART_ID
register and confirming it matches the expected value (0x15 for MAX30101) before
completing initialisation.

**REQ-003.1** — If the PART_ID does not match, the driver shall return an error
code and leave the device in an uninitialised state. The driver shall not attempt
data acquisition from an unrecognised device.

**REQ-003.2** — The driver shall reset the sensor to a known state via the
MODE_CONFIG reset bit at the start of every initialisation sequence.

**REQ-003.3** — The driver shall configure the sensor with the following default
operating parameters at initialisation unless overridden via `sensor_attr_set()`:

| Parameter | Default Value |
|---|---|
| Operating mode | SpO2 (RED + IR channels active) |
| Sample rate | 100 samples per second |
| LED pulse width | 411 µs |
| ADC range | 16384 nA full scale |
| RED LED current | 6.4 mA |
| IR LED current | 6.4 mA |
| FIFO almost-full threshold | 17 samples |
| FIFO rollover | Enabled |

**REQ-003.4** — The driver shall configure the hardware interrupt to assert
on the FIFO almost-full condition.

---

### 4.2 Data Acquisition

**REQ-001** — The driver shall read PPG sample data from the sensor hardware
FIFO via the I2C bus when the FIFO almost-full interrupt fires.

**REQ-001.1** — The driver shall read all available samples from the FIFO in a
single burst I2C transfer per interrupt event.

**REQ-001.2** — The driver shall parse each raw 3-byte FIFO entry into an
18-bit unsigned integer ADC value.

**REQ-001.3** — The driver shall maintain separate sample buffers for the RED
channel and the IR channel.

**REQ-001.4** — The driver shall detect and log FIFO overflow (OVF_COUNTER > 0).
On overflow, the driver shall discard the affected samples and reset the FIFO
read pointer. The driver shall not deliver samples from an overflowed FIFO
window to the application layer.

---

### 4.3 Data Delivery

**REQ-002** — The driver shall notify the application layer when new PPG sample
data is available, using the Zephyr `SENSOR_TRIG_DATA_READY` trigger mechanism.

**REQ-002.1** — The driver shall invoke the application-registered trigger
callback from a thread context (Zephyr system workqueue). The driver shall
not invoke the callback from ISR context.

**REQ-002.2** — The driver shall provide RED channel ADC values to the
application via `sensor_channel_get(SENSOR_CHAN_RED)`.

**REQ-002.3** — The driver shall provide IR channel ADC values to the
application via `sensor_channel_get(SENSOR_CHAN_IR)`.

**REQ-002.4** — The driver shall return only the most recently acquired sample
set when `sensor_channel_get()` is called. The driver is not required to buffer
historical samples beyond the current acquisition window.

---

### 4.4 Configuration

**REQ-006** — The driver shall allow the application to configure LED current
(RED and IR channels independently) via `sensor_attr_set()` at runtime.

**REQ-006.1** — The driver shall allow the application to configure the sensor
sample rate via `sensor_attr_set()`.

**REQ-006.2** — After any configuration change, the driver shall verify the
written register value by reading it back. If the read-back value does not
match the written value, the driver shall return an error.

---

### 4.5 Error Handling

**REQ-005** — The driver shall return a negative error code to the caller for
every I2C communication failure. The driver shall not silently ignore I2C errors.

**REQ-005.1** — The driver shall track consecutive I2C failures during
acquisition. After three consecutive failures, the driver shall attempt a sensor
reset and re-initialisation sequence.

**REQ-005.2** — If re-initialisation after consecutive failures does not succeed,
the driver shall place itself in a FAULT state and return `-EIO` to all subsequent
calls until the device is power-cycled or explicitly re-initialised.

**REQ-005.3** — The driver shall never enter an infinite blocking loop. All
I2C operations shall be bounded by the Zephyr I2C timeout (default: 100 ms).

---

## 5. Performance Requirements

**REQ-007** — The driver shall service the FIFO almost-full interrupt and
complete the FIFO burst read within 50 ms of interrupt assertion at the
default sample rate of 100 samples per second.

*Rationale: At 100 sps with FIFO almost-full threshold of 17 samples,
the FIFO fills in approximately 170 ms. A 50 ms service deadline provides
a 120 ms margin before overflow.*

**REQ-007.1** — The driver shall support sample rates of 50, 100, 200, 400,
800, and 1000 samples per second as defined in the MAX30101 datasheet.

**REQ-007.2** — The driver shall sustain continuous data acquisition at 100
samples per second for a minimum of 60 seconds without FIFO overflow under
normal system load conditions.

---

## 6. Interface Requirements

### 6.1 Hardware Interface — I2C

**REQ-008** — The driver shall communicate with the sensor exclusively via
the Zephyr I2C subsystem API. The driver shall not access I2C hardware registers
directly.

**REQ-008.1** — The driver shall use I2C address 0x57 as defined in the
MAX30101 datasheet.

**REQ-008.2** — The driver shall support both 100 kHz (Standard) and 400 kHz
(Fast) I2C bus speeds. The bus speed shall be configurable via DeviceTree.

### 6.2 Hardware Interface — GPIO Interrupt

**REQ-009** — The driver shall configure the INT pin as a falling-edge,
active-low interrupt using the Zephyr GPIO subsystem API.

**REQ-009.1** — The INT GPIO pin assignment shall be specified in the device's
DeviceTree source and shall not be hardcoded in driver source.

### 6.3 Software Interface — Zephyr Sensor API (northbound)

**REQ-010** — The driver shall implement the Zephyr sensor driver API and
shall be usable by any application that calls the standard Zephyr sensor
functions without knowledge of the underlying hardware.

**REQ-010.1** — The driver shall declare itself compatible with the
`"maxim,max30101"` DeviceTree binding string.

**REQ-010.2** — The driver's DeviceTree binding shall be defined in a YAML
schema file and shall be validated by `make dt_binding_check` without errors.

---

## 7. Safety Requirements

**REQ-004** — The driver shall not perform I2C bus operations or call blocking
functions from within the GPIO ISR context.

*Rationale: I2C operations in ISR context can cause system hangs or missed
interrupts on Zephyr platforms. This requirement prevents a class of system
reliability failures identified in FMEA-MAX30101-001 FM-007.*

**REQ-004.1** — The driver shall use the Zephyr system workqueue to defer
all FIFO read operations out of ISR context into thread context.

**REQ-011** — The driver shall not deliver samples to the application layer
that were acquired during a detected FIFO overflow event.

*Rationale: Overflow samples may be corrupted or incomplete. Delivering them
risks incorrect clinical data reaching the algorithm layer. Identified in
FMEA-MAX30101-001 FM-003.*

**REQ-012** — The driver shall be compatible with a system-level hardware
watchdog. The driver shall not hold any mutex or lock for longer than 10 ms.

*Rationale: Long lock hold times can prevent the system watchdog thread from
executing its feed sequence, causing unintended system reset.*

---

## 8. Constraints and Assumptions

**CON-001** — The driver assumes the MAX30101 is the only device at I2C
address 0x57 on the bus. No bus sharing with another device at this address
is supported.

**CON-002** — The driver is designed for a single-reader model. Only one
application task shall call `sensor_read()` concurrently. Concurrent calls
from multiple tasks produce undefined behaviour.

**CON-003** — The driver does not implement SpO2 or heart rate algorithms.
Raw ADC values are provided. Algorithm correctness is the responsibility of
the application layer.

**CON-004** — Supply voltage requirements (1.8 V I2C, 3.3 V LED supply)
are a hardware constraint, not enforced by this driver.

---

## 9. Traceability Matrix

| Requirement | Design Section (SDS-MAX30101-001) | FMEA Entry |
|---|---|---|
| REQ-001 | Section 4.2 — FIFO Management | FM-002, FM-003 |
| REQ-001.4 | Section 4.2 — Overflow handling | FM-003 |
| REQ-002 | Section 4.3 — Trigger Subsystem | FM-004 |
| REQ-002.1 | Section 4.3 — ISR behaviour | FM-007 |
| REQ-003 | Section 4.1 — Initialisation Subsystem | FM-005 |
| REQ-003.1 | Section 4.1 — Error handling | FM-005 |
| REQ-004 | Section 4.3 — ISR behaviour | FM-007 |
| REQ-005 | Section 4.2 — Error handling | FM-002 |
| REQ-006.2 | Section 4.1 — Register write verification | FM-008 |
| REQ-007 | Section 7 — Timing constraints | FM-003 |
| REQ-008 | Section 5.2 — I2C interface | FM-001, FM-002 |
| REQ-009 | Section 5.3 — GPIO interface | FM-004 |
| REQ-010 | Section 5.1 — Zephyr Sensor API | — |
| REQ-011 | Section 4.2 — Overflow handling | FM-003 |
| REQ-012 | Section 7 — Concurrency | FM-007 |
