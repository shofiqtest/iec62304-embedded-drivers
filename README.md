# Embedded Medical Software Portfolio
### Md Shofiqul Islam — Embedded Linux Engineer & Linux Kernel Contributor

This repository contains IEC 62304-aligned software engineering artefacts based on
the MAX30101/MAX30102 PPG sensor driver I authored and merged into Zephyr RTOS
([PR #108697](https://github.com/zephyrproject-rtos/zephyr/pull/108697)).

These documents demonstrate practical application of medical device software
engineering standards to a real, publicly verifiable upstream open source driver.

---

## Documents

| Document | Standard | Description |
|---|---|---|
| [SRS](SRS_MAX30101_Driver.md) | IEC 62304 §5.2 | Software Requirements Specification — 12 shall-statements covering functional, performance, interface, and safety requirements |
| [SDS](SDS_MAX30101_Driver.md) | IEC 62304 §5.4 | Software Design Specification — architecture, component design, interfaces, timing, concurrency |
| [SOUP Record](SOUP_Record_MAX30101_Driver.md) | IEC 62304 §8 | SOUP management record for all Zephyr RTOS dependencies with risk classification and monitoring plan |
| [FMEA](FMEA_MAX30101_Driver.md) | ISO 14971:2019 | Failure Mode and Effects Analysis — 8 failure modes with severity, probability, risk level, and mitigations |

All requirements in the SRS trace to design sections in the SDS and failure modes in the FMEA.

---

## The Driver

The MAX30101/MAX30102 driver is a complete bare-metal RTOS sensor driver:
- I2C protocol, interrupt-driven FIFO sampling, hardware abstraction
- Zephyr Sensor API, DeviceTree bindings, Kconfig, CI
- Merged into Zephyr mainline: [PR #108697](https://github.com/zephyrproject-rtos/zephyr/pull/108697)

---

## About

14 upstream Linux kernel patches · IIO · DRM/Accel · SCSI · SCTP · MFD · DT bindings  
Reviewed by Intel · Red Hat · Microsoft · Linaro  
Master of Health Sciences, Biomedical Engineering · University of Oulu, Finland  

[lore.kernel.org](https://lore.kernel.org/all/?q=Md+Shofiqul+Islam) · [LinkedIn](https://www.linkedin.com/in/mdshofiqul/)
