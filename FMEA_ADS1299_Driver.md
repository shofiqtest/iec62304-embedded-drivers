# FMEA — TI ADS1299 Linux Kernel IIO EEG ADC Driver

**ISO 14971:2019 — Failure Mode and Effects Analysis**

| Field | Value |
|---|---|
| **Component** | ADS1299 Linux kernel IIO driver |
| **Document version** | 1.0 — 2026-06-30 |
| **Severity scale** | 1 (negligible) → 5 (catastrophic) |
| **Probability scale** | 1 (very low) → 5 (frequent) |
| **Risk** | Severity × Probability: ≤4 acceptable, 5–9 ALARP, ≥10 unacceptable |

---

## Failure Modes

| ID | Failure Mode | Effect on Patient / System | Severity | Probability | Risk | Mitigation |
|---|---|---|---|---|---|---|
| FM-01 | SPI transfer error — corrupted sample | Incorrect EEG data displayed or recorded | 4 | 2 | 8 | SPI bus integrity check; application-layer plausibility filter; alarm on out-of-range values |
| FM-02 | DRDY interrupt missed — sample dropped | Missing EEG epoch; gap in recording | 3 | 2 | 6 | Application watchdog; epoch continuity check; re-sync on gap detection |
| FM-03 | Wrong PGA gain written to CHnSET | Signal amplitude error — misdiagnosis risk | 4 | 1 | 4 | Read-back gain register after write; verify at driver init |
| FM-04 | Internal reference not powered up | Zero output on all channels | 4 | 1 | 4 | Verify CONFIG3 register at init; application-level zero-signal alarm |
| FM-05 | IIO triggered buffer overflow | Samples lost; acquisition gap | 2 | 2 | 4 | Increase buffer depth via iio_buffer_set_length(); monitor overrun counter |
| FM-06 | Kernel driver crash | Acquisition stops; system may restart | 4 | 1 | 4 | Linux kernel memory protection; watchdog restart; redundant recording path |
| FM-07 | SPI clock frequency exceeds ADS1299 spec | Register corruption; invalid samples | 5 | 1 | 5 | Enforce spi-max-frequency ≤ 20 MHz in Device Tree; validated at bring-up |
| FM-08 | Power supply glitch during acquisition | Saturation / rail-to-rail output on all channels | 4 | 1 | 4 | Power supply monitoring; discard samples during supply transient |
| FM-09 | Lead-off condition undetected by driver | Flat-line signal misread as valid EEG | 5 | 2 | 10 | **Mandatory** lead-off detection at application layer; alarm on low-variance signal |
| FM-10 | Device Tree misconfiguration (wrong IRQ polarity) | DRDY interrupt never fires; no acquisition | 3 | 1 | 3 | Validated DT binding; board bring-up test procedure |

---

## Risk Summary

| Risk level | Count | Items |
|---|---|---|
| Unacceptable (≥10) | 1 | FM-09 |
| ALARP (5–9) | 3 | FM-01, FM-02, FM-07 |
| Acceptable (≤4) | 6 | FM-03, FM-04, FM-05, FM-06, FM-08, FM-10 |

**FM-09 requires mandatory application-layer lead-off detection before clinical use.**

---

## Residual Risk Acceptance

After application-layer mitigations are implemented and verified, all residual
risks are classified as acceptable per ISO 14971:2019 §7.

Reviewed by: Md Shofiqul Islam · 2026-06-30
