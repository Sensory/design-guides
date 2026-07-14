# Sensory Design Guides

Technical design guides for Sensory TrulyHandsfree (THF) and TrulyNatural (TNL) SDK integration.

These guides are confidential and shared under NDA. Do not distribute.

---

## Available Guides

### [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md)

How to measure real-time factor (RTF), MIPS, and memory usage for Sensory THF/TNL models using `snsr-eval`, `perf stat`, Valgrind `massif`, and standard Linux utilities. Covers peak and average CPU load, worst-case and steady-state memory, and MIPS measurement for continuous-listening wake word deployments.

**Applies to:** TrulyHandsfree (THF), TrulyNatural (TNL) SDKs — versions 7.6.1 / 7.7.0 / 7.8.0+
**Platform:** Raspberry Pi 4 (ARMv8 / NEON)
**Models used:** Voice Genie wake word + Automotive STT pipeline

---

### [Creating and Using Enrolled Models](enrolled-models.md)

How to create and use enrolled models with Sensory THF/TNL — training custom user-defined wake words/commands or voice-biometric models from a speaker's own recordings via `spot-enroll` and `live-enroll`, adapting context models, enrolling programmatically via the SDK API, and using an enrolled model at runtime (including combining it with a fixed model and converting it to a deeply embedded model for low-power targets).

**Applies to:** TrulyHandsfree (THF), TrulyNatural (TNL) SDKs — THF 5.x+ / TNL 7.6.1 / 7.7.0 / 7.8.0+
**Platform:** All THF/TNL-supported platforms (on-device enrollment only — no cloud enrollment for THF)
**Models used:** Enrollment task models (`udt-*.snsr`, `eft-*.snsr`) and their resulting enrolled models

---

*For questions or access requests, contact your Sensory FAE.*
