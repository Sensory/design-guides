# Capturing Debug Audio and Data in THF/TNL

**Product:** TrulyHandsfree (THF) / TrulyNatural (TNL) SDK
**Models:** `tpl-spot-debug` (debug template), wrapping any wake word, command, or enrolled phrase-spotter model
**Version:** THF/TNL 6.x+ / 7.6.1 / 7.7.0 / 7.8.0+ (`tpl-spot-debug-1.5.1.snsr` shown below — the template's own version number increments independently of the SDK version, so always use the copy shipped with your installed SDK)
**Document Version:** 1.1.0
**Audience:** Field Application Engineers, Customer Integration Engineers
**Status:** Released under NDA — Do not distribute

*Copyright © 2026 Sensory Inc. All rights reserved. This document is confidential and proprietary to Sensory Inc. It may not be reproduced, distributed, or disclosed to any third party without prior written permission from Sensory Inc.*

---

## Table of Contents

1. [Overview](#1-overview)
   - [1.1 Purpose](#11-purpose)
   - [1.2 The Troubleshooting Workflow](#12-the-troubleshooting-workflow)
2. [Security and Privacy](#2-security-and-privacy)
3. [Prerequisites](#3-prerequisites)
4. [Over-The-Air vs. Simulation Testing](#4-over-the-air-vs-simulation-testing)
5. [Capturing Debug Audio from an App](#5-capturing-debug-audio-from-an-app)
   - [5.1 Reference Implementation: the snsr-debug Sample](#51-reference-implementation-the-snsr-debug-sample)
   - [5.2 Debug Template Settings Reference](#52-debug-template-settings-reference)
   - [5.3 Wrapping the Session in the Debug Template](#53-wrapping-the-session-in-the-debug-template)
   - [5.4 Slot Addressing Changes When Wrapped](#54-slot-addressing-changes-when-wrapped)
   - [5.5 Where the Log File Is Written](#55-where-the-log-file-is-written)
   - [5.6 Pulling the Log Off the Device](#56-pulling-the-log-off-the-device)
6. [PC Bench-Testing Alternative: Command-Line Model Swap](#6-pc-bench-testing-alternative-command-line-model-swap)
   - [6.1 Building and Running a Debug-Wrapped Model](#61-building-and-running-a-debug-wrapped-model)
   - [6.2 Compiling the Debug Template into Firmware](#62-compiling-the-debug-template-into-firmware)
7. [Extract, Verify, and Compare in Simulation](#7-extract-verify-and-compare-in-simulation)
   - [7.1 Extract Audio and Events with snsr-log-split](#71-extract-audio-and-events-with-snsr-log-split)
   - [7.2 Verify Captured Audio Quality with audio-check](#72-verify-captured-audio-quality-with-audio-check)
   - [7.3 Reproduce and Compare in Simulation](#73-reproduce-and-compare-in-simulation)
   - [7.4 Requesting a Free Log Review from Sensory](#74-requesting-a-free-log-review-from-sensory)
8. [Lightweight Alternative: In-Memory Audio Capture](#8-lightweight-alternative-in-memory-audio-capture)
9. [Design Guidance](#9-design-guidance)
   - [9.1 Storage and Memory Budgeting](#91-storage-and-memory-budgeting)
   - [9.2 Timestamped, Non-Colliding Log Filenames](#92-timestamped-non-colliding-log-filenames)
   - [9.3 The include-model Trade-off](#93-the-include-model-trade-off)
   - [9.4 Choosing a Capture Approach](#94-choosing-a-capture-approach)
10. [Common Pitfalls](#10-common-pitfalls)
11. [Quick Reference — Command Summary](#11-quick-reference--command-summary)
12. [References](#12-references)

---

## 1. Overview

### 1.1 Purpose

This guide explains how to capture the actual audio and event data a Sensory THF/TNL model saw on a customer's device-under-test (DUT), so it can be replayed and diagnosed in simulation on a PC. This is the primary tool for diagnosing a report of the form *"it doesn't perform as well on our device as it does in the SDK."* Without a recording of exactly what the model heard and how it responded, that gap is nearly impossible to root-cause — you're left guessing between a microphone/audio-path problem, an integration bug, and a genuine model accuracy issue.

### 1.2 The Troubleshooting Workflow

Debugging a field accuracy report follows the same three-step loop no matter how the audio gets captured:

1. **Capture** — Record audio (and, preferably, the model and event timeline) from the DUT using the debug template.
2. **Recreate** — Reproduce the failure in simulation on a PC, and compare results between the on-device (over-the-air) run and the simulated run using the *original, non-debug* model. This isolates whether the problem is in the model itself or in the audio path leading up to it.
3. **Fix** — Address the suspected problem: a microphone/gain/AEC issue in the customer's audio pipeline, an SDK integration bug, or a genuine model accuracy gap that needs to go back to Sensory.

The rest of this guide walks through each part of that loop in detail.

---

## 2. Security and Privacy

Capturing debug audio is a deliberate, temporary departure from Sensory's **all-edge, nothing-saved** design philosophy: THF/TNL normally processes audio in memory and discards it, transmitting and persisting nothing. `tpl-spot-debug` exists specifically to break that guarantee — it writes raw, unencrypted audio to persistent storage — which makes it a capability that has to be handled deliberately, not something to leave switched on.

> **Important:** Enabling debug capture means user speech is being written to disk on a real device, potentially in a customer's production environment. Treat this the same way you'd treat any other sensitive/biometric data collection.

Before shipping or using debug capture in the field, apply the following safeguards:

- **Gate it behind an explicit, defaulted-off switch** — a hidden developer menu, a build flag, or a remote config flag. Never ship a production build with debug capture enabled by default.
- **Get consent before capturing in the field.** If you're recording a customer's or end user's speech to diagnose an issue, make sure they know it's happening and why.
- **Treat captured logs as sensitive data once they exist.** Restrict who can access the `.snsrlog`/`.wav` files pulled off a device, don't drop them into shared drives without access control, and delete them once they've served their diagnostic purpose — both on the device and on whatever machine you pulled them to.
- **Storage location matters** — see [5.5](#55-where-the-log-file-is-written) for why app-external storage is more exposed than app-internal storage, and choose deliberately based on who needs access and how sensitive the audio is.
- **`include-model=1`** additionally embeds a copy of the model file in every log — if that model is customer-specific or otherwise proprietary, that's an IP consideration on top of the audio privacy consideration. See [9.3](#93-the-include-model-trade-off).

This guide describes how to make debug capture *possible*. Whether and when to actually enable it in a given build or deployment is a product decision — make it deliberately, not by accident.

---

## 3. Prerequisites

- The SDK's `snsr-debug` sample (or your own application, following the same pattern) as your reference implementation for on-device capture — see [Section 5](#5-capturing-debug-audio-from-an-app). It ships under `sample/android/snsr-debug` in the SDK distribution; the same wrapping pattern applies on iOS via the SDK's platform bindings, even though no dedicated iOS sample ships for it.
- The debug template shipped with your SDK, e.g. `model/tpl-spot-debug-1.5.1.snsr`.
- The spotter model you want to debug — a fixed wake word, command, LVCSR, STT, or enrolled model. Combined and VoiceHub-trained models are supported.
- CLI tools accessible on your PC's `PATH` for the post-processing side, regardless of how you captured the audio:
  - `snsr-log-split` — splits a debug log into its audio, event, and model components
  - `audio-check` — checks a WAV file for clipping, dropouts, and estimates SNR
  - `snsr-eval` — replays a WAV file through a model in simulation
  - `snsr-edit` — only needed if you use the PC bench-testing alternative in [Section 6](#6-pc-bench-testing-alternative-command-line-model-swap)
- `adb` (or your platform's equivalent) to pull the captured log off the DUT — see [5.6](#56-pulling-the-log-off-the-device).
- Sufficient free storage on the DUT to hold the captured log — see [9.1 Storage and Memory Budgeting](#91-storage-and-memory-budgeting).
- Sign-off that debug capture is appropriate and permittes for this build/device — see [Section 2](#2-security-and-privacy).

---

## 4. Over-The-Air vs. Simulation Testing

| | Over-The-Air Testing | Simulation Testing |
|---|---|---|
| **Runs on** | The DUT, running your test/production application | A PC, running `snsr-eval` |
| **Audio source** | Live microphone audio | Pre-recorded audio (ideally captured from the DUT itself) |
| **Results** | Acted on by the system in real time | Logged and compared against the over-the-air results |

Over-the-air testing exercises the customer's actual microphone, audio front-end, and integration code — it's the only way to see the real-world conditions the model has to work with. Simulation testing is repeatable and fast to iterate on, but it's only as representative as the audio fed into it.

> **In practice:** the normal workflow is to capture debug audio from the DUT over-the-air, using live microphone input exactly as the end user would trigger it — that's what Section 5 walks through. That captured audio is then replayed and recreated in simulation on a PC ([Section 7](#7-extract-verify-and-compare-in-simulation)) so the comparison is reproducible. Capturing live is what makes the simulation trustworthy; a simulation run against SDK reference audio alone can't tell you anything about what's actually happening on this DUT.

---

## 5. Capturing Debug Audio from an App

Most debug audio capture happens from an Android or iOS application running on the actual DUT, not from a PC-attached microphone. `tpl-spot-debug` adds runtime audio and event-timing capture to a wake word model — it has `task-type == phrasespot`, expects a `phrasespot`-type model in slot 0, and produces a model that behaves the same as the one it wraps for the purposes of your app's event handling (with one addressing caveat — see [5.4](#54-slot-addressing-changes-when-wrapped)).

### 5.1 Reference Implementation: the snsr-debug Sample

The SDK ships a complete working example at `sample/android/snsr-debug`. It's a small demo app that runs recognition in one of four modes — wake word only, wake word + commands, STT only, and wake word + STT — with a checkbox that turns debug capture on or off for the current session. Use it as the reference implementation for adding this capability to your own app; the pattern in [5.3](#53-wrapping-the-session-in-the-debug-template) below is drawn directly from it.

### 5.2 Debug Template Settings Reference

`tpl-spot-debug` adds two settings of its own on top of whatever settings the wrapped model already supports (operating point, `sv-threshold`, etc. all still apply and pass through unchanged):

| Setting | Type | Description |
|---|---|---|
| `debug-log-file` (`SNSR_DEBUG_LOG_FILE` / `Snsr.DEBUG_LOG_FILE`) | string, read-write | **Required** — the log file `tpl-spot-debug` writes to. No default is defined; the directory must already exist and be writable. Supports two mutually exclusive timestamp substitutions: `%@` (`year-month-day_hour-minute-second.milliseconds`, UTC) and `%#` (milliseconds since the Unix epoch). |
| `include-model` (`SNSR_INCLUDE_MODEL` / `Snsr.INCLUDE_MODEL`) | int, read-write | Whether the log includes a copy of the wrapped task model (the `.snsr` file). **Default is `1`.** Set to `0` for smaller, less complete debug logs — see [9.3](#93-the-include-model-trade-off). |
| `0` (`SNSR_SLOT_0` / `Snsr.SLOT_0`) | stream | The slot the wrapped `phrasespot`-type model is loaded into. Literal value `"0."`. |

Log files contain time-stamped entries with: SDK library information, the spotter model being used (if `include-model=1`), captured audio samples, and event callbacks (`^result`, `^sample-count`). `snsr-log-split` extracts these into separate `.txt`, `.wav`, and `.snsr` files ([7.1](#71-extract-audio-and-events-with-snsr-log-split)).

### 5.3 Wrapping the Session in the Debug Template

There's one wrinkle that shapes how this is implemented: `SnsrSession.load()` can only be called once per session. If you loaded the debug template *instead of* your spotter model, you'd have to restructure however your app already builds its session — wake word only, wake word + commands, STT, wake word + STT, whatever combination it supports.

The `snsr-debug` sample avoids that by leaving its normal session-building logic completely untouched, and only *afterward* — if debug capture is enabled — creating a **second** session for the debug template and transplanting the already-loaded model into it as an in-memory stream:

```java
if (mDebugging) {
    // Create debug session
    SnsrSession debug = new SnsrSession();
    debug.load(assetToString(BuildConfig.DEBUG_TEMPLATE));
    debug.setString(Snsr.DEBUG_LOG_FILE, mLogPath);
    debug.setInt(Snsr.INCLUDE_MODEL, 0);
    // Load existing session into the debug model
    SnsrStream modelData = SnsrStream.fromBuffer(1<<20, 1<<30);
    session.save(SnsrDataFormat.CONFIG, modelData);
    session.release();
    debug.setStream(Snsr.SLOT_0, modelData);
    modelData.release();
    // Replace session with the same model wrapped in the tpl-spot-debug template
    session = debug;

    // Show the debug log file name in the UI
    File file = new File(mLogPath);
    mUi.logToConsole("\nAudio will be logged to " + file.getName());
}

// Main result handler
session.setHandler(Snsr.RESULT_EVENT, this);
```

*(from `PhraseSpot.java` in the `snsr-debug` sample)*

The key move is `session.save(SnsrDataFormat.CONFIG, modelData)` followed by `debug.setStream(Snsr.SLOT_0, modelData)`: instead of reopening the spotter model from its original asset/file path a second time, the already-configured session is serialized to an in-memory buffer and handed to the debug session's slot 0 directly. `session.load()` is still only ever called once — the entire block above is the only code that needs to know about debugging at all, and it runs after every other mode-specific setup path (wake word only, wake word + commands, STT, wake word + STT) has already built `session` normally.

### 5.4 Slot Addressing Changes When Wrapped

The debug-wrapped model is **not a strict drop-in replacement** — wrapping adds one more level of slot nesting, because your original model (and whatever slots it defines) now lives inside the debug template's own slot 0.

The session's own top-level `^result` handler is unaffected — `Snsr.RESULT_EVENT` (`"^result"`) is registered the same way whether or not debugging is on, since `tpl-spot-debug` forwards the wrapped model's top-level result event unprefixed. What changes is any address that reaches into a **specific slot** of your original model. `Snsr.SLOT_0` is the literal string `"0."`, so an address like `Snsr.SLOT_0 + Snsr.RESULT_EVENT` (`"0.^result"`) — used, for example, to catch the wake word result inside a wake-word-then-STT pipeline — needs an extra `Snsr.SLOT_0` prefix once that pipeline is itself sitting in the debug template's slot 0:

```java
try {
    if (mDebugging) {
        session.setHandler(Snsr.SLOT_0 + Snsr.SLOT_0 + Snsr.RESULT_EVENT, this);
    } else {
        session.setHandler(Snsr.SLOT_0 + Snsr.RESULT_EVENT, this);
    }
} catch (Exception ignored) {}
```

The two addresses generally can't share a single handler case, because `getString` also needs the slot-qualified address to read the result's *text field* — not just to register the handler:

```java
    @Override
    public SnsrRC onEvent(SnsrSession s, String key) {
        switch (key) {
...
            case Snsr.SLOT_0 + Snsr.RESULT_EVENT:
                // callback for a wakeword result in WW_CMDS mode
                mUi.logToConsole(String.format(Locale.US, "Wakeword: '%s'",
                        s.getString(Snsr.SLOT_0 + Snsr.RES_TEXT)));
                return SnsrRC.OK;
            case Snsr.SLOT_0 + Snsr.SLOT_0 + Snsr.RESULT_EVENT:
                // callback for a wakeword result in WW_STT mode
                mUi.logToConsole(String.format(Locale.US, "Wakeword: '%s'",
                        s.getString(Snsr.SLOT_0 + Snsr.SLOT_0 + Snsr.RES_TEXT)));
                return SnsrRC.OK;
...
        }
    }
```

*(from `PhraseSpot.java` — registering the wake word result handler for the wake-word-then-STT mode)*

Non-debug: `"0.^result"`. Debug-wrapped: `"0.0.^result"`. In most cases this doesn't matter because most apps only listen for the top-level `^result`. But if your app hardcodes any slot-qualified event or setting address (common with `tpl-spot-sequential`, `tpl-spot-vad-lvcsr`, or any other multi-slot template), audit those addresses when adding debug wrapping — each one needs the extra `Snsr.SLOT_0` prefix, and it's an easy thing to miss since the app otherwise looks and behaves identically.

### 5.5 Where the Log File Is Written

Where you point `debug-log-file` determines who can retrieve it later, and how much friction that takes. The sample's `MainActivity.java` builds its log path like this:

```java
File base = getFilesDir();
File sensoryFolder = new File(base, "logs");
sensoryFolder.mkdirs();
long sec = System.currentTimeMillis() / 1000;
String logName = getString(R.string.app_name) + "-" + sec + ".snsrlog";
mLogFile = new File(sensoryFolder, logName);
```

`getFilesDir()` is **app-private internal storage** — it's sandboxed, and on a non-rooted, non-debuggable production device there is no way to retrieve files from it (not even via `adb pull`) without `adb shell run-as <package>`, which itself requires a debuggable build. That's fine for the sample app (which is a debug tool by definition), but it's a real limitation for capturing on a customer's shipping, non-debuggable device.

For that case, use `getExternalFilesDir(null)` instead:

```java
File base = getExternalFilesDir(null);
```

This still scopes the files to your app (under `/sdcard/Android/data/<package>/files/...`), but it's retrievable with a plain `adb pull` — or even a file manager — on a non-rooted, non-debuggable device, with no `run-as` needed. Weigh this against Section 2: external storage is more accessible, which is convenient for retrieval but also means the captured audio is easier for anything else on the device to reach. Choose deliberately based on the device and the sensitivity of what's being captured, not by copy-pasting the sample as-is.

### 5.6 Pulling the Log Off the Device

If the log was written to app-internal storage (`getFilesDir()`), retrieval on a debuggable build looks like:

```bash
adb -d shell "run-as com.sensory.speech.snsr.demo.snsrdebug tar -C /data/user/0/com.sensory.speech.snsr.demo.snsrdebug/files -cf - logs" | tar -xf -
```

If it was written to app-external storage (`getExternalFilesDir(null)`), a plain pull is enough and doesn't require a debuggable build or `run-as`:

```bash
adb pull /sdcard/Android/data/<package>/files/logs
```
or for the demo Android app
```bash
adb pull /sdcard/Android/data/com.sensory.speech.snsr.demo.snsrdebug/files/logs
```


Either way, you'll end up with one or more `.snsrlog` files on your PC, ready for [Section 7](#7-extract-verify-and-compare-in-simulation).

---

## 6. PC Bench-Testing Alternative: Command-Line Model Swap

### 6.1 Building and Running a Debug-Wrapped Model

For a PC-attached microphone/bench setup, or a quick one-off test where you don't want to touch application code at all, you can build a debug-wrapped `.snsr` file with `snsr-edit` and run it directly with `snsr-eval` on a PC instead of going through an app:

```bash
cd ~/Sensory/TrulyHandsfreeSDK/7.8.0

bin/snsr-edit -o vg-debug.snsr \
    -t model/tpl-spot-debug-1.5.1.snsr \
    -f 0 model/spot-voicegenie-enUS-6.5.1-m.snsr \
    -s debug-log-file=debug-%#.log \
    -s include-model=1

bin/snsr-eval -v -t vg-debug.snsr
```

- `-t` — the debug template
- `-f 0` — loads the spotter model to debug into slot 0
- `-s debug-log-file=...` — the `%#` substitution timestamps each run's log so repeated tests don't collide (see [9.2](#92-timestamped-non-colliding-log-filenames))
- `-s include-model=1` — embeds the model in the log (this is the default; shown explicitly here for clarity)

`snsr-eval` with no WAV file given reads live audio from the default capture device, which lets you exercise this on a bench with a real microphone. `^C` to stop:

```
  2925   3690 hello blue genie
  4995   5790 hello blue genie
  7920   8640 hello blue genie
^C
```

> **Note:** Interrupting `snsr-eval` with `^C` before it reaches a clean stream end is expected and produces a harmless "truncated" warning from `snsr-log-split` in the next step — the audio, event, and model data recorded up to that point are still valid and usable.

In most cases this can't capture the actual DUT's microphone, audio front-end, or app integration the way [Section 5](#5-capturing-debug-audio-from-an-app) can — it's a substitute for bench-testing a model in isolation, or for the rare case where the model can only be replaced by loading a different file (e.g. it's not compiled into firmware, but the application also can't be modified to add runtime debug wrapping). One notable exception: if the DUT hardware exposes an audio-out path — a headphone/line-out jack, a debug audio tap, etc. — that can be routed into the PC's line-in or microphone input, `snsr-eval`'s live-capture mode ends up listening to the DUT's actual microphone and audio front-end after all, just relayed over an analog cable instead of through the DUT's own application code. In that setup, this alternative *does* give you a real DUT-audio debug log, without touching the app at all.

### 6.2 Compiling the Debug Template into Firmware

If the model *is* compiled directly into the firmware image (via `snsr-edit -c`, as covered in [Creating and Using Enrolled Models §6.1](enrolled-models.md#61-compiling-a-model-into-the-application-snsr-edit)), the file-swap shown in [6.1](#61-building-and-running-a-debug-wrapped-model) isn't available as-is — but the same debug-wrapped model built for the bench test can itself be compiled into a C array and temporarily substituted into the firmware image, giving you a debug build without a permanent code change:

```bash
bin/snsr-edit -c vg-debug.c -t vg-debug.snsr
```

This produces `vg-debug.c`, a byte array in the same form as the production model's compiled-in data — see the SDK's `spot-data.c` sample (`sample/c/`) for how that data is passed to the SDK. Swap it in wherever the firmware normally embeds the production model's array, rebuild, flash, and capture; then revert to the real compiled-in model once you're done. Treat this as a temporary diagnostic build only, never something to ship — Section 5 remains the better option whenever the application itself can be modified instead of the firmware.

---

## 7. Extract, Verify, and Compare in Simulation

Everything from here on is the same regardless of whether the `.log` came from an app on the DUT (Section 5) or a PC bench test (Section 6).

### 7.1 Extract Audio and Events with snsr-log-split

```bash
bin/snsr-log-split -vv vg-debug-1699999999999.log
```

```
Writing to './'
Processing vg-debug-1699999999999.log
  -> audio ./vg-debug-1699999999999.wav
  -> event ./vg-debug-1699999999999.txt
  -> model ./vg-debug-1699999999999.snsr
Error: Input file "vg-debug-1699999999999.log" is truncated.
Processed 1273 items.
```

This produces three files:

| File | Contents |
|---|---|
| `<name>.wav` | The captured audio |
| `<name>.txt` | The event/result timeline from the test |
| `<name>.snsr` | The model used in the test (only present if `include-model=1`) |

Use `-d directory` to write the output files somewhere other than the current directory (the directory must already exist), and stack `-v` up to three times for increasing verbosity.

### 7.2 Verify Captured Audio Quality with audio-check

Before spending time comparing recognition results, confirm the capture itself is clean — a bad microphone path or clipped gain stage on the DUT can masquerade as a model accuracy problem. `audio-check` runs checks for problems like all-zero/flat runs and clipping, and estimates signal-to-noise ratio, on a 16 kHz mono WAV file:

```bash
bin/audio-check vg-debug-1699999999999.wav
```

The tool reports any clipping or flat/silent runs it finds, along with an SNR estimate in dBA. If it flags clipping, a flat signal, or a poor SNR estimate, suspect the DUT's audio path (gain staging, AEC, microphone placement/hardware) before suspecting the model — a genuinely bad recording will produce worse results in simulation too, independent of any model issue.

`audio-check`'s automated checks are a fast first pass, not a substitute for actually looking at (and listening to) the waveform. Open the extracted `.wav` in an editor like Adobe Audition or Audacity and check for:

- **Skips or glitches** — discontinuities in the waveform that suggest dropped or duplicated audio buffers somewhere in the DUT's capture path.
- **Saturation or attenuation** — clipped peaks or an unusually weak signal, beyond what `audio-check`'s automated clipping/SNR checks already catch.
- **Frequency-domain issues** — use the editor's spectrogram view to judge overall fidelity: missing or rolled-off frequency bands (a narrowband mic, overly aggressive filtering) and noise concentrated in specific bands (mains hum, AEC/NS artifacts) both point to the DUT's audio front end rather than the model.

### 7.3 Reproduce and Compare in Simulation

Run the extracted audio back through the **original, non-debug** model (not the debug-wrapped one):

```bash
bin/snsr-eval -v -t model/spot-voicegenie-enUS-6.5.1-m.snsr vg-debug-1699999999999.wav
```

> **Optional — recovering the model from the log instead of the original file:** if `include-model=1` was used ([6.1](#61-building-and-running-a-debug-wrapped-model)), the `<name>.snsr` extracted in [7.1](#71-extract-audio-and-events-with-snsr-log-split) is the debug-wrapped model, with the original spotter model sitting in its slot 0 — the same nesting described in [5.4](#54-slot-addressing-changes-when-wrapped). If you don't have the original standalone model file on hand (or aren't certain it's the exact version that was running), pull it back out with `snsr-edit` instead of tracking it down separately:
>
> ```bash
> bin/snsr-edit -v -t ./vg-debug-1699999999999.snsr -e 0 ./vg-1699999999999.snsr
> ```
>
> Use the resulting `vg-1699999999999.snsr` in place of `model/spot-voicegenie-enUS-6.5.1-m.snsr` above — it's the same original, non-debug model, recovered directly from the log.

Compare these results against what was observed over-the-air. Recognition in THF/TNL is deterministic: given the exact audio the model saw, `snsr-eval` is *expected* to reproduce the exact same wake word detections, not merely similar ones.

- **Results match** — the expected outcome, and confirms recognition itself isn't the problem. If the original report still doesn't add up, look elsewhere in the customer's system (application logic, action handling, downstream NLU, etc.) rather than at the model.
- **Results differ** — because recognition is deterministic against a given audio stream, this is *not* expected, and it does not point at a model accuracy gap. It means either the captured audio isn't a faithful copy of what the model actually consumed on the DUT, or the DUT's application code is mishandling/misreporting the model's results — both are integration bugs, not recognition issues. Rule out a bad capture first with `audio-check` and a waveform editor (7.2); if the audio checks out clean, focus the investigation on the DUT's audio-capture and result-handling code rather than on the model.

### 7.4 Requesting a Free Log Review from Sensory

Sensory offers a free review of captured debug logs for any commercial product built on Sensory's SDK — a useful option if the checks in 7.1–7.3 don't settle the question, or you'd simply like an independent second opinion before escalating further. Contact your Sensory FAE and share the raw `.snsrlog`/`.log` file — captured with `include-model=1` if possible, so Sensory has the exact model version alongside the audio — along with a description of what was expected versus what was actually observed on the DUT.

---

## 8. Lightweight Alternative: In-Memory Audio Capture

The debug template captures continuously to disk, which is the right tool for open-ended false-accept/false-reject soak testing (Section 9.1 covers the storage cost of that). For a lighter-weight capture that avoids writing a growing log file at all, use the SDK's built-in in-memory audio ring buffer instead — useful when you only need to inspect the audio immediately surrounding individual detections, on a device where storage is at a premium.

Enable buffering by setting `audio-stream-size` to the number of samples you want retained (0 disables buffering, and is the default):

```c
snsrSetInt(s, SNSR_AUDIO_STREAM_SIZE, 16000 * 5); /* ~5 seconds at 16 kHz */
```

Then, from a `^result` handler, retrieve the buffered audio and the range it currently covers:

```c
SnsrStream audio;
snsrGetStream(s, SNSR_AUDIO_STREAM, &audio);
/* audio-stream-first / audio-stream-last mark the oldest/newest sample
 * indexes currently held in the buffer; begin-sample / end-sample on the
 * result itself mark the matched utterance within that range */
```

**Trade-off:** this only ever holds the most recent `audio-stream-size` samples — it's a rolling window, not a continuous recording. It's well suited to capturing "what did the model hear right around this specific detection," but it cannot help diagnose problems that only show up over a long idle-listening session (e.g. an intermittent false accept once every few hours), which is exactly what the continuous `debug-log-file` capture in Sections 5–7 is for. Use `audio-stream` for point-in-time spot checks, and `tpl-spot-debug` for open-ended soak testing.

---

## 9. Design Guidance

### 9.1 Storage and Memory Budgeting

Data is written to the debug log at roughly the raw PCM rate: 16 kHz, 16-bit, mono audio is 32,000 bytes/second (≈31.25 KB/s), and the `.snsrlog` container's event/timestamp overhead brings this to **approximately 35 KB/second** in practice.

| Test Duration | Approximate Log Size |
|---|---|
| 1 hour | ~125 MB |
| 8 hours | ~1 GB |
| 24 hours | ~3 GB |

A statistically meaningful false-accept test in particular may require many hours of continuous idle-listening capture, so confirm the DUT has sufficient free storage before starting a long soak test — this is the same kind of budgeting exercise as the peak/average memory figures in [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md), just applied to disk rather than heap.

If storage is tight, `include-model=0` (9.3) trims a fixed amount per log (the size of the wrapped model file, paid once per log rather than per second), and the [in-memory alternative](#8-lightweight-alternative-in-memory-audio-capture) avoids continuous disk writes entirely at the cost of only covering a rolling window.

### 9.2 Timestamped, Non-Colliding Log Filenames

Because `debug-log-file` is a single, exact path, reusing a static filename across multiple test runs risks each run overwriting the previous one's capture. Use the `%@` or `%#` substitution so every run gets a unique name automatically — the sample does this with `System.currentTimeMillis()` baked into the filename itself (5.5), which is equivalent to letting the SDK do it for you:

```c
snsrSetString(session, SNSR_DEBUG_LOG_FILE, "debug-%#.log");
```

`%@` produces a human-readable UTC timestamp (`year-month-day_hour-minute-second.milliseconds`); `%#` produces milliseconds since the Unix epoch. They're mutually exclusive — pick one per filename.

### 9.3 The include-model Trade-off

`include-model` defaults to `1`, meaning every debug log carries a full copy of the wrapped model file. This makes each log fully self-contained — `snsr-log-split` can hand you back the exact model that was running, which is valuable when you don't otherwise have strict version control over what was deployed to a given DUT. Set `include-model=0` when you're confident you already have the exact model file on hand (or storage is tight): the log will be smaller by roughly the model's file size, but `snsr-log-split` won't be able to extract a `.snsr` file from it — you'll need to supply the original model separately for simulation playback ([7.3](#73-reproduce-and-compare-in-simulation)). This is also a privacy/IP lever, not just a storage one — see [Section 2](#2-security-and-privacy).

### 9.4 Choosing a Capture Approach

| Situation | Recommended approach |
|---|---|
| Capturing from a real DUT — the normal case | [App-based capture](#5-capturing-debug-audio-from-an-app) |
| Model is compiled into firmware and you can still modify the app | [App-based capture](#5-capturing-debug-audio-from-an-app) |
| PC bench test, or a device that can't be modified but can load a different model file | [PC bench-testing alternative](#6-pc-bench-testing-alternative-command-line-model-swap) |
| Long idle-listening false-accept/false-reject soak test | Continuous `debug-log-file` capture, budget storage per 9.1 |
| Storage-constrained device, only need audio around individual detections | [In-memory `audio-stream` capture](#8-lightweight-alternative-in-memory-audio-capture) |
| Suspect the capture itself before trusting a comparison | Run `audio-check` first (7.2) |

---

## 10. Common Pitfalls

- Assuming a results mismatch between over-the-air and simulation playback is a model accuracy bug — recognition is deterministic against a given audio stream, so a real mismatch means a bad capture (rule out with `audio-check` and a waveform editor, 7.2) or a DUT-side integration bug, not the model (7.3).
- Reusing a static `debug-log-file` name across multiple runs and unknowingly overwriting a previous capture — use `%@`/`%#` or a timestamped name (9.2).
- Replaying the **debug-wrapped** model instead of the original in the simulation comparison step — this re-captures another log instead of giving you a clean comparison.
- Hardcoding a slot-qualified event or setting address (e.g. `"0.^result"`) and being surprised it stops firing once the model is wrapped for debugging — it needs the extra `Snsr.SLOT_0` prefix (5.4).
- Using `getFilesDir()` for the debug log on a non-rooted, non-debuggable production device, then being unable to retrieve it without `run-as` — use `getExternalFilesDir()` instead when field retrieval needs to be simple (5.5).
- Shipping debug capture enabled by default, or capturing field audio without user awareness — see [Section 2](#2-security-and-privacy).
- Starting a multi-hour soak test without checking free storage on the DUT first (9.1).
- Forgetting that `audio-stream` (Section 8) only holds a rolling window — it will not have the audio you need if you go looking for it after the buffer has wrapped around.

---

## 11. Quick Reference — Command Summary

```bash
# App-based capture (Section 5): build/run the snsr-debug sample (or your own
# app following its pattern) with debug capture enabled, then pull the log:

# — app-internal storage (getFilesDir(), debuggable build required):
adb -d shell "run-as <package> tar -C /data/user/0/<package>/files -cf - logs" | tar -xf -

# — app-external storage (getExternalFilesDir(), no debuggable build needed):
adb pull /sdcard/Android/data/<package>/files/logs

# PC bench-testing alternative (Section 6):
bin/snsr-edit -o vg-debug.snsr \
    -t model/tpl-spot-debug-1.5.1.snsr \
    -f 0 model/spot-voicegenie-enUS-6.5.1-m.snsr \
    -s debug-log-file=voicegenie-debug-%#.snsrlog \
    -s include-model=1
bin/snsr-eval -v -t vg-debug.snsr

# Extract, verify, and compare (Section 7) — same for either capture path:
bin/snsr-log-split -vv voicegenie-debug-<timestamp>.snsrlog
bin/audio-check voicegenie-debug-<timestamp>.wav
bin/snsr-eval -v -t model/spot-voicegenie-enUS-6.5.1-m.snsr \
    voicegenie-debug-<timestamp>.wav
```

---

## 12. References

- TNL SDK 7.8 Docs: https://doc.sensory.com/tnl/7.8/
- `tpl-spot-debug` template: https://doc.sensory.com/tnl/7.8/models/tpl/tpl-spot-debug/
- `snsr-log-split` reference: https://doc.sensory.com/tnl/7.8/tools/snsr-log-split/
- `audio-check` reference: https://doc.sensory.com/tnl/7.8/tools/audio-check/
- `snsr-edit` reference: https://doc.sensory.com/tnl/7.8/tools/snsr-edit/
- `snsr-eval` reference: https://doc.sensory.com/tnl/7.8/tools/snsr-eval/
- Configuration setting keys (`debug-log-file`, `include-model`): https://doc.sensory.com/tnl/7.8/api/setting-keys/configuration/
- Result setting keys (`audio-stream` and related): https://doc.sensory.com/tnl/7.8/api/setting-keys/results/
- `snsr-debug` sample: `sample/android/snsr-debug` in the SDK distribution
- [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md) — for the broader storage/memory budgeting methodology this guide's Section 9.1 borrows from
- [Creating and Using Enrolled Models](enrolled-models.md) — for compiling a model into firmware (§6.1, relevant to Section 6) and for the security handling this guide's Section 2 mirrors (§7.3)
