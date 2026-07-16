# Creating and Using Enrolled Models with Sensory TrulyHandsfree/TrulyNatural

**Product:** TrulyHandsfree (THF) and TrulyNatural (TNL) SDKs
**Models:** Enrollment task models (`udt-*.snsr`, `eft-*.snsr`) and the enrolled models they produce
**Version:** THF / TNL 6+, 7+
**Document Version:** 1.0.0
**Audience:** External developers and integration partners
**Status:** Released under NDA — Do not distribute

*Copyright © 2026 Sensory Inc. All rights reserved. This document is confidential and proprietary to Sensory Inc. It may not be reproduced, distributed, or disclosed to any third party without prior written permission from Sensory Inc.*

---

## Table of Contents

1. [Overview](#1-overview)
   - [What Is an Enrolled Model?](#what-is-an-enrolled-model)
   - [Enroller Models vs. Enrolled Models](#enroller-models-vs-enrolled-models)
   - [Why Enroll a Model?](#why-enroll-a-model)
2. [Prerequisites](#2-prerequisites)
3. [Enrollment Types](#3-enrollment-types)
   - [3.1 User-Defined Enrollment](#31-user-defined-enrollment)
   - [3.2 Enrolled-Fixed Enrollment](#32-enrolled-fixed-enrollment)
   - [3.3 Simulated Enrolled-Fixed Enrollment (SEFW)](#33-simulated-enrolled-fixed-enrollment-sefw)
4. [Enrolling a Model](#4-enrolling-a-model)
   - [4.1 The Two-Step Enrollment Process](#41-the-two-step-enrollment-process)
   - [4.2 Recording Guidelines](#42-recording-guidelines)
   - [4.3 Enrolling from the Command Line with spot-enroll](#43-enrolling-from-the-command-line-with-spot-enroll)
   - [4.4 Interactive Enrollment with live-enroll](#44-interactive-enrollment-with-live-enroll)
   - [4.5 Context Models: Adapting, Adding, and Deleting Users](#45-context-models-adapting-adding-and-deleting-users)
   - [4.6 Enrolling Programmatically via the API](#46-enrolling-programmatically-via-the-api)
5. [Using an Enrolled Model](#5-using-an-enrolled-model)
   - [5.1 Loading and Running an Enrolled Model](#51-loading-and-running-an-enrolled-model)
   - [5.2 Reading Results](#52-reading-results)
   - [5.3 Combining Fixed and Enrolled Models](#53-combining-fixed-and-enrolled-models)
   - [5.4 Biometric Scoring: THF/TNL vs. THF-Micro](#54-biometric-scoring-thftnl-vs-thf-micro)
   - [5.5 Recognition Sensitivity: Operating Points and score-offset](#55-recognition-sensitivity-operating-points-and-score-offset)
6. [Converting to a Deeply Embedded Model](#6-converting-to-a-deeply-embedded-model)
   - [6.1 Compiling a Model into the Application (snsr-edit)](#61-compiling-a-model-into-the-application-snsr-edit)
   - [6.2 Converting an Enrolled Model at Runtime (spot-convert)](#62-converting-an-enrolled-model-at-runtime-spot-convert)
7. [Design Guidelines](#7-design-guidelines)
   - [7.1 Recording Quality and Environment](#71-recording-quality-and-environment)
   - [7.2 Choosing an Enrollment Type](#72-choosing-an-enrollment-type)
   - [7.3 Security Considerations](#73-security-considerations)
   - [7.4 CPU and Memory Budgeting](#74-cpu-and-memory-budgeting)
   - [7.5 Testing an Enrolled Model](#75-testing-an-enrolled-model)
   - [7.6 Common Pitfalls](#76-common-pitfalls)
8. [Quick Reference — Command Summary](#8-quick-reference--command-summary)
9. [References](#9-references)

---

## 1. Overview

This guide explains how to create and use **enrolled models** with the Sensory TrulyHandsfree (THF) and TrulyNatural (TNL) SDKs — models that are trained to a specific person's voice, either to enable custom user-defined wake words and commands or to perform voice biometric verification and/or speaker identification (ID).

### What Is an Enrolled Model?

An enrolled model is a wake word or command model that has been trained (**enrolled**) from a small number of recordings — usually four — by a specific speaker, rather than shipped pre-trained to work for the general population, like a fixed wake word model. Because it is trained on one speaker's voice, an enrolled model works best — and, for biometric or speaker ID use cases, is intended to work *only* — for the person who enrolled it.

Enrolled models are used for three related purposes:

- **Custom wake words and commands** — letting an end user define their own trigger phrase rather than being limited to a phrase fixed at build time.
- **Speaker identification** — identifying which of a number of enrolled users is speaking, without necessarily enforcing biometric security.
- **Voice biometrics (speaker verification)** — confirming that the speaker is who they claim to be, in addition to recognizing the phrase.

### Enroller Models vs. Enrolled Models

It's important not to confuse these two distinct model types:

| Model | Role |
|---|---|
| **Enroller model** | A task model (e.g. `udt-enUS-5.1.1.9.snsr`) distributed by Sensory. It is not itself a recognizer — it is the tool that converts a speaker's recordings into a new enrolled model. Enroller models are typically 10–25 MB. |
| **Enrolled model** | The output of the enrollment process (e.g. `user1.snsr`). It is a small, speaker-specific recognizer, comparable in size and runtime cost to a fixed wake word model. |

### Why Enroll a Model?

Fixed models work identically for every speaker and are set at build time. Enrollment trades that generality for personalization: an end user can define their own wake phrase on-device, or an application can restrict recognition — and optionally verify identity — to a specific enrolled speaker. This is a two-step, entirely on-device process; Sensory does not offer a hosted/cloud enrollment service for THF, nor are recordings uploaded to the cloud.

---

## 2. Prerequisites

- TrulyHandsfree or TrulyNatural SDK installed, with the following tools accessible on your `PATH`:
  - `spot-enroll` — offline enrollment from pre-recorded audio
  - `live-enroll` — interactive enrollment from a live microphone
  - `spot-convert` — converts an enrolled model to deeply embedded format; only needed if targeting a deeply embedded platform (see [6.2](#62-converting-an-enrolled-model-at-runtime-spot-convert))
- An **enroller task model**, distributed with the SDK. The THF SDK ships three:

  | Filename | Enrollment type | Notes |
  |---|---|---|
  | `eft-hbg-enUS-23.0.0.9.snsr` | Enrolled-fixed (EFW) — see [3.2](#32-enrolled-fixed-enrollment) | No operating points — tune with `score-offset` (see [5.5](#55-recognition-sensitivity-operating-points-and-score-offset)) |
  | `udt-enUS-5.1.1.9.snsr` | User-defined (UDW) — see [3.1](#31-user-defined-enrollment) | No operating points — tune with `score-offset` (see [5.5](#55-recognition-sensitivity-operating-points-and-score-offset)) |
  | `udt-universal-3.67.1.0.snsr` | User-defined (UDW) — see [3.1](#31-user-defined-enrollment) | Supports operating points (see [5.5](#55-recognition-sensitivity-operating-points-and-score-offset)) |

  — or, for a simulated enrolled-fixed (SEFW) model, a VoiceHub wake word project and Sensory FAE assistance to build it (see [3.3](#33-simulated-enrolled-fixed-enrollment-sefw)).
- Audio recordings of the target speaker, if enrolling offline: 16-bit PCM WAV, recorded in a quiet environment (see [4.2 Recording Guidelines](#42-recording-guidelines)).

> **Note:** Enrollment is on-device. Recordings and the resulting enrolled model never need to leave the local system, though some deployments do transfer the *recordings* to a more powerful machine to run the enroller there before shipping the resulting enrolled model back to the device — see [7.4 CPU and Memory Budgeting](#74-cpu-and-memory-budgeting).

---

## 3. Enrollment Types

Sensory supports three enrollment types: two selected by which enroller task model you use — **user-defined (UDW)** and **enrolled-fixed (EFW)** — plus a hybrid, **simulated enrolled-fixed (SEFW)**, built by combining the two.

> **Note — terminology:** Sensory historically called wake words "triggers," so you may still see **UDT** (User-Defined Trigger) used interchangeably with **UDW** (User-Defined Wake word), and **EFT** (Enrolled-Fixed Trigger) interchangeably with **EFW** (Enrolled-Fixed Wake word). This guide uses **UDW** and **EFW** throughout, since "wake word" is Sensory's current terminology — the one exception is enroller task model filenames, which still use the older `udt-`/`eft-` prefixes (e.g. `udt-enUS-5.1.1.9.snsr`).

### 3.1 User-Defined Enrollment

In user-defined enrollment, the target phrase is **not known in advance** — the speaker may enroll any phrase of three or more syllables. Because the phrase is unknown, a voice activity detector (VAD) is used to find the start and end of speech in each recording. Speech should be recorded in relative quiet, since background noise cannot be distinguished from the target phrase ahead of time.

### 3.2 Enrolled-Fixed Enrollment

In enrolled-fixed enrollment, the target phrase is **known in advance** and hard-coded into the enroller model — only that phrase can be enrolled; anything else is ignored. Newer enrolled-fixed enroller models internally combine a fixed wake word phrase spotter with the enroller to detect the start and end of the utterance, rather than relying solely on a VAD. This makes enrolled-fixed enrollment inherently more noise-robust, giving better endpointing and a more accurate enrolled model — but it is still good practice to minimize background noise during recording.

> **Design guidance:** Enrolled-fixed technology is **not recommended for critical system voice biometrics**. Because the target phrase is known in advance, an attacker knows exactly which phrase to attempt to spoof. Use user-defined enrollment for biometric use cases.

### 3.3 Simulated Enrolled-Fixed Enrollment (SEFW)

A **Simulated EFW (SEFW)** is a hybrid: from the end user's perspective it behaves like enrolled-fixed enrollment — only one specific phrase can ever be enrolled — but it's built on top of user-defined (UDW) enrollment technology rather than a purpose-built enrolled-fixed enroller model.

**How it's built:**

1. Create a wake word project for the target phrase in **VoiceHub**, Sensory's model-authoring portal.
2. Download the resulting fixed wake word (FW) model from that project and combine it with a UDW enroller model. The result is technically still a UDW enroller, but restricted to only accept recordings of the exact target phrase defined in the VoiceHub job.

> **Note:** As of this writing, combining a FW model with a UDW enroller to produce a SEFW model can only be done by a Sensory FAE — it isn't yet a self-service step for developers, though Sensory is working to remove this limitation. It's an option where a development or support agreement is in place; ask your FAE to build the SEFW model for you.

**Without FAE involvement**, you can approximate the same restriction two ways:

- **UI-level:** Enroll normally with a standard UDW enroller model, and instruct the user (in your application's UI) to say the specific target phrase during enrollment. Nothing at the model level prevents a different phrase from being enrolled — the restriction is enforced only by what you ask the user to say.
- **Application-code level:** Run fixed wake word (FW) recognition on the incoming audio first, and only forward audio to the UDW enroller when the FW model actually fires on the target phrase. This restricts what gets enrolled without requiring a combined SEFW model.

Once built (by either path), a SEFW enroller model is used exactly like any other UDW enroller model — the same `spot-enroll`/`live-enroll` workflow in [Section 4](#4-enrolling-a-model) applies unchanged.

---

## 4. Enrolling a Model

### 4.1 The Two-Step Enrollment Process

Enrollment always happens in two steps:

1. **Recording** — the speaker repeats the target phrase some number of times (`req-enroll`, typically 3–5). Each recording is automatically checked to confirm it's usable and captures the correct phrase; recordings that fail this check are rejected (and, in interactive mode, re-recorded).
2. **Enrolling** — the enroller model converts the accepted recordings into an enrolled model. Depending on the enroller task, the resulting model may or may not include biometric (speaker verification) security.

It is possible to enroll more than one user, or more than one phrase, into a single enrolled model — there is no fixed limit on the number of enrolled users other than available system RAM (see [7.4](#74-cpu-and-memory-budgeting)).

### 4.2 Recording Guidelines

- **Keep it quiet.** Enrollment recordings must be taken with minimal background noise. If the signal-to-noise ratio is too low, the audio check will fail and the recording will be rejected.
- **Record some context.** For wake word enrollment, it's recommended that a few of the recordings include a word or two spoken immediately after the target phrase. This trains the model **with context** (distinct from a *context model*, described in [4.5](#45-context-models-adapting-adding-and-deleting-users)), which helps with **co-articulation** — where the end of the target phrase blends into the start of the following word. For example, in "*Voice Genie, email messages*," many speakers merge the trailing "e" in "Genie" with the leading "e" in "email"; training with that context helps the model generalize to real usage.
- **Enrollment isn't noise-free by design.** The enroller model itself injects noise during the enrollment process to produce a noise-robust enrolled model — this is separate from, and not a substitute for, recording in a quiet environment.

### 4.3 Enrolling from the Command Line with `spot-enroll`

`spot-enroll` performs offline enrollment from pre-recorded WAV files.

```
spot-enroll -t task [options] [+user1 file1 [-c] file2 ...] [+user2 ...]
```

| Flag | Description |
|---|---|
| `-t task` | Enroller task model filename (required) |
| `-o out` | Enrolled model output filename (default: `enrolled-sv.snsr`) |
| `-e enrolledfile` | Save an enrollment context file (see [4.5](#45-context-models-adapting-adding-and-deleting-users)) |
| `-a adaptedfile` | Save an adapted enrollment context file |
| `-c file` | Mark a recording as containing trailing context speech |
| `-s setting=value` | Override a task setting (e.g. `accuracy`, `req-enroll`, `delete-user`) |
| `-v [-v [-v]]` | Increase verbosity |

Enroll a single user from four recordings:

```bash
spot-enroll -t udt-enUS-5.1.1.9.snsr -e user1-context.snsr -o user1.snsr \
    +user1 user1-001.wav user1-002.wav user1-003.wav user1-004.wav
```

### 4.4 Interactive Enrollment with `live-enroll`

`live-enroll` performs the same job interactively from a live microphone, prompting the speaker to re-record any recording that fails the audio check.

```
live-enroll -t task [options] +user1 [+user2 ...] [file ...]
```

| Flag | Description |
|---|---|
| `-t task` | Enroller task model filename (required) |
| `-e enrollments` | Enrollment context output filename |
| `-o out` | Enrolled model output filename (default: `enrolled-sv.snsr`) |
| `-p prefix` | Save each recording as `<prefix>-<user>-{pass,fail}-<index>.wav` |
| `-s setting=value` | Override a task setting |
| `-v [-v [-v]]` | Increase verbosity |

```bash
live-enroll -t udt-enUS-5.1.1.9.snsr -o hey-sensory.snsr +user1
```

> **Note:** `live-enroll` runs in **interactive mode** by default — it re-records any failed attempt rather than discarding it. `spot-enroll` against pre-recorded files runs in **offline mode** — a failed recording is simply excluded, not re-requested.

### 4.5 Context Models: Adapting, Adding, and Deleting Users

Once enrollment finishes, the enrolled model itself cannot be modified further. However, enrollment can optionally save an intermediate **context model** (via `-e context.snsr`), which *can* be modified — users can be added or deleted — and then **adapted** to produce a new enrolled model.

Enroll a second user into an existing context, then delete the first user, then enroll two users at once in a single pass:

```bash
# Enroll user1
spot-enroll -t udt-enUS-5.1.1.9.snsr -e user1-context.snsr -o user1.snsr \
    +user1 user1-001.wav user1-002.wav user1-003.wav user1-004.wav

# Add user2 to the existing context
spot-enroll -t udt-enUS-5.1.1.9.snsr -t user1-context.snsr \
    -e user1&2-context.snsr -o user1&2.snsr \
    +user2 user2-001.wav user2-002.wav user2-003.wav user2-004.wav

# Delete user1, keeping only user2
spot-enroll -t udt-enUS-5.1.1.9.snsr -t user1&2-context.snsr \
    -s delete-user=user1 -o user2.snsr

# Enroll two users in a single pass (no prior context needed)
spot-enroll -t udt-universal-3.66.1.9.snsr \
    -e user1+2-context.snsr -o user1+2.snsr \
    +user1 user11.wav user12.wav user13.wav user14.wav \
    +user2 user21.wav user22.wav user23.wav user24.wav
```

There is no limit on the number of users that can be enrolled at once.

> **Important — security note:** A context model can optionally retain the original enrollment recordings (`-s save-enroll-audio=1`), which is useful for replacing a questionable recording without re-recording everyone. For voice biometric deployments, treat this setting with care — retained raw voice recordings are sensitive data. See [7.3 Security Considerations](#73-security-considerations).

### 4.6 Enrolling Programmatically via the API

The `spot-enroll` and `live-enroll` tools are thin wrappers around the same SDK inference API used everywhere else in THF/TNL, built around a session handle (`SnsrSession`) that you create, configure, feed audio, and read results/state from. The table below summarizes the relevant calls and settings; consult the **Inference & I/O** and **Setting Keys** sections of the API reference at `doc.sensory.com` and the `snsr.h` header in your installed SDK for exact prototypes, since these can shift slightly between SDK versions.

#### 4.6.1 Core Session Functions Used During Enrollment

| Function | Purpose |
|---|---|
| `snsrNew` | Create a new session handle |
| `snsrLoad` | Load the enroller task model (or a saved context model) into the session |
| `snsrSet` / `snsrSetString` / `snsrSetInt` / `snsrSetDouble` | Apply enrollment settings (`user`, `req-enroll`, `accuracy`, …) |
| `snsrSetHandler` | Register callbacks for enrollment events (`^pass`, `^fail`, `^enrolled`, …) |
| `snsrPush` / `snsrRun` | Feed audio into the session, incrementally or from an attached stream |
| `snsrSave` | Serialize the resulting enrolled model (or context model) to a file |
| `snsrRC` / `snsrRCMessage` | Check and describe the session's error state |
| `snsrRelease` | Release the session handle |

#### 4.6.2 Enrollment Settings Keys

| Key | Type | Description |
|---|---|---|
| `user` | string | Tag for the current enrollment — a unique alphanumeric identifier, no spaces. Use `user/phrase` (one `/`) to enroll multiple phrases per user. |
| `req-enroll` | int | Required number of recordings per enrollment — adaptation will not take place until at least this many recordings have been accepted. In interactive mode the user is prompted to repeat the phrase until the number is reached. |
| `accuracy` | double, 0.0–1.0 | Trades enrollment speed for enrolled-model accuracy; higher is more accurate but slower to enroll. Default `1.0`. |
| `ctx-enroll` | int | Recommended number of recordings that should include trailing context speech (see [4.2](#42-recording-guidelines)). |
| `interactive` | int | `0` processes the stream to completion (offline mode); nonzero enables interactive re-recording of failed attempts. |
| `enrollment-task-index` | int | Selects which sub-task receives recordings, for multi-task enrollment models. Default `0`. |
| `delete-user` | string | Removes the named user from a loaded context model. |
| `save-enroll-audio` | int | `1` retains raw enrollment recordings in a saved context model; `0` (default) discards them. See [7.3](#73-security-considerations). |

#### 4.6.3 Enrollment Event Callbacks

| Event | Fired when |
|---|---|
| `^pass` | A recording passes the audio check |
| `^fail` | A recording fails the audio check |
| `^next` | The session is ready for the next recording |
| `^pause` | A time-consuming processing step is about to start — use this to pause the input stream during interactive enrollment |
| `^progress` | Reports adaptation progress (`percent-done`) |
| `^resume` | A time-consuming processing step has completed — use this to restart an input stream that was stopped on `^pause` |
| `^enrolled` | Enrollment for a user/phrase completes |
| `^adapted` | A context model has been adapted into a new enrolled model |
| `^done` | The overall enrollment run completes |

#### 4.6.4 Enrollment Iterators

Accessed via `snsrForEach`:

| Iterator | Description | Exposed fields | Availability |
|---|---|---|---|
| `enrollment-iterator` | Iterate over all wake word enrollments for the current user | `audio-stream`, `audio-stream-first`, `audio-stream-last`, `begin-sample`, `end-sample`, `enrollment-id`, `user` | Any context — use to retrieve enrollment audio when `save-enroll-audio` is enabled |
| `reason-iterator` | Iterate over all reasons for a wake word enrollment failure | `reason`, `reason-guidance`, `reason-pass`, `reason-threshold`, `reason-value` | Only within the `^fail` event callback |
| `user-iterator` | Iterate over all enrolled users | `enrollment-count`, `user` | Any context — automatically sets the `user` setting for each iteration as you loop |

#### 4.6.5 Illustrative Enrollment Sequence

Function names and settings per the tables above — see the [`spot-enroll.c`](https://doc.sensory.com/tnl/7.8/api/sample/c/spot-enroll/) / [`live-enroll.c`](https://doc.sensory.com/tnl/7.8/api/sample/c/live-enroll/) / [`live_enroll.py`](https://doc.sensory.com/tnl/7.8/api/sample/python/live_enroll/#live_enrollpy) / [`enrollUDT.java`](https://doc.sensory.com/tnl/7.8/api/sample/java/enrollUDT/) samples included with your SDK for a complete, compilable example, including the exact stream-attachment calls, which are omitted here:

```c
SnsrSession s;
snsrNew(&s);
snsrLoad(s, /* stream on udt-enUS-5.1.1.9.snsr */);

snsrSetString(s, "user", "user1");
snsrSetInt(s, "req-enroll", 4);
snsrSetDouble(s, "accuracy", 1.0);

snsrSetHandler(s, "^pass", onPass);
snsrSetHandler(s, "^fail", onFail);
snsrSetHandler(s, "^enrolled", onEnrolled);

/* attach each recording in turn and process it — see the SDK sample
 * for the exact stream setup calls (audio I/O and attach syntax) */
for (each recording) {
    /* attach recording as the session's audio input */
    snsrRun(s);
}

snsrSave(s, /* format */, /* stream on user1.snsr */);

if (snsrRC(s) != SNSR_RC_OK) {
    /* handle error — snsrRCMessage(snsrRC(s)) */
}
snsrRelease(s);
```

---

## 5. Using an Enrolled Model

### 5.1 Loading and Running an Enrolled Model

At runtime, an enrolled model is used exactly like a fixed wake word or command model — it is a self-contained recognizer. Load it into a session, register a result handler, and run the session (in pull mode):

```c
SnsrSession s;
snsrNew(&s);
snsrLoad(s, /* stream on user1.snsr */);

snsrSetHandler(s, "^result", onResult);

/* attach live or streaming audio input — see your platform's SDK
 * sample (e.g. live audio capture samples) for the exact I/O calls */
snsrRun(s);
```

### 5.2 Reading Results

The recognizer raises a `^result` event when a final recognition hypothesis is available. Think of the settings you configure going in as the **request**, and the fields on this event as the **response**:

| Field | Description |
|---|---|
| `id` | Identifier of the matched phrase/user |
| `text` | Recognized phrase text |
| `score` / `confidence-score` | Recognition confidence |
| `sv-score` | Speaker verification score, present for enrolled models trained with biometric security. On THF/TNL, a `^result` firing already reflects the configured `sv-threshold` (see [5.4](#54-biometric-scoring-thftnl-vs-thf-micro)) |
| `begin-ms` / `end-ms`, `begin-sample` / `end-sample` | Timing of the matched utterance |
| `noise-energy` / `signal-energy` / `snr` | Audio quality metrics for the matched segment |
| `domain` | NLU domain, if applicable |
| `phone-iterator` / `phrase-iterator` / `word-iterator` | Iterators over sub-word recognition detail |

> **Design guidance:** On THF/TNL, don't re-implement threshold logic in application code — set `sv-threshold` on the session and let the SDK enforce it (see [5.4](#54-biometric-scoring-thftnl-vs-thf-micro)). THF-Micro works differently and requires an explicit check; see the same section.

### 5.3 Combining Fixed and Enrolled Models

It is possible to **concurrently combine** a fixed model (recognizes anyone) and an enrolled model (recognizes a specific speaker) — even for the same target phrase. In this configuration, the enrolled speaker is matched by both the fixed and enrolled models, while any other speaker is matched only by the fixed model. This is a common pattern for adding an optional "recognize me specifically" tier on top of baseline recognition that works for everyone.

### 5.4 Biometric Scoring: THF/TNL vs. THF-Micro

Recognizing a match with an enrolled model happens in two conceptually distinct steps:

1. **Recognition** — phrase spotting, exactly like a fixed wake word model. This step alone is already speaker-biased: because the underlying model was trained (enrolled) on recordings from one speaker, it will tend to recognize that speaker's voice better than anyone else's, independent of any biometric security. Recognition sensitivity is tunable — see [5.5](#55-recognition-sensitivity-operating-points-and-score-offset).
2. **Verification** — an additional speaker-verification score (`sv-score`/`svScore`) compared against a threshold (`sv-threshold`/`SvThreshold`), covered below.

> **Common mistake:** Setting the verification threshold to `0` disables step 2's security check, but it does **not** turn an enrolled model into a generic, "works for anyone" fixed wake word — step 1's recognition is already tuned to the enroller's voice and will continue to favor them. Enrolling multiple people, or driving the threshold to `0`, is not a substitute for a real fixed wake word model. To build a wake word that performs well across the general population, train it in **VoiceHub** (see [3.3](#33-simulated-enrolled-fixed-enrollment-sefw)) rather than through enrollment.

Speaker verification scoring (step 2 above) is enforced differently depending on which Sensory runtime is evaluating the enrolled model, and this matters for where you put your threshold-checking code.

| | THF / TNL | THF-Micro |
|---|---|---|
| **Threshold setting** | `sv-threshold`, set on the session before running | `SvThreshold`, a field of the recognizer's configuration struct (`t2siStruct`) |
| **Score range** | Floating point, `0.0`–`1.0` | Integer, `0`–`8192` (THF-Micro is integer-only — no floating point) |
| **Enforcement** | Internal. Results scoring below `sv-threshold` are filtered by the SDK and never raise `^result` — a fired event already means the speaker passed verification | None. THF-Micro always returns `svScore` on the `RecoResult` struct; comparing it against `SvThreshold` is the application's responsibility |

> **Important:** Because THF/TNL enforces `sv-threshold` internally, application code does **not** need to re-check `sv-score` on a `^result` event — that check has already happened. On THF-Micro, the opposite is true: your application **must** compare `svScore` to `SvThreshold` itself before treating a match as verified, or biometric security will silently be a no-op.

Confirm exact field names, struct layout, and default threshold behavior against your installed THF-Micro SDK version — see the full THF-Micro documentation at https://doc.sensory.com/thf-micro/latest/.

### 5.5 Recognition Sensitivity: Operating Points and score-offset

TNL enroller models come in two generations, which determine how you tune *recognition* sensitivity (step 1 in [5.4](#54-biometric-scoring-thftnl-vs-thf-micro)) for the resulting enrolled model:

- **Newer enroller models** support **operating points (OPs)** — a small selectable range built into the enrolled model itself (commonly 6–14 or 7–13, with 10 as the default), trading off recognition sensitivity. Example: `udt-universal-3.67.1.0.snsr`.
- **Older enroller models** have no concept of an operating point. Examples: `udt-enUS-5.1.1.9.snsr` and `eft-hbg-enUS-23.0.0.9.snsr`.

| If the enrolled model... | Tune recognition sensitivity with |
|---|---|
| Supports operating points | `operating-point` |
| Does not support operating points | `score-offset` |

`score-offset` defaults to `0` and effectively ranges about ±30. Higher values make recognition more *accepting* (looser matching, fewer false rejects); negative values make it more *rejecting* (stricter matching, fewer false accepts).

> **Note:** `sv-threshold` (step 2, biometric verification — see [5.4](#54-biometric-scoring-thftnl-vs-thf-micro)) behaves identically regardless of which generation of enroller model you're using. OP and `score-offset` only affect step 1 (recognition); they have no effect on verification.

---

## 6. Converting to a Deeply Embedded Model

There are two distinct ways to convert a `.snsr` model to run on a deeply embedded target (e.g. a standalone DSP running THF-Micro), and they serve different purposes.

### 6.1 Compiling a Model into the Application (snsr-edit)

Any model loaded from a file at runtime is held in heap memory. On memory-constrained embedded targets, it can be converted to a C array and linked directly into the executable's read-only data segment, removing it from the heap entirely — the same technique used for any pipeline model, described in detail in [Benchmarking RTF and Avg/Max Memory Usage §5.4](benchmarking-rtf-memory.md#54-reducing-heap-usage-by-embedding-the-model-in-code-space):

```bash
snsr-edit -c user1-model.c -t user1.snsr
```

Study the `spot-data.c` sample included with the SDK (`sample/c/`) for how to pass embedded model data to the SDK via the data API instead of a file path.

> **Note:** For **enrolled models specifically, this is uncommon.** Compiling an enrolled model into the application hard-codes that one person's enrollment into the firmware image at build time — the app will only ever recognize whoever was enrolled when the binary was compiled. That's rarely what you want, since enrollment normally happens per end user, on their own device. Reach for this only if you genuinely intend to ship a fixed, factory-enrolled voice.

### 6.2 Converting an Enrolled Model at Runtime (spot-convert)

The more common path for enrolled models is `spot-convert`, which converts a `.snsr` model to deeply embedded format directly, without compiling it into the application. This keeps enrollment a runtime, per-user operation — the conversion can be run as part of an on-device enrollment flow — rather than baking one person's voice into the firmware image.

```
spot-convert -t task [options] target
```

- `task` — the `.snsr` file to convert (typically the enrolled model)
- `target` — a 4- or 5-character deeply embedded target code identifying the destination platform/format

| Enrolled model built from... | Common target code |
|---|---|
| An older enroller model (e.g. `udt-enUS-5.1.1.9.snsr`) | `pc38` |
| A newer enroller model (e.g. `udt-universal-3.67.1.0.snsr`) | `pc62w` |

Target codes are platform- and SDK-version-specific; confirm the current code for your target against `doc.sensory.com` or with your Sensory FAE. See [Output Formats and DSP Platform Versions](https://doc.sensory.com/thf-micro/latest/VoiceHub%20Versions.html) for the current list.

This is particularly relevant for enrolled models: as noted in [7.4](#74-cpu-and-memory-budgeting), enrollment itself is comparatively expensive, but the *resulting* enrolled model is small and cheap to run — small enough that it's often converted this way and deployed to a low-power standalone DSP running THF-Micro, separate from the (typically more capable) hardware used to perform the enrollment itself.

---

## 7. Design Guidelines

### 7.1 Recording Quality and Environment

Enroll in a quiet environment. Outside noise that leaks into a recording gets trained into the enrolled model, which degrades accuracy for the enrolled speaker and, for biometric use cases, can weaken the security margin. This applies to both enrollment types, though enrolled-fixed enrollment is somewhat more noise-robust due to its built-in fixed-phrase endpointing (see [3.2](#32-enrolled-fixed-enrollment)).

### 7.2 Choosing an Enrollment Type

- Use **user-defined enrollment (UDW)** when the phrase should be user-chosen, or for voice biometrics — an unknown target phrase doesn't give an attacker a fixed target to attempt to spoof.
- Use **enrolled-fixed enrollment (EFW)** when the phrase is fixed at build time and you want personalization (e.g. a "recognize me specifically" tier per [5.3](#53-combining-fixed-and-enrolled-models)) without biometric security guarantees.
- Use **simulated enrolled-fixed enrollment (SEFW)** — see [3.3](#33-simulated-enrolled-fixed-enrollment-sefw) — when you want a specific, non-user-chosen phrase (e.g. a partner or product-defined trigger built in VoiceHub) but need UDW-style enrollment mechanics. Plan for FAE involvement (or the UI-level/application-level workarounds in 3.3) since model combination isn't currently self-service.

### 7.3 Security Considerations

If you enable `save-enroll-audio=1` on a context model to make it easier to replace a bad recording, remember that the context model now contains raw voice recordings of the enrolled speaker. Treat that file with the same care as any other biometric or PII data store — restrict access to it, and avoid enabling this setting by default in a production biometric deployment.

### 7.4 CPU and Memory Budgeting

Enrollment itself is resource-intensive: a typical enroller model is 10–25 MB, and the enrollment process can take significant time even on a fairly powerful platform (PC or high-end mobile device). Processing time grows linearly with the number of users enrolled simultaneously. On slower platforms, consider enrolling twice — once at low `accuracy` for a fast first pass, then again at high `accuracy` — rather than making the end user wait through a single slow high-accuracy pass.

Once enrollment is complete, however, the **enrolled model** used at recognition time is much smaller and costs no more MIPS or memory than a fixed wake word or command model of similar complexity — see [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md) for how to measure that runtime cost directly. This split — expensive to *create*, cheap to *run* — is why enrollment is often done on more capable hardware (or, per §6, exported and embedded) even when the enrolled model itself will run on a constrained target.

### 7.5 Testing an Enrolled Model

When validating an enrolled model, measure all three of the following, not just raw recognition accuracy:

- **False reject rate (FRR)** — how often the enrolled speaker, saying the correct phrase, is *not* recognized.
- **False accept rate (FAR)** — how often a phrase is recognized when it should have been rejected.
- **Imposter accept rate (IAR)** — how often a *different* speaker is incorrectly accepted as the enrolled speaker. This is the metric that matters most for biometric deployments and should drive your `sv-threshold` (THF/TNL) or `SvThreshold` (THF-Micro) setting — see [5.4](#54-biometric-scoring-thftnl-vs-thf-micro).

### 7.6 Common Pitfalls

- Using enrolled-fixed enrollment for a security-sensitive biometric use case, where the known target phrase gives an attacker a fixed target.
- Enrolling in a noisy room and then being surprised by poor real-world accuracy.
- Skipping context recordings for phrases prone to co-articulation, then seeing inconsistent endpointing in production.
- Leaving `save-enroll-audio=1` set in a shipped configuration, unintentionally persisting raw voice recordings.
- Porting a THF/TNL biometric integration to THF-Micro without adding an explicit `svScore`/`SvThreshold` check — THF-Micro doesn't enforce the threshold internally the way THF/TNL does, so the check silently becomes a no-op (see [5.4](#54-biometric-scoring-thftnl-vs-thf-micro)).
- Planning a product around a self-service SEFW workflow — building one currently requires a Sensory FAE (see [3.3](#33-simulated-enrolled-fixed-enrollment-sefw)); budget for that dependency or use the UI-level/application-level workaround instead.
- Trying to manufacture a generic, "works for anyone" wake word by enrolling many users and/or setting the verification threshold to `0` — recognition itself is speaker-biased from training, so this doesn't produce a real fixed wake word. Use VoiceHub instead (see [5.4](#54-biometric-scoring-thftnl-vs-thf-micro)).
- Reaching for `score-offset` on a model that actually supports operating points, or vice versa — check which generation of enroller model produced your enrolled model before choosing a sensitivity-tuning approach (see [5.5](#55-recognition-sensitivity-operating-points-and-score-offset)).
- Using `snsr-edit -c` to embed an enrolled model, unintentionally hard-coding a single person's voice into the shipped firmware — `spot-convert` is the right tool for a deeply embedded target that still needs per-user enrollment (see [6.2](#62-converting-an-enrolled-model-at-runtime-spot-convert)).

---

## 8. Quick Reference — Command Summary

```bash
# Enroll a single user offline from pre-recorded WAV files
spot-enroll -t udt-enUS-5.1.1.9.snsr -e user1-context.snsr -o user1.snsr \
    +user1 user1-001.wav user1-002.wav user1-003.wav user1-004.wav

# Enroll interactively from a live microphone
live-enroll -t udt-enUS-5.1.1.9.snsr -o hey-sensory.snsr +user1

# Add a second user to an existing context model
spot-enroll -t udt-enUS-5.1.1.9.snsr -t user1-context.snsr \
    -e user1&2-context.snsr -o user1&2.snsr \
    +user2 user2-001.wav user2-002.wav user2-003.wav user2-004.wav

# Delete a user from a context model, then re-adapt
spot-enroll -t udt-enUS-5.1.1.9.snsr -t user1&2-context.snsr \
    -s delete-user=user1 -o user2.snsr

# Enroll two users in a single pass
spot-enroll -t udt-universal-3.66.1.9.snsr \
    -e user1+2-context.snsr -o user1+2.snsr \
    +user1 user11.wav user12.wav user13.wav user14.wav \
    +user2 user21.wav user22.wav user23.wav user24.wav

# Compile an enrolled model into the application (hard-codes one user — see 6.1)
snsr-edit -c user1-model.c -t user1.snsr

# Convert an enrolled model to deeply embedded format at runtime (typical for enrolled models)
spot-convert -t user1.snsr pc38    # older enroller models (e.g. udt-enUS-5.1.1.9.snsr)
spot-convert -t user1.snsr pc62w   # newer enroller models
```

---

## 9. References

- TNL SDK 7.8 Docs: https://doc.sensory.com/tnl/7.8/
- `spot-enroll` reference: https://doc.sensory.com/tnl/7.8/tools/spot-enroll/
- `live-enroll` reference: https://doc.sensory.com/tnl/7.8/tools/live-enroll/
- `spot-convert` reference: https://doc.sensory.com/tnl/7.8/tools/spot-convert/
- Enrollment model types: https://doc.sensory.com/tnl/7.8/models/types/enroll/
- Inference & I/O API: https://doc.sensory.com/tnl/7.8/api/inference/
- Setting keys reference: https://doc.sensory.com/tnl/7.8/api/setting-keys/
- Runtime event keys: https://doc.sensory.com/tnl/7.8/api/setting-keys/events/
- Iterator keys: https://doc.sensory.com/tnl/7.8/api/setting-keys/iterators/#enrollment--adaptation
- THF-Micro Docs: https://doc.sensory.com/thf-micro/latest/
- [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md) — for measuring enrolled-model runtime cost and embedding models in code space
