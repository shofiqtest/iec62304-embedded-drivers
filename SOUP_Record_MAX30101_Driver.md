# SOUP Management Record
## MAX30101/MAX30102 Sensor Driver — Zephyr RTOS

| Field | Value |
|---|---|
| Document ID | SOUP-MAX30101-001 |
| Version | 1.0 |
| Author | Md Shofiqul Islam |
| Date | 2026-06-10 |
| Standard | IEC 62304:2006/Amd 1:2015 Clause 8 |

---

## What is SOUP?

Per IEC 62304 Clause 3.1.6: *Software of Unknown Provenance (SOUP)* is software
that is already developed and generally available, and that has not been developed
for the purpose of being incorporated into the medical device.

Every third-party or open-source component the driver depends on is SOUP and
must be managed under this record.

---

## SOUP Item 1 — Zephyr RTOS Kernel

| Field | Value |
|---|---|
| SOUP ID | SOUP-001 |
| Name | Zephyr RTOS |
| Version | 3.6.0 (pinned) |
| Manufacturer | Zephyr Project (Linux Foundation) |
| License | Apache 2.0 |
| Source | github.com/zephyrproject-rtos/zephyr |
| IEC 62304 Class | Class B (contributes to non-serious injury risk) |

**Intended use in this software:**
Task scheduling, system workqueue for deferred interrupt handling, mutex for
concurrent access protection, and kernel timing services.

**Functional requirements on this SOUP:**
- SHALL provide preemptive task scheduling with deterministic context switch
- SHALL provide a system workqueue capable of executing submitted work items
  within 50 ms of submission under normal load
- SHALL provide a mutex primitive that prevents concurrent access to the
  sample buffer from ISR workqueue and application task

**Failure modes and effects:**

| Failure | Effect on device | Severity |
|---|---|---|
| Workqueue starvation — work item not executed | Sensor data not delivered to application; heart rate display freezes | Medium |
| Mutex deadlock | Driver becomes unresponsive; application hangs | High |
| Incorrect task scheduling | Timing guarantees lost; data latency unpredictable | Medium |

**Known anomalies:** Monitor Zephyr GitHub issue tracker and CVE database for
version 3.6.0. At time of writing: no safety-relevant known anomalies.

**Verification approach:**
- Integration test: verify data delivery at configured sample rate over 60-second
  continuous acquisition run
- Stress test: concurrent sensor_read() and interrupt delivery under maximum
  sample rate
- Static analysis: Zephyr project runs CI with undefined behaviour sanitiser

---

## SOUP Item 2 — Zephyr I2C Subsystem

| Field | Value |
|---|---|
| SOUP ID | SOUP-002 |
| Name | Zephyr I2C Driver Subsystem |
| Version | 3.6.0 (part of Zephyr RTOS) |
| Manufacturer | Zephyr Project (Linux Foundation) |
| License | Apache 2.0 |
| IEC 62304 Class | Class B |

**Intended use:**
Provides the `i2c_write_read()` and `i2c_burst_read()` APIs used for all
communication with the MAX30101 sensor hardware.

**Functional requirements on this SOUP:**
- SHALL complete I2C transfers within 100 ms timeout
- SHALL return `-EIO` on bus error (NACK, arbitration loss, timeout)
- SHALL support burst reads of up to 192 bytes in a single transaction

**Failure modes and effects:**

| Failure | Effect | Severity |
|---|---|---|
| Silent data corruption on I2C read | Wrong sensor values delivered to application | High |
| Timeout not enforced | Driver blocks indefinitely; system hangs | High |
| Bus not released after error | All subsequent I2C devices on bus inaccessible | Medium |

**Verification approach:**
- Fault injection test: disconnect sensor mid-acquisition, verify `-EIO` returned
- Timeout test: configure sensor to NAK, verify driver returns within 200 ms
- Checksum: MAX30101 PART_ID read at initialisation provides implicit communication check

---

## SOUP Item 3 — Zephyr GPIO Subsystem

| Field | Value |
|---|---|
| SOUP ID | SOUP-003 |
| Name | Zephyr GPIO Driver Subsystem |
| Version | 3.6.0 (part of Zephyr RTOS) |
| Manufacturer | Zephyr Project (Linux Foundation) |
| License | Apache 2.0 |
| IEC 62304 Class | Class A (failure causes missed interrupt; no direct patient harm) |

**Intended use:**
Configures the MAX30101 INT pin as a falling-edge interrupt source and registers
the ISR callback that triggers FIFO reads.

**Functional requirements on this SOUP:**
- SHALL detect falling edge on INT GPIO within one sample period (10 ms at 100 sps)
- SHALL invoke registered callback in ISR context

**Failure modes and effects:**

| Failure | Effect | Severity |
|---|---|---|
| Interrupt missed | FIFO fills; new data not delivered until next interrupt | Low (data delayed, not corrupted) |
| Callback invoked spuriously | Unnecessary FIFO read; no sample corruption | Low |

**Verification approach:**
- Hardware test: oscilloscope on INT pin vs workqueue execution timestamp;
  verify latency < 5 ms

---

## SOUP Item 4 — Zephyr Sensor API

| Field | Value |
|---|---|
| SOUP ID | SOUP-004 |
| Name | Zephyr Sensor Subsystem API |
| Version | 3.6.0 (part of Zephyr RTOS) |
| Manufacturer | Zephyr Project (Linux Foundation) |
| License | Apache 2.0 |
| IEC 62304 Class | Class A |

**Intended use:**
Provides the standardised northbound interface (`sensor_sample_fetch`,
`sensor_channel_get`, `sensor_trigger_set`) that the application layer
uses to interact with the driver without hardware-specific knowledge.

**Failure modes and effects:**

| Failure | Effect | Severity |
|---|---|---|
| channel_get returns wrong channel enum | Application reads wrong data channel | Medium |
| trigger_set callback not invoked | Application never receives data | Medium |

---

## SOUP Monitoring Plan

| Activity | Frequency | Owner |
|---|---|---|
| Check Zephyr release notes for security/safety fixes | Each Zephyr release | Driver maintainer |
| Search CVE database for Zephyr RTOS | Monthly | Driver maintainer |
| Review Zephyr GitHub issues tagged `bug` for I2C / GPIO subsystems | Quarterly | Driver maintainer |
| Re-run integration test suite after any SOUP version update | On update | QA |

---

## Version Update Procedure

When updating any SOUP item to a new version:
1. Update version field in this record
2. Review changelog for changes affecting functional requirements listed above
3. Re-run full driver test suite (see FMEA_MAX30101_Driver.md for test coverage)
4. Document results in this record under a new revision entry
5. Obtain sign-off before releasing updated software
