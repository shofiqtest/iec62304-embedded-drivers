# Failure Mode and Effects Analysis (FMEA)
## MAX30101/MAX30102 Sensor Driver — Zephyr RTOS

| Field | Value |
|---|---|
| Document ID | FMEA-MAX30101-001 |
| Version | 1.0 |
| Author | Md Shofiqul Islam |
| Date | 2026-06-10 |
| Standard | IEC 62304:2006/Amd 1:2015 + ISO 14971:2019 |
| Device context | Wearable heart rate / SpO2 monitor (Class II) |

---

## Severity and Probability Definitions

**Severity (S):**
| Level | Label | Definition |
|---|---|---|
| 5 | Critical | Could cause serious patient injury or death |
| 4 | High | Could cause patient harm requiring medical intervention |
| 3 | Medium | Incorrect clinical data displayed; could lead to wrong decision |
| 2 | Low | Device malfunction; no direct patient harm |
| 1 | Negligible | Minor inconvenience; no clinical impact |

**Probability (P):**
| Level | Label | Definition |
|---|---|---|
| 5 | Frequent | Likely to occur in normal use |
| 4 | Probable | Will occur several times in device lifetime |
| 3 | Occasional | May occur once in device lifetime |
| 2 | Remote | Unlikely but conceivable |
| 1 | Improbable | Extremely unlikely; only in worst-case scenarios |

**Risk Level = S × P:**
| Score | Risk |
|---|---|
| 15–25 | HIGH — must mitigate before release |
| 8–14 | MEDIUM — mitigation required or justified acceptance |
| 1–7 | LOW — acceptable with standard controls |

---

## FMEA Table

### FM-001 — I2C Communication Failure at Initialisation

| Field | Value |
|---|---|
| Component | Initialisation Subsystem |
| Function | Verify sensor presence via PART_ID register read |
| Failure Mode | I2C NACK or bus timeout at boot |
| Cause | Wiring fault; wrong I2C address in DeviceTree; sensor not powered |
| Effect on System | Driver initialisation fails; device reports sensor unavailable |
| Effect on Patient | No heart rate / SpO2 data available; patient not monitored |
| Severity | 3 (Medium — clinician notified; can switch to backup) |
| Probability | 2 (Remote — hardware fault at manufacture or handling) |
| Risk | **6 — LOW** |
| Detection Method | `sensor_init()` returns `-EIO` or `-ENODEV`; application displays error |
| Control Measure | Initialisation failure logged; application-level alarm raised; device self-test at boot |
| Residual Risk | LOW — operator informed; no silent failure |

---

### FM-002 — I2C Communication Failure During Operation

| Field | Value |
|---|---|
| Component | FIFO Management Subsystem |
| Function | Burst-read 192 bytes of sample data from hardware FIFO |
| Failure Mode | I2C transfer returns `-EIO` mid-acquisition |
| Cause | Electromagnetic interference; loose connector; bus contention |
| Effect on System | Current FIFO read aborted; sample data lost for this acquisition cycle |
| Effect on Patient | Gap in PPG waveform; algorithm may produce incorrect reading |
| Severity | 3 (Medium — transient incorrect reading possible) |
| Probability | 2 (Remote — production-grade hardware) |
| Risk | **6 — LOW** |
| Detection Method | Driver logs error; marks sample as invalid; application discards invalid samples |
| Control Measure | Error returned to application layer; consecutive failure counter triggers sensor reset after 3 failures |
| Residual Risk | LOW — isolated transient errors discarded; persistent errors trigger reset and alarm |

---

### FM-003 — FIFO Overflow

| Field | Value |
|---|---|
| Component | FIFO Management Subsystem |
| Function | Read FIFO before it overflows (32-sample limit) |
| Failure Mode | `OVF_COUNTER > 0` detected on FIFO read |
| Cause | System workqueue blocked by higher-priority task; interrupt serviced too late |
| Effect on System | Up to 32 samples lost; continuity of PPG waveform broken |
| Effect on Patient | Missed heartbeats in data record; potential missed arrhythmia |
| Severity | 3 (Medium — depends on algorithm robustness to gaps) |
| Probability | 2 (Remote — latency budget analysis shows 170 ms margin at 100 sps) |
| Risk | **6 — LOW** |
| Detection Method | `OVF_COUNTER` register checked on every FIFO read; overflow logged |
| Control Measure | Overflow event flagged to application; FIFO reset; application discards result from this acquisition window |
| Residual Risk | LOW — overflow detected, not silent |

---

### FM-004 — Spurious or Missing Hardware Interrupt

| Field | Value |
|---|---|
| Component | Trigger Subsystem |
| Function | Detect FIFO almost-full via INT pin falling edge |
| Failure Mode A | Interrupt never fires — FIFO fills, overflows, driver not notified |
| Failure Mode B | Spurious interrupt — workqueue executes FIFO read when FIFO is empty |
| Cause A | INT GPIO misconfigured in DeviceTree; hardware INT pin not connected |
| Cause B | GPIO noise; shared interrupt line |
| Effect A | No data delivered to application; effectively same as FIFO overflow (FM-003) |
| Effect B | Empty FIFO read returns 0 samples; unnecessary CPU work |
| Severity A | 3 (Medium) / Severity B | 1 (Negligible) |
| Probability | 2 (Remote) |
| Risk A | **6 — LOW** / Risk B | **2 — LOW** |
| Detection Method | Watchdog timer: if no data received within 2× expected interval, raise alarm |
| Control Measure | Application-level timeout watchdog; hardware INT connection verified in manufacturing test |
| Residual Risk | LOW |

---

### FM-005 — Wrong Sensor Identity (Wrong Device on I2C Bus)

| Field | Value |
|---|---|
| Component | Initialisation Subsystem |
| Function | Confirm MAX30101 PART_ID = 0x15 at boot |
| Failure Mode | Different device present at I2C address 0x57; PART_ID check passes for wrong device |
| Cause | Assembly error; another sensor sharing address 0x57 responding |
| Effect on System | Driver configured for MAX30101 but reading data from wrong sensor |
| Effect on Patient | Completely wrong PPG data; incorrect heart rate / SpO2 displayed |
| Severity | 5 (Critical — wrong clinical data silently displayed) |
| Probability | 1 (Improbable — I2C address 0x57 is MAX30101-specific; requires assembly defect) |
| Risk | **5 — LOW** |
| Detection Method | PART_ID register verified against expected value 0x15; mismatch returns `-ENODEV` |
| Control Measure | PART_ID check is mandatory at initialisation; driver refuses to operate on mismatch |
| Residual Risk | LOW — control measure eliminates failure mode at boot |

---

### FM-006 — Silent Data Corruption in FIFO Read

| Field | Value |
|---|---|
| Component | FIFO Management Subsystem |
| Function | Parse raw 3-byte I2C payload into 18-bit ADC values |
| Failure Mode | Bit error in I2C transmission returns wrong ADC value without error flag |
| Cause | I2C without CRC (standard I2C has no error detection); EMI-induced bit flip |
| Effect on System | Single corrupted sample in data stream; application receives wrong ADC value |
| Effect on Patient | Single incorrect data point in PPG waveform; unlikely to affect heart rate algorithm if isolated |
| Severity | 3 (Medium — depends on algorithm design) |
| Probability | 1 (Improbable — I2C is robust on short PCB traces) |
| Risk | **3 — LOW** |
| Detection Method | Physiological range check: valid PPG ADC value is 0 to 262143 (18-bit); outliers rejected |
| Control Measure | Application-layer range validation; algorithmic outlier rejection; I2C SMBus CRC if hardware supports it |
| Residual Risk | LOW |

---

### FM-007 — Mutex Deadlock in Driver

| Field | Value |
|---|---|
| Component | FIFO Management + Trigger Subsystems |
| Function | Protect sample buffer with `k_mutex` |
| Failure Mode | Deadlock between workqueue handler and application calling `sensor_read()` |
| Cause | Application holds mutex, preempted by workqueue which waits for same mutex |
| Effect on System | Driver hangs; no further data delivered; system watchdog may reset device |
| Effect on Patient | Monitoring interrupted; alarm if watchdog resets device |
| Severity | 3 (Medium — device recovers via watchdog) |
| Probability | 1 (Improbable — Zephyr workqueue does not preempt cooperative threads holding mutex) |
| Risk | **3 — LOW** |
| Detection Method | System watchdog; application-level timeout; Zephyr lockdep (debug builds) |
| Control Measure | Mutex held for minimum duration (copy only); workqueue and application use try-lock with timeout |
| Residual Risk | LOW |

---

### FM-008 — LED Current Misconfiguration

| Field | Value |
|---|---|
| Component | Initialisation Subsystem |
| Function | Write LED current registers at boot |
| Failure Mode | Wrong LED current written — LED too dim or off |
| Cause | Wrong value passed via `sensor_attr_set()`; register write silently fails |
| Effect on System | PPG signal too weak; heart rate algorithm cannot detect peaks |
| Effect on Patient | No heart rate reading; missing data alarm raised |
| Severity | 2 (Low — no direct harm; device reports error) |
| Probability | 2 (Remote) |
| Risk | **4 — LOW** |
| Detection Method | Read-back LED current register after write; verify against written value |
| Control Measure | Register write followed by read-back verification at initialisation |
| Residual Risk | LOW |

---

## Risk Summary

| ID | Failure Mode | Initial Risk | Residual Risk |
|---|---|---|---|
| FM-001 | I2C failure at boot | LOW | LOW |
| FM-002 | I2C failure during operation | LOW | LOW |
| FM-003 | FIFO overflow | LOW | LOW |
| FM-004 | Missing/spurious interrupt | LOW | LOW |
| FM-005 | Wrong device on I2C bus | LOW | LOW |
| FM-006 | Silent data corruption | LOW | LOW |
| FM-007 | Mutex deadlock | LOW | LOW |
| FM-008 | LED current misconfiguration | LOW | LOW |

**All residual risks are LOW.** No HIGH risks remain unmitigated.

---

## Notes on Methodology

This FMEA was conducted by the driver author (Md Shofiqul Islam) as a bottom-up
software FMEA at the driver unit level. For a full medical device risk file under
ISO 14971, this FMEA feeds into the system-level hazard analysis which includes
hardware failure modes, use errors, and clinical context. Algorithm-level failure
modes (heart rate algorithm errors, SpO2 calculation errors) are out of scope for
this document and must be covered in a separate FMEA for the application layer.
