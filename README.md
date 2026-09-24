# Sensory Design Guides

Technical design guides for Sensory TrulyHandsfree (THF) and TrulyNatural (TNL) SDK integration.

These guides are confidential and shared under NDA. Do not distribute.

---

## Available Guides

### [Getting Started with Sensory THF and TNL Lite/STT SDKs](getting-started-thf-tnl.md)

An onboarding overview for developers new to Sensory's SDKs — what THF and TNL Lite/TNL STT each provide, how wake words, voice commands, speech-to-text, and voice biometrics map onto those two product families, how to install the SDK and activate a license, and where to find sample code, domain-specific models (automotive, IoT, wearables), and the rest of this repo's deeper how-to guides. Start here before the guides below.

**Applies to:** TrulyHandsfree (THF), TrulyNatural Lite/TNL STT SDKs — THF 5.x+ / TNL 7.8.0+ (7.9.0 recommended)
**Platform:** Windows, Linux, macOS (development); iOS, Android (deployment via platform bindings)
**Models used:** Any pre-trained wake word, command, or STT model shipped with the SDK

---

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

### [How to Save Debug Audio and Data in THF/TNL](debug-audio-capture.md)

How to capture the exact audio and event data a THF/TNL model saw on a customer's device using the `tpl-spot-debug` template — primarily by wrapping a live session at runtime in an Android/iOS app (following the SDK's `snsr-debug` sample), with a command-line model-swap alternative for PC bench testing — then extract and replay it in simulation with `snsr-log-split`, `snsr-eval`, and `audio-check`. Covers slot-addressing changes introduced by the debug wrapper, on-device storage location trade-offs, a lightweight in-memory alternative for storage-constrained devices, and the security/privacy handling required since this tool intentionally breaks Sensory's all-edge, nothing-saved design philosophy.

**Applies to:** TrulyHandsfree (THF), TrulyNatural (TNL) SDKs — THF/TNL 6.x+ / 7.6.1 / 7.7.0 / 7.8.0+
**Platform:** All THF/TNL-supported platforms
**Models used:** `tpl-spot-debug-1.5.1.snsr` wrapping any wake word, command, or enrolled phrase-spotter model

---

### [Measuring Word Error Rate (WER) for TrulyNatural STT Models](measuring-stt-wer.md)

How to measure the word error rate (WER) of a TNL STT model against a labeled audio corpus using `snsr-eval-batch` — obtaining a standalone STT task file (either downloaded directly or extracted from an assembled pipeline with `snsr-edit`), preparing a test corpus with reference transcripts, generating the CSV manifest `snsr-eval-batch` requires, and interpreting the resulting substitution/insertion/deletion and WER figures. Covers SDK-version differences in audio format support (FLAC vs. WAV) and common pitfalls like inconsistent normalization between runs.

**Applies to:** TrulyNatural (TNL) SDK — 7.9.0+ recommended, 7.8.0 also supported
**Platform:** PC/workstation (Windows, Linux, or macOS) — not an embedded-target measurement
**Models used:** TNL STT models (e.g. `stt-enUS-general-*.snsr`, `stt-enUS-automotive-*.snsr`)

---

*For questions or access requests, contact your Sensory FAE.*
