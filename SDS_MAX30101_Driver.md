# Software Design Specification
## MAX30101/MAX30102 Sensor Driver — Zephyr RTOS

| Field | Value |
|---|---|
| Document ID | SDS-MAX30101-001 |
| Version | 1.0 |
| Author | Md Shofiqul Islam |
| Date | 2026-06-10 |
| Status | Released |
| Reference | SRS-MAX30101-001, Zephyr PR #108697, IEC 62304:2006/Amd 1:2015 |

---

## 1. Purpose and Scope

This document describes the software design of the MAX30101/MAX30102 sensor driver
implemented for the Zephyr RTOS platform. The driver enables acquisition of
photoplethysmography (PPG) data for heart rate and SpO2 monitoring applications
on embedded microcontroller platforms.

This document satisfies IEC 62304 Clause 5.4 (Software Detailed Design) for the
sensor driver software unit.

**In scope:** Driver initialisation, I2C communication, interrupt handling,
FIFO management, data acquisition, Zephyr sensor API integration.

**Out of scope:** Heart rate and SpO2 algorithm computation; host application logic;
clinical interpretation of sensor data.

---

## 2. Definitions

| Term | Definition |
|---|---|
| PPG | Photoplethysmography — optical technique for measuring blood volume changes |
| SpO2 | Peripheral oxygen saturation measured via pulse oximetry |
| FIFO | First-In-First-Out hardware buffer inside the MAX30101 sensor |
| ISR | Interrupt Service Routine |
| DTS | Zephyr DeviceTree Source — hardware description file |
| SOUP | Software of Unknown Provenance (IEC 62304 Clause 8) |

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                 Application Layer                    │
│          (heart rate / SpO2 algorithms)              │
└──────────────────────┬──────────────────────────────┘
                       │ Zephyr Sensor API
                       │ sensor_read() / sensor_trigger_set()
┌──────────────────────▼──────────────────────────────┐
│            MAX30101 Driver (this component)          │
│                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │    Init     │  │  FIFO Mgmt   │  │  Trigger   │  │
│  │  Subsystem  │  │  Subsystem   │  │  Subsystem │  │
│  └──────┬──────┘  └──────┬───────┘  └─────┬──────┘  │
│         └────────────────┴────────────────┘          │
│                          │                            │
│              Zephyr I2C Driver API                    │
└──────────────────────────┬──────────────────────────┘
                           │ I2C bus (400 kHz)
┌──────────────────────────▼──────────────────────────┐
│            MAX30101 / MAX30102 Hardware               │
│         (I2C address 0x57, INT pin active-low)        │
└─────────────────────────────────────────────────────┘
```

---

## 4. Component Descriptions

### 4.1 Initialisation Subsystem

**Responsibility:** Configure the sensor hardware and Zephyr driver framework
on device boot.

**Sequence:**
1. Validate DeviceTree configuration (I2C bus handle, GPIO interrupt pin)
2. Verify I2C communication — read PART_ID register (expected: 0x15 for MAX30101)
3. Reset sensor via MODE_CONFIG register (reset bit)
4. Configure SpO2 registers: ADC range, sample rate, pulse width, LED current
5. Configure FIFO: almost-full threshold = 17 samples, rollover enabled
6. Register Zephyr interrupt callback on INT GPIO pin
7. Set driver state to READY

**Error handling:**
- I2C communication failure at step 2: return `-EIO`, device left uninitialised
- Wrong PART_ID: return `-ENODEV`
- GPIO configuration failure: return `-EINVAL`

### 4.2 FIFO Management Subsystem

**Responsibility:** Read accumulated sensor samples from the hardware FIFO
and deliver them to the application layer.

**FIFO structure:** Each sample is 3 bytes (18-bit ADC value, MSB-justified).
For SpO2 mode: RED channel sample followed by IR channel sample = 6 bytes per
measurement pair. Maximum FIFO depth: 32 samples.

**Read procedure:**
1. Read FIFO_WR_PTR and FIFO_RD_PTR registers
2. Calculate number of available samples: `(WR_PTR - RD_PTR) mod 32`
3. Burst-read all available samples via I2C
4. Parse raw bytes into 18-bit ADC values
5. Update internal sample buffer

**Overflow handling:**
If `OVF_COUNTER > 0`, log overflow event and reset FIFO pointers. Overflow
represents lost samples — the application layer is responsible for interpreting
the data gap.

### 4.3 Trigger Subsystem

**Responsibility:** Deliver data-ready notifications to the application layer
via the Zephyr sensor trigger API.

**Supported trigger:** `SENSOR_TRIG_DATA_READY` — fired when the hardware INT
pin asserts (active-low) indicating the FIFO almost-full condition has been reached.

**ISR behaviour:** The hardware interrupt is handled in the Zephyr GPIO ISR context.
The ISR does **not** perform I2C reads (I2C operations are not ISR-safe in Zephyr).
Instead, it submits a work item to the system workqueue. The workqueue handler
performs the FIFO read and invokes the application callback.

**Thread safety:** The driver uses Zephyr's `k_mutex` to protect concurrent
access to the sample buffer between the workqueue handler and `sensor_read()`.

---

## 5. External Interfaces

### 5.1 Zephyr Sensor API (northbound)

| Function | Description |
|---|---|
| `sensor_sample_fetch()` | Triggers a FIFO read; blocks until complete |
| `sensor_channel_get(SENSOR_CHAN_RED)` | Returns latest RED LED ADC value |
| `sensor_channel_get(SENSOR_CHAN_IR)` | Returns latest IR LED ADC value |
| `sensor_trigger_set()` | Registers application callback for DATA_READY |
| `sensor_attr_set()` | Configures LED current, sample rate, pulse width |

### 5.2 I2C Bus (southbound)

| Parameter | Value |
|---|---|
| I2C address | 0x57 (fixed, MAX30101 hardware) |
| Bus speed | Standard (100 kHz) or Fast (400 kHz) |
| Max transfer | 192 bytes (32 samples × 6 bytes) per burst read |
| Timeout | 100 ms (Zephyr I2C default) |

### 5.3 GPIO Interrupt Pin

| Parameter | Value |
|---|---|
| Signal | Active-low, open-drain |
| Trigger | Falling edge |
| Zephyr flag | `GPIO_INT_EDGE_FALLING` |

---

## 6. DeviceTree Binding

The driver requires the following DeviceTree node:

```dts
max30101: max30101@57 {
    compatible = "maxim,max30101";
    reg = <0x57>;
    interrupt-parent = <&gpio0>;
    interrupts = <5 GPIO_ACTIVE_LOW>;
};
```

Required properties are defined in `dts/bindings/sensor/maxim,max30101.yaml`.

---

## 7. Timing and Concurrency

| Constraint | Value | Basis |
|---|---|---|
| FIFO almost-full threshold | 17 samples | Configurable; default leaves margin before overflow |
| Sample rate (SpO2 mode) | 100 samples/sec | Register default; configurable via sensor_attr_set |
| FIFO fill time at 100 sps | ~170 ms | 17 samples ÷ 100 sps |
| Max I2C burst read time | ~5 ms | 192 bytes at 400 kHz |
| Interrupt latency budget | < 50 ms | Must service FIFO before overflow at 100 sps |

The workqueue handler must complete the FIFO read within the interrupt latency
budget. Applications using lower sample rates have proportionally more margin.

---

## 8. Dependencies (SOUP)

See companion document: `SOUP_Record_MAX30101_Driver.md`

| SOUP Item | Version | Role |
|---|---|---|
| Zephyr RTOS kernel | ≥ 3.4.0 | Task scheduling, workqueue, mutex |
| Zephyr I2C subsystem | ≥ 3.4.0 | I2C bus abstraction |
| Zephyr GPIO subsystem | ≥ 3.4.0 | Interrupt pin management |
| Zephyr Sensor API | ≥ 3.4.0 | Northbound interface framework |

---

## 9. Assumptions and Constraints

1. A single application task calls `sensor_read()` at a time. Concurrent calls
   are not supported and will result in undefined behaviour.
2. The I2C bus is dedicated to this sensor or properly arbitrated by the
   application. The driver does not perform bus arbitration.
3. The MAX30101 INT pin is connected to a GPIO capable of edge-triggered
   interrupts.
4. Supply voltage is 1.8 V for I2C interface, 3.3 V for LED supply, per
   MAX30101 datasheet Table 1.
5. This driver does not implement SpO2 or heart rate calculation algorithms.
   Raw ADC values only are provided to the application layer.

---

## 10. Traceability

| Requirement | Design Element |
|---|---|
| REQ-001: Driver shall read PPG data via I2C | Section 4.2 FIFO Management |
| REQ-002: Driver shall notify application on data ready | Section 4.3 Trigger Subsystem |
| REQ-003: Driver shall configure sensor at boot | Section 4.1 Initialisation |
| REQ-004: Driver shall not block in ISR context | Section 4.3 — workqueue pattern |
| REQ-005: Driver shall handle I2C failure gracefully | Section 4.1 error handling |
