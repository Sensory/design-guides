# Benchmarking Enrolled Wake Word Models (FRR, IAR, FAR)

**Product:** TrulyHandsfree (THF) and TrulyNatural (TNL) SDKs  
**Models:** Enrolled models produced by the enroller task models (`udt-*.snsr`, `eft-*.snsr`)  
**Version:** THF / TNL 7.8.0+ (examples run on TNL 7.9.0)  
**Document Version:** 1.2.0  
**Last Modified:** 2026-10-26  
**Audience:** Field Application Engineers, Customer Integration Engineers, external developers and integration partners  
**Status:** Released

*Copyright © 2026 Sensory Inc. All rights reserved.*

---

## Table of Contents

1. [Overview](#1-overview)
   - [1.1 The Three Metrics](#11-the-three-metrics)
   - [1.2 Why Enrolled Models Need Their Own Benchmark](#12-why-enrolled-models-need-their-own-benchmark)
   - [1.3 Simulate, Don't Test Live](#13-simulate-dont-test-live)
2. [Prerequisites](#2-prerequisites)
3. [Step 1 — Collect the Test Data](#3-step-1--collect-the-test-data)
   - [3.1 How Much Data Is Enough](#31-how-much-data-is-enough)
   - [3.2 Enrollment Recordings](#32-enrollment-recordings)
   - [3.3 Test Recordings](#33-test-recordings)
   - [3.4 Non-Target Audio for FAR](#34-non-target-audio-for-far)
   - [3.5 Real vs. Synthetic Speakers](#35-real-vs-synthetic-speakers)
   - [3.6 If You Must Record Over the Air](#36-if-you-must-record-over-the-air)
   - [3.7 Audio Requirements](#37-audio-requirements)
4. [Step 2 — Lay Out the Corpus](#4-step-2--lay-out-the-corpus)
5. [Step 3 — Add Noise Conditions](#5-step-3--add-noise-conditions)
6. [Step 4 — Enroll and Score Every Speaker](#6-step-4--enroll-and-score-every-speaker)
   - [6.1 How Scoring Works](#61-how-scoring-works)
   - [6.2 The Benchmark Script](#62-the-benchmark-script)
   - [6.3 Running It](#63-running-it)
   - [6.4 Reading the Output](#64-reading-the-output)
7. [Complete Walkthrough (Windows, TNL SDK 7.9.0)](#7-complete-walkthrough-windows-tnl-sdk-790)
8. [Step 5 — Interpreting Results](#8-step-5--interpreting-results)
   - [8.1 Choosing an sv-threshold](#81-choosing-an-sv-threshold)
   - [8.2 Recognition Sensitivity vs. Verification](#82-recognition-sensitivity-vs-verification)
   - [8.3 Confidence Intervals](#83-confidence-intervals)
   - [8.4 Report Every Speaker](#84-report-every-speaker)
   - [8.5 Example: Synthetic 55-Speaker Results](#85-example-synthetic-55-speaker-results)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Quick Reference — Command Summary](#10-quick-reference--command-summary)
11. [References](#11-references)

---

## 1. Overview

This guide explains how to measure the accuracy of **enrolled wake word models** — models trained from a specific speaker's own recordings with the Sensory THF/TNL enrollment tools (see [Creating and Using Enrolled Models](enrolled-models.md)). It covers what data to collect, how much of it you need, how to add noise conditions, and how to enroll and score every speaker in simulation with `spot-enroll`, `snsr-eval`, and a short Python script.

### 1.1 The Three Metrics

An enrolled model can fail in three distinct ways, and each needs its own test audio:

| Metric | Question it answers | Test audio | How it's computed |
|---|---|---|---|
| **False reject rate (FRR)** | How often is the enrolled speaker, saying the phrase, *not* recognized? | The enrolled speaker's own test recordings | Each speaker's model against **their own** test recordings |
| **Imposter accept rate (IAR)** | How often is a *different* speaker, saying the same phrase, accepted as the enrolled speaker? | Every other speaker's test recordings | Each speaker's model against **everyone else's** test recordings |
| **False accept rate (FAR)** | How often does the model fire when nobody said the phrase? | Long recordings of non-target audio — conversation, TV, room noise | Each speaker's model against the noise audio, reported as **false accepts per day** |

FRR and IAR are percentages of trials. FAR is a rate over time, so it's reported as false accepts per day (24 hours) of audio. Because every speaker's model listens to the same noise, the amount of audio tested is counted in **model-hours** (number of models × hours of noise), and FAR = false accepts ÷ model-hours × 24.

These trade off against each other. Raising `sv-threshold` lowers IAR and FAR but raises FRR. Benchmark at the threshold the enrolled model ships with, and change it only if the results show a reason to — see [8.1](#81-choosing-an-sv-threshold).

### 1.2 Why Enrolled Models Need Their Own Benchmark

A fixed wake word is a single model, so you benchmark it once against a pool of speakers. An enrolled model is different for every user: there is one model per speaker, built on-device from that speaker's own recordings. A meaningful benchmark therefore has to **enroll many speakers**, then test each resulting model separately. That's what makes enrolled testing harder than fixed wake word testing — not the scoring, but collecting enough speakers who each provide enrollment *and* test recordings.

### 1.3 Simulate, Don't Test Live

Speech recognition in the THF/TNL SDKs is **fully deterministic**: the exact same model and the exact same audio always produce the exact same result. That means a benchmark run in simulation on a PC — feeding recorded audio through `snsr-eval` — gives the same answer as running the same audio live on the device.

That makes live, over-the-air testing unnecessary, and at the scale a benchmark needs it isn't realistic either. Live testing runs in real time: playing 24 hours of noise to a device to measure FAR takes 24 hours, and that's for one enrolled model. Testing a 50-speaker benchmark one model at a time would take 50 days of playback, and you'd have to repeat it every time you change a threshold or setting. Simulation processes audio much faster than real time, so the same 24 hours of noise takes only a few minutes.

If you need the benchmark to reflect your device's microphone and audio front end, **capture** audio through that path once (see [How to Save Debug Audio and Data in THF/TNL](debug-audio-capture.md)) and reuse the recordings as simulation test data from then on.

---

## 2. Prerequisites

- THF or TNL SDK installed, with these tools in the SDK's `bin/` folder:
  - `spot-enroll` — offline enrollment from pre-recorded WAV files (see [Creating and Using Enrolled Models §4.3](enrolled-models.md#43-enrolling-from-the-command-line-with-spot-enroll))
  - `snsr-eval` — runs a model against audio files
- An **enroller task model** — the one your product will ship with. The examples here use `udt-universal-3.67.1.0.snsr` (user-defined, supports operating points), which ships in the SDK's `model/` folder. See [Creating and Using Enrolled Models §2](enrolled-models.md#2-prerequisites) for the full list.
- **Python 3.8+** for the benchmark script in [6.2](#62-the-benchmark-script). It uses only the standard library.
- Audio as **16-bit mono PCM WAV** at the model's sample rate (16 kHz for the shipped enroller models).
- Test audio — see [Step 1](#3-step-1--collect-the-test-data).

---

## 3. Step 1 — Collect the Test Data

### 3.1 How Much Data Is Enough

Small tests are the most common benchmarking mistake. Testing with a handful of people around the office and making product decisions from the result is not a meaningful benchmark — the numbers can swing wildly from one person to the next.

| Item | Minimum | Recommended |
|---|---|---|
| Speakers (each enrolls and records test audio) | 50 | 100 or more |
| Enrollment recordings per speaker | 4 | 4 (match what your product will ask for) |
| Test recordings per speaker | 10 | 20 |
| Non-target audio for FAR (every model hears all of it) | 15 hours, for quick informal tests | 150 hours, for statistically accurate results |

Because every speaker serves as an imposter for every other speaker, the number of imposter trials grows with the square of the speaker count: 50 speakers × 15 test recordings each gives 750 target trials but 36,750 imposter trials. More speakers is what improves IAR confidence the most.

### 3.2 Enrollment Recordings

Each speaker enrolls by saying the target phrase **4 times in a quiet environment** — a home or office with no one talking in the background and no fans or other noise running. These recordings train that speaker's enrolled model, exactly as an end user would on the device. Follow the recording guidelines in [Creating and Using Enrolled Models §4.2](enrolled-models.md#42-recording-guidelines), including recording some takes with trailing context speech if your product's enrollment flow does so.

### 3.3 Test Recordings

Each speaker also records **10–20 test recordings** of the same target phrase, separate from their enrollment recordings. Never reuse an enrollment recording as a test recording — the model has already been trained on it, so it will make FRR look better than it really is.

To make the test recordings representative of how the product will actually be used:

- Record them on a different day or in a different session from enrollment if possible, so they capture natural variation in the speaker's voice.
- Vary the speaker's distance and direction from the microphone the way real use will — for example, a laptop user may be at arm's length or across a desk. Sensory doesn't prescribe fixed distances or angles; match your product's use case, and record each distinct setup as its own condition (see [Step 2](#4-step-2--lay-out-the-corpus)) so you can compare them.

**Capture natural speech, not performed speech.** The test is only as realistic as the way people say the phrase. When someone uses a product, they're focused on what they want it to do, and they say the phrase naturally, as if talking to a person. Recording sessions easily produce something else:

- **Don't ask speakers to change their voice.** Instructions like "now say it softer", "faster", or "slower" produce speech no one would use with a real device. Let loudness and speed vary naturally — for example with distance, or between sessions.
- **Don't have them repeat the phrase over and over.** After a few repetitions in a row, people get bored and start saying it carelessly or with exaggerated variation. Mix the target phrase in among other prompts in random order, or have them say it followed by a different request each time, such as "Hey Sensory, what's the weather tomorrow?"
- **Ask for the phrase without pauses between the words.** A pause inside the phrase raises the error rate considerably, and it's not how people say it in use.

The same applies to enrollment: people tend to speak slowly and over-carefully when they know they're being recorded. Clear but natural speech gives the best enrolled model.
- Cover noise by either recording in noisy conditions or — more commonly and more reproducibly — recording clean and **adding noise digitally afterwards** at fixed signal-to-noise ratios (see [Step 3](#5-step-3--add-noise-conditions)).

### 3.4 Non-Target Audio for FAR

FAR audio is the easiest data to collect, since it doesn't need volunteers: any long recording of realistic sound that doesn't contain the target phrase works — conversations, TV and radio, music, room and household noise. The challenge is quantity. False accepts are rare events, so you need many hours of audio to see enough of them (or to be confident there are few — see [8.3](#when-you-see-zero-errors)):

- **15 hours** is the absolute minimum, and only for quick, informal tests where statistical significance isn't needed.
- **150 hours** is what you need for statistically accurate results. It's also the amount Sensory evaluates each enrolled model on before delivering a model.
- As a rule of thumb, evaluate every model on at least as many hours of noise as one false accept is allowed to take at your target rate. For example, a target of 0.05 FA/hour (1.2 FA/day, or one false accept per 20 hours) needs at least 20 hours of noise per model, and even that isn't enough to show the target is met with confidence.

Every speaker's model is scored on all of the noise, so these amounts are hours of noise audio, not totals divided across models. Reusing the same noise across many models is what makes FAR testing of enrolled models practical, but it doesn't replace hours: 2 hours of noise heard by 50 models isn't the same as 100 hours of different noise.

**Mix the noise types the product will hear.** FAR depends heavily on the kind of noise: it's usually higher on speech, TV, and radio than on car or fan noise. Use a mixture that reflects a typical day for the product, in roughly the proportions the product will hear each type. If you don't know the proportions, use equal amounts of each — for example office noise, babble (several people talking at once), car noise, household noise, single-speaker speech, and TV and radio. Speech, TV, and radio should be in the same language as the trigger phrase. Ideally, record the noise through the device's own microphone.

Screen the audio for the target phrase (and close soundalikes, if you want to keep them out) before using it — a genuine occurrence of the phrase in the noise would be scored as a false accept. Don't record FAR noise in the office where the product is being developed, where people say the phrase while they work on it.

### 3.5 Real vs. Synthetic Speakers

Sensory can generate **synthesized (voice-cloned) enrollment and test data** for a target phrase, which makes it practical to reach 50+ speakers quickly. Synthetic data is useful, but it shouldn't be the only data:

- A cloned-voice library may not be as diverse as real speakers. Similar-sounding synthetic voices make it easier for one "speaker" to pass as another, which can inflate IAR above what you'd see in the field. In Sensory's experience, tests with real speakers have shown IARs much closer to zero than synthetic tests of the same model.
- Some of the benchmark **must** use real recordings of actual people speaking. Use synthetic data to scale up and to compare configurations against each other; use real speakers to confirm the absolute numbers.

Contact your Sensory FAE about synthetic data for your phrase.

### 3.6 If You Must Record Over the Air

If the test audio has to be played back over the air into the device — for example, to capture it through the device's own microphones — play back **unmodified, clean recordings** through a loudspeaker that accurately reproduces a human voice (such as an artificial mouth or a head-and-torso simulator), not a generic speaker. A small consumer speaker colors the audio and can change the results. Capture what the device hears once, then score the captured audio in simulation ([1.3](#13-simulate-dont-test-live)).

### 3.7 Audio Requirements

Enrollment and test recordings should be as close as possible to what the recognizer will hear on the device. Check them against this list before you benchmark; the script's `check` subcommand verifies the items marked ✓ ([Step 2](#4-step-2--lay-out-the-corpus)).

| Requirement | Why |
|---|---|
| ✓ **16-bit PCM WAV, mono, 16 kHz** (or the model's sample rate) | The format the SDK models expect |
| **Never upsample** audio recorded at a lower rate (such as 8 kHz) | The missing high frequencies can't be restored, and the model relies on them |
| **No µ-law or lossy compression** (MP3, AAC, and so on), even as an intermediate step | Compression artifacts change what the model hears |
| ✓ **At least 0.5 s of naturally recorded quiet before and after the phrase** | Enrollment and recognition use the audio around the phrase; padding must be real recorded background, not inserted silence |
| **No editing or post-processing** — no trimming, splicing, noise reduction, automatic gain control, or level "cleanup" | Only controlled amounts of added noise ([Step 3](#5-step-3--add-noise-conditions)) should change the recordings |
| ✓ **No runs of identical samples** (digital silence) | Never occurs on a real device, and skews results |
| ✓ **No clipping**, at any gain stage between the microphone and the file | Clipping distorts the speech |
| ✓ **A good recording level** — peaks using about 12–15 of the 16 bits (roughly −18 to −6 dBFS) | Too low loses detail, especially for soft talkers; too high risks clipping |
| **A microphone with a flat response from 100 Hz to 8 kHz** — ideally the device's own microphone and audio path | A different microphone or front end changes the results |
| **The same audio path for enrollment and recognition** — for a DSP target, the DSP's own channel | A mismatch between enrollment and test audio raises errors |

---

## 4. Step 2 — Lay Out the Corpus

The benchmark script in [6.2](#62-the-benchmark-script) expects one target phrase per corpus folder, laid out like this:

```
my-corpus/
  speakers/
    spk001/
      enroll/          4 enrollment recordings
        001.wav ...
      test/
        clean/         clean test recordings
          001.wav ...
        snr20/         optional: other conditions, one folder each
          001.wav ...
    spk002/
      ...
  noise/               non-target audio for FAR (optional)
    conversation-01.wav
    tv-01.wav ...
```

- **Speaker folder names** become the enrolled user tag, so keep them alphanumeric with no spaces.
- **Condition folder names** under `test/` are free-form — `clean`, `snr20`, `arms-length`, `across-room` — and each is reported separately. Folders named `snrNN` sort from least to most noisy after `clean`.
- Every speaker should have the same set of condition folders, so each condition has the same speakers behind it.

**Check the recordings before you benchmark.** The script's `check` subcommand checks every enrollment and test recording against the audio requirements it can verify automatically ([3.7](#37-audio-requirements)):

```bash
python enrolled_benchmark.py check --corpus my-corpus
```

```
speakers/spk012/enroll/002.wav: 8000 Hz, expected 16000 Hz
speakers/spk012/test/clean/007.wav: 1 run(s) of identical samples, longest 200 ms of value 0 at 0.10 s
speakers/spk031/test/clean/004.wav: clipped in 492 place(s)
speakers/spk047/enroll/001.wav: peak level only -30 dBFS
speakers/spk047/test/clean/012.wav: 0.20 s of quiet before the phrase and 1.27 s after; at least 0.5 s needed
checked 1840 recordings
  sample rate:        1 recording(s)
  identical samples:  1 recording(s)
  clipping:           1 recording(s)
  low level:          1 recording(s)
  short padding:      1 recording(s)
```

| Check | Flags a recording when |
|---|---|
| sample rate | It isn't at the expected rate (`--rate`, default 16000 Hz) |
| identical samples | It has a run of identical sample values at least `--min-run-ms` long (default 10 ms) |
| clipping | It has three or more full-scale samples in a row |
| low level | Its loudest sample is below −18 dBFS (uses fewer than 12 bits) |
| short padding | It has less than 0.5 s of quiet before or after the phrase. Checked on enrollment and `clean` test recordings only, since in noise there's no quiet to measure |

**Identical samples** need a word of explanation. A real microphone always picks up a little noise, so even the quiet parts of a real recording change slightly from one sample to the next. Long runs of exactly the same value — usually zeros, "digital silence" — never happen on a device. They come from synthetic or edited audio, splicing, or dropped audio buffers, and they give the recognizer a perfectly silent background it will never hear in use. Adding noise hides the runs in the noisy conditions but not in `clean`, so a corpus with digital silence can show a *higher* FRR on clean audio than with light noise added.

Fix flagged recordings at the source — re-record, re-capture, or regenerate them — rather than editing the audio, which breaks the "no editing" requirement. `check` also flags clipping in noisy conditions; if `noisify` output clips, the noise or speech is too loud for that SNR. `check` only reports problems; it doesn't stop a benchmark from running.

---

## 5. Step 3 — Add Noise Conditions

To test in noise, mix noise into each clean test recording at fixed signal-to-noise ratios. Sensory typically uses **30, 20, 10, and 5 dBA SNR**, but use whatever range matches your product. The script's `noisify` subcommand reads every `test/clean/*.wav` file, mixes in a random segment of noise at each SNR, and writes the results to `test/snr30/`, `test/snr20/`, and so on:

```bash
python enrolled_benchmark.py noisify --corpus my-corpus --noise mix-noise --snr 30,20,10,5
```

**Which noise to use.** `--noise` takes a folder of noise recordings (or a single file). For each test recording, `noisify` picks one file from the folder at random and mixes in a random segment of it. Use the same principles as for FAR noise ([3.4](#34-non-target-audio-for-far)):

- **Use real recordings, mixed by type.** Put recordings of each noise type the product will hear in the folder — competing speakers, TV, radio, car, and so on. Files are picked in proportion to their length, so the mix follows how many hours of each type the folder holds. Agree the mix in advance with everyone comparing results, written for example as `{competing speaker 33%, TV 20%, radio 13%, car 34%}`.
- **Avoid synthetic, steady noise.** White, pink, or brown noise — even filtered — has the same character from moment to moment. The recognizer's normalization removes that kind of noise easily, so results on it look better than they will in real life. Real noise changes over time, which is what makes it hard.
- **Don't focus on extreme levels.** Results at very low SNRs say little about everyday use, and tuning settings to fix them usually makes IAR and FAR worse at typical levels.

**How SNR is measured.** SNR should compare the level of the *speech* with the level of the noise. A test recording also contains silence before and after the phrase, and that silence has to stay in the file: enrollment and recognition both use the audio around the phrase, so trimming it would change the results. If the speech level were averaged over the whole file, the silence would pull it down, and a recording mixed at "5 dB" would really have a much higher SNR on the speech. `noisify` avoids this without any trimming:

- **Speech level** is measured over the speech only. The recording is split into 20 ms frames, and only frames that are at least 15 dB above the recording's own background level (its quietest 10% of frames) and within 30 dB of its loudest frame are averaged. This approximates the *active speech level* of ITU-T P.56.
- **Noise level** is measured over the whole noise segment that's mixed in, since the noise is present the whole time.
- **Both levels are A-weighted**, so the SNR is in **dBA** (decibels with A-weighting). A-weighting reduces the contribution of very low frequencies, so the measured SNR is closer to how loud the noise sounds to a listener. Without it, noise with strong low-frequency content, such as car noise, measures as much louder than it sounds. The weighting is used only to measure the levels: the noise is mixed into the recording unweighted, and the recording keeps its natural silence.

Other details:

- Noise files must be 16-bit mono PCM at the same sample rate as the corpus. A file shorter than a test recording isn't used for that recording; at least one file must be longer than every test recording.
- `noisify` writes `noisify-log.csv` in the corpus folder, listing which noise file and starting point went into each test recording, so you can trace a surprising result back to its noise.
- The speech-level rule assumes the "clean" recordings really are quiet. If a recording's own background noise is less than about 15 dB below its speech, the rule can't separate the two well, and the SNR on the speech will be somewhat off. Recordings that noisy shouldn't be used as `clean` anyway.
- The random choices are seeded, so re-running `noisify` with the same noise folder reproduces the same noisy corpus.
- Use different noise recordings here from the FAR noise in `noise/`, and keep the folder outside the corpus folder, so the two tests stay independent.

---

## 6. Step 4 — Enroll and Score Every Speaker

### 6.1 How Scoring Works

For each speaker, the script:

1. Enrolls the speaker's `enroll/*.wav` recordings into an enrolled model with `spot-enroll`.
2. Runs that model with `snsr-eval` over **every** test recording in the corpus, plus the FAR noise, at the `sv-threshold` stored in the enrolled model (or at the value you pass with `--sv-threshold`).
3. Scores each test recording as detected or not:
   - the speaker's own recording, not detected → **false reject**
   - another speaker's recording, detected → **imposter accept**
   - every detection in the noise audio → **false accept**

It uses `snsr-eval`'s list mode: `-i list.txt` runs the model on each file in the list separately (resetting between files), and `-o results.txt` writes one tab-separated line per file — the index, the filename, then a group of four fields (begin ms, end ms, score, recognized text) for each detection:

```
1	corpus/speakers/spk1/test/clean/armadillo-1-4.wav	135	765	-1	spk1
2	corpus/speakers/spk1/test/clean/armadillo-1-5.wav	195	810	-1	spk1
3	corpus/speakers/spk6/test/clean/armadillo-6-4.wav
```

A line with only the index and filename means nothing was detected in that file.

On THF/TNL, `sv-threshold` is enforced inside the SDK — a detection below the threshold is never reported — so running at a given `sv-threshold` gives exactly what the device will do at that setting. See [Creating and Using Enrolled Models §5.4](enrolled-models.md#54-biometric-scoring-thftnl-vs-thf-micro).

### 6.2 The Benchmark Script

Save this as `enrolled_benchmark.py`:

```python
#!/usr/bin/env python3
"""Benchmark enrolled wake word models: FRR, IAR, and false accepts per day.

Corpus layout (one target phrase per corpus):
  <corpus>/speakers/<speaker>/enroll/*.wav            enrollment recordings
  <corpus>/speakers/<speaker>/test/<condition>/*.wav  test recordings
  <corpus>/noise/*.wav                                non-target audio (optional)

Subcommands:
  check    check the recordings against the audio requirements
  noisify  mix noise into clean test recordings at fixed SNRs
  run      enroll every speaker, then score every model
"""

import argparse
import array
import csv
import itertools
import math
import random
import subprocess
import sys
import threading
import time
import wave
from concurrent.futures import ThreadPoolExecutor
from pathlib import Path


def exe(sdk: Path, name: str) -> str:
    path = sdk / "bin" / name
    return str(path.with_suffix(".exe")) if path.with_suffix(".exe").exists() else str(path)


def wav_seconds(path: Path) -> float:
    with wave.open(str(path)) as w:
        return w.getnframes() / w.getframerate()


def read_pcm(path: Path):
    with wave.open(str(path)) as w:
        if w.getsampwidth() != 2 or w.getnchannels() != 1:
            sys.exit(f"{path}: expected 16-bit mono PCM")
        params = w.getparams()
        samples = array.array("h", w.readframes(w.getnframes()))
    return params, samples


def power(samples) -> float:
    return sum(s * s for s in samples) / max(len(samples), 1)


# ---------------------------------------------------------------- noisify

# A-weighting pole frequencies in Hz (IEC 61672). The analog response is
#   s^4 / ((s + w1)^2 (s + w2) (s + w3) (s + w4)^2),  wN = 2 * pi * fN
A_WEIGHT_POLES = (20.598997, 107.65265, 737.86223, 12194.217)


def a_weight(samples, rate: int):
    """A-weight a signal, as a cascade of six first-order filters.

    The four high-pass sections use the bilinear transform. The two 12.2 kHz
    low-pass poles lie above the Nyquist frequency at 16 kHz, where the bilinear
    transform badly warps them, so they map directly to z-plane poles instead.
    At 16 kHz this is within 0.7 dB of IEC 61672 up to 4 kHz and 2.5 dB up to 7.5 kHz.

    Only used to measure levels: noisify compares the A-weighted power of speech
    and noise, so the filter's overall gain cancels out and isn't normalized.
    """
    f1, f2, f3, f4 = A_WEIGHT_POLES
    k = 2 * rate
    sections = []                              # (b0, b1, a1) for each section
    for f in (f1, f1, f2, f3):                 # high-pass: s / (s + w)
        w = 2 * math.pi * f
        b0 = k / (k + w)
        sections.append((b0, -b0, (w - k) / (k + w)))
    p = math.exp(-2 * math.pi * f4 / rate)     # low-pass: w / (s + w), twice
    sections += [(1 - p, 0.0, -p)] * 2

    out = [float(s) for s in samples]
    for b0, b1, a1 in sections:
        x1 = y1 = 0.0
        for i, x in enumerate(out):
            y1 = b0 * x + b1 * x1 - a1 * y1
            x1 = x
            out[i] = y1
    return out


def speech_frames(signal, rate: int, frame_ms: int = 20):
    """Split a signal into frames and pick out the ones that hold speech.

    Returns (frame length in samples, frame powers, indexes of speech frames).
    A speech frame is at least 15 dB above the recording's background level (its
    10th-percentile frame) and within 30 dB of its loudest frame. Used for both
    the speech level in noisify and the padding check.
    """
    n = rate * frame_ms // 1000
    frames = [power(signal[i:i + n]) for i in range(0, len(signal) - n + 1, n)]
    if not frames or max(frames) == 0:
        return n, frames, []
    background = sorted(frames)[len(frames) // 10]
    threshold = max(max(frames) * 10 ** (-30 / 10), background * 10 ** (15 / 10))
    return n, frames, [i for i, p in enumerate(frames) if p >= threshold]


def speech_power(weighted, rate: int) -> float:
    """Average power over the speech frames only, so leading/trailing silence
    doesn't count; this approximates the active speech level of ITU-T P.56."""
    _, frames, speech = speech_frames(weighted, rate)
    return sum(frames[i] for i in speech) / len(speech) if speech else 0.0


def noisify(args) -> None:
    corpus = Path(args.corpus)
    source = Path(args.noise)
    noise_files = sorted(source.glob("*.wav")) if source.is_dir() else [source]
    if not noise_files:
        sys.exit(f"no .wav files in {source}")
    noise_info = {}                            # file -> (sample rate, length in samples)
    for f in noise_files:
        with wave.open(str(f)) as w:
            if w.getsampwidth() != 2 or w.getnchannels() != 1:
                sys.exit(f"{f}: expected 16-bit mono PCM")
            noise_info[f] = (w.getframerate(), w.getnframes())
    rng = random.Random(0)  # fixed seed: the noisy corpus is reproducible
    snrs = [int(s) for s in args.snr.split(",")]

    cleans = sorted(corpus.glob("speakers/*/test/clean/*.wav"))
    log = [["recording", "noise_file", "noise_start_s"]]
    for k, clean in enumerate(cleans, 1):
        params, speech = read_pcm(clean)
        rate = params.framerate
        usable = [f for f in noise_files
                  if noise_info[f][0] == rate and noise_info[f][1] >= len(speech)]
        if not usable:
            sys.exit(f"no noise file at {rate} Hz is as long as {clean}")
        # Pick a noise file with probability proportional to its length, so the
        # mix of noise types follows how many hours of each the folder holds.
        noise_file = usable[0] if len(usable) == 1 else \
            rng.choices(usable, weights=[noise_info[f][1] for f in usable])[0]
        start = rng.randrange(0, noise_info[noise_file][1] - len(speech) + 1)
        with wave.open(str(noise_file)) as w:
            w.setpos(start)
            segment = array.array("h", w.readframes(len(speech)))
        log.append([clean.relative_to(corpus).as_posix(), noise_file.name, f"{start / rate:.3f}"])

        # SNR in dBA: A-weighted speech level over the loud frames only, against
        # the A-weighted level of the noise segment actually mixed in. The audio
        # itself is mixed unweighted, with its natural silence left in place.
        p_speech = speech_power(a_weight(speech, rate), rate)
        p_noise = power(a_weight(segment, rate))
        if p_speech == 0 or p_noise == 0:
            print(f"warning: {clean} or its noise segment is silent, skipping", file=sys.stderr)
            continue

        for snr in snrs:
            gain = math.sqrt(p_speech / (p_noise * 10 ** (snr / 10)))
            mixed = array.array("h", (
                max(-32768, min(32767, int(s + gain * n)))
                for s, n in zip(speech, segment)))
            out = clean.parent.parent / f"snr{snr}" / clean.name
            out.parent.mkdir(exist_ok=True)
            with wave.open(str(out), "wb") as w:
                w.setparams(params)
                w.writeframes(mixed.tobytes())
        if sys.stdout.isatty():
            print(f"\rnoisified {k}/{len(cleans)} recordings", end="", flush=True)
    with open(corpus / "noisify-log.csv", "w", newline="", encoding="utf-8") as fh:
        csv.writer(fh).writerows(log)
    print(("\n" if sys.stdout.isatty() else "") +
          f"wrote {', '.join(f'snr{s}' for s in snrs)} test conditions (SNR in dBA)"
          f" and {corpus / 'noisify-log.csv'}")


# ---------------------------------------------------------------- check

FULL_SCALE = (-32768, 32767)


def sample_runs(samples, rate: int, min_ms: float):
    """Scan for runs of identical samples and for clipping, in one pass.

    Returns (runs, clips): runs is (start seconds, length ms, value) for each run
    of one value at least min_ms long; clips counts runs of 3 or more full-scale
    samples. A real microphone always picks up some noise, so even a quiet
    recording changes from sample to sample. Long runs of one value (usually 0)
    are artificial: digital silence, splicing, or a dropped audio buffer.
    """
    runs, clips, pos = [], 0, 0
    for value, group in itertools.groupby(samples):
        n = sum(1 for _ in group)
        if n * 1000 >= min_ms * rate:
            runs.append((pos / rate, n * 1000 / rate, value))
        if value in FULL_SCALE and n >= 3:
            clips += 1
        pos += n
    return runs, clips


def padding(samples, rate: int):
    """Return (leading, trailing) seconds of quiet before and after the speech.

    Uses noisify's speech-frame rule, but speech must last at least three frames
    (60 ms) in a row, so a click at the start or end doesn't count as speech.
    """
    n, frames, speech = speech_frames(samples, rate)
    marks = set(speech)
    sustained = [i for i in speech if {i, i + 1, i + 2} <= marks]
    if not sustained:
        return 0.0, 0.0
    return sustained[0] * n / rate, (len(samples) - (sustained[-1] + 3) * n) / rate


def check(args) -> None:
    corpus = Path(args.corpus)
    files = sorted(corpus.glob("speakers/*/enroll/*.wav")) + \
        sorted(corpus.glob("speakers/*/test/*/*.wav"))
    tty = sys.stdout.isatty()
    totals = {}
    for k, f in enumerate(files, 1):
        if tty:
            print(f"\rchecking {k}/{len(files)} recordings", end="", flush=True)
        params, samples = read_pcm(f)
        rate = params.framerate
        issues = []

        if rate != args.rate:
            issues.append(("sample rate", f"{rate} Hz, expected {args.rate} Hz"))
        runs, clips = sample_runs(samples, rate, args.min_run_ms)
        if runs:
            start, ms, value = max(runs, key=lambda r: r[1])
            issues.append(("identical samples", f"{len(runs)} run(s) of identical samples, "
                           f"longest {ms:.0f} ms of value {value} at {start:.2f} s"))
        if clips:
            issues.append(("clipping", f"clipped in {clips} place(s)"))
        peak = max((abs(s) for s in samples), default=0)
        if 0 < peak < 4096:                    # under 12 bits of the 16 in use
            issues.append(("low level", f"peak level only {20 * math.log10(peak / 32768):.0f} dBFS"))
        # Padding only makes sense on clean audio: in noise, every frame is "loud".
        if f.parent.name in ("enroll", "clean"):
            lead, trail = padding(samples, rate)
            if min(lead, trail) < 0.5:
                issues.append(("short padding", f"{lead:.2f} s of quiet before the phrase "
                               f"and {trail:.2f} s after; at least 0.5 s needed"))

        if issues:
            print(f"\r{f.relative_to(corpus).as_posix()}: " + "; ".join(t for _, t in issues))
        for name, _ in issues:
            totals[name] = totals.get(name, 0) + 1

    print(f"\rchecked {len(files)} recordings" + (" " * 20 if tty else ""))
    for name in ("sample rate", "identical samples", "clipping", "low level", "short padding"):
        print(f"  {name + ':':<19} {totals.get(name, 0)} recording(s)")


# ---------------------------------------------------------------- run

def enroll(sdk: Path, enroller: str, speaker: Path, model: Path) -> None:
    recordings = sorted(str(p) for p in (speaker / "enroll").glob("*.wav"))
    cmd = [exe(sdk, "spot-enroll"), "-t", enroller, "-o", str(model),
           f"+{speaker.name}", *recordings]
    subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL)


def sv_threshold(sdk: Path, model: Path, probe: Path) -> str:
    """Query the sv-threshold stored in an enrolled model."""
    # Pass an audio file: with none, snsr-eval listens to the live microphone.
    cmd = [exe(sdk, "snsr-eval"), "-t", str(model), "-q", "sv-threshold", str(probe)]
    result = subprocess.run(cmd, check=True, capture_output=True, text=True)
    for line in result.stdout.splitlines():
        if line.startswith("sv-threshold ="):
            return line.split("=")[1].strip()
    return "unknown"


def evaluate(sdk: Path, model: Path, files, work: Path, threshold=None):
    """Return {file: detection count} for one model."""
    list_file = work / "list.txt"
    out_file = work / "results.txt"
    list_file.write_text("\n".join(str(f) for f in files) + "\n", encoding="utf-8")
    out_file.unlink(missing_ok=True)  # don't let a previous run's results look like progress
    cmd = [exe(sdk, "snsr-eval"), "-t", str(model)]
    if threshold is not None:
        cmd += ["-s", f"sv-threshold={threshold}"]
    cmd += ["-i", str(list_file), "-o", str(out_file)]
    subprocess.run(cmd, check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

    # Each line: index, file, then (begin, end, score, text) per detection.
    counts = {}
    for line in out_file.read_text(encoding="utf-8").splitlines():
        cols = line.split("\t")
        counts[cols[1]] = (len(cols) - 2) // 4
    return counts


class Progress:
    """One-line progress display, weighted by seconds of audio scored.

    Polls each running job's results.txt (one line per finished file), so the
    percentage moves while long snsr-eval jobs run, not only when they finish.
    """

    def __init__(self, files, n_models):
        self.cum = [0.0]                       # cum[k] = audio seconds in the first k files
        for f in files:
            self.cum.append(self.cum[-1] + wav_seconds(f))
        self.per_model = self.cum[-1]
        self.total = self.per_model * n_models
        self.n_models = n_models
        self.done = 0
        self.active = set()                    # results.txt paths being written now
        self.lock = threading.Lock()
        self.start = time.time()
        self.tty = sys.stdout.isatty()
        self.last_step = -1
        self.stop = threading.Event()
        self.thread = threading.Thread(target=self._loop, daemon=True)

    def _partial(self, path: Path) -> float:
        try:
            with open(path, encoding="utf-8", errors="replace") as fh:
                k = sum(1 for _ in fh)
        except OSError:
            return 0.0
        return self.cum[min(k, len(self.cum) - 1)]

    def _line(self) -> tuple:
        with self.lock:
            done, active = self.done, list(self.active)
        secs = done * self.per_model + sum(self._partial(p) for p in active)
        frac = min(secs / self.total, 1.0) if self.total else 1.0
        elapsed = time.time() - self.start
        eta = elapsed * (1 - frac) / frac if frac > 0.001 else None
        clock = lambda t: "--:--" if t is None else f"{int(t) // 60:02d}:{int(t) % 60:02d}"
        return (f"scoring {100 * frac:5.1f}%  {done}/{self.n_models} models  "
                f"elapsed {clock(elapsed)}  remaining {clock(eta)}"), frac

    def show(self, final=False) -> None:
        text, frac = self._line()
        if self.tty:
            print("\r" + text.ljust(70), end="\n" if final else "", flush=True)
        else:                                  # redirected output: print every 10%
            step = int(frac * 10)
            if step > self.last_step or final:
                self.last_step = step
                print(text, flush=True)

    def message(self, text: str) -> None:
        """Print a normal line without leaving a half-drawn progress line above it."""
        if self.tty:
            print("\r" + " " * 70 + "\r", end="")
        print(text, flush=True)

    def _loop(self) -> None:
        while not self.stop.wait(1.0):
            self.show()

    def __enter__(self):
        self.thread.start()
        return self

    def __exit__(self, *exc):
        self.stop.set()
        self.thread.join()
        self.show(final=True)


def run(args) -> None:
    sdk = Path(args.sdk)
    corpus = Path(args.corpus)
    out = Path(args.out)
    (out / "models").mkdir(parents=True, exist_ok=True)

    speakers = sorted(p for p in (corpus / "speakers").iterdir() if p.is_dir())
    tests = [(spk.name, f.parent.name, f)
             for spk in speakers for f in sorted(spk.glob("test/*/*.wav"))]
    if args.conditions:
        wanted = {c.strip() for c in args.conditions.split(",")}
        tests = [t for t in tests if t[1] in wanted]
        if not tests:
            sys.exit(f"no test recordings in conditions: {args.conditions}")
    noise = [] if args.skip_noise else sorted((corpus / "noise").glob("*.wav"))
    noise_hours = sum(wav_seconds(f) for f in noise) / 3600
    # clean first, then snrNN from least to most noisy
    conditions = sorted({cond for _, cond, _ in tests},
                        key=lambda c: -float(c[3:]) if c.startswith("snr") else -math.inf)

    def score(spk: Path):
        """Enroll one speaker if needed, then score their model on everything."""
        model = out / "models" / f"{spk.name}.snsr"
        if not model.exists():
            progress.message(f"enrolling {spk.name}")
            enroll(sdk, args.enroller, spk, model)
        threshold = None
        if args.sv_threshold is None:
            threshold = sv_threshold(sdk, model, tests[0][2])
        work = out / "work" / spk.name  # each job needs its own list/results files
        work.mkdir(parents=True, exist_ok=True)
        results_file = work / "results.txt"
        with progress.lock:
            progress.active.add(results_file)
        try:
            counts = evaluate(sdk, model, [f for _, _, f in tests] + noise, work,
                              args.sv_threshold)
        finally:
            with progress.lock:
                progress.active.discard(results_file)
                progress.done += 1
        return spk.name, threshold, counts

    # Each job runs its own spot-enroll/snsr-eval processes, so -j N uses N cores.
    # Results are identical for any -j: each model is still scored on the same audio.
    progress = Progress([f for _, _, f in tests] + noise, len(speakers))
    with progress, ThreadPoolExecutor(max_workers=args.jobs) as pool:
        results = list(pool.map(score, speakers))

    # Per-speaker counts, so results can be recomputed for any resample of speakers.
    n = len(speakers)
    idx = {spk.name: k for k, spk in enumerate(speakers)}
    n_files = {c: [0] * n for c in conditions}        # test recordings per speaker
    for owner, cond, _ in tests:
        n_files[cond][idx[owner]] += 1
    false_rejects = {c: [0] * n for c in conditions}  # [model]
    imposter_accepts = {c: [[0] * n for _ in range(n)] for c in conditions}  # [model][owner]
    false_accepts = [0] * n                           # [model]
    model_thresholds = set()

    for name, threshold, counts in results:
        k = idx[name]
        if threshold is not None:
            model_thresholds.add(threshold)
        for owner, cond, f in tests:
            hit = counts.get(str(f), 0) > 0
            if owner == name:
                false_rejects[cond][k] += not hit
            else:
                imposter_accepts[cond][k][idx[owner]] += hit
        false_accepts[k] = sum(counts.get(str(f), 0) for f in noise)

    def rates(weights):
        """FRR and IAR (%) per condition, and FA/day, with speaker k counted weights[k] times."""
        drawn = [(k, w) for k, w in enumerate(weights) if w]
        frr, iar = {}, {}
        for c in conditions:
            files = n_files[c]
            target = sum(w * files[k] for k, w in drawn)
            fr = sum(w * false_rejects[c][k] for k, w in drawn)
            # Every drawn model against every other drawn speaker's recordings.
            all_files = sum(w * files[k] for k, w in drawn)
            imposter = sum(w * (all_files - w * files[k]) for k, w in drawn)
            ia = sum(w * sum(w2 * imposter_accepts[c][k][l] for l, w2 in drawn)
                     for k, w in drawn)
            frr[c] = 100 * fr / target if target else math.nan
            iar[c] = 100 * ia / imposter if imposter else math.nan
        hours = noise_hours * sum(weights)
        fa_day = 24 * sum(w * false_accepts[k] for k, w in drawn) / hours if hours else math.nan
        return frr, iar, fa_day

    # Bootstrap 95% confidence intervals: resample whole speakers with replacement,
    # since one speaker's recordings (and their model's results) aren't independent.
    # Seeded, so the same results always give the same intervals.
    rng = random.Random(0)
    samples = []
    for _ in range(args.bootstrap):
        weights = [0] * n
        for _ in range(n):
            weights[rng.randrange(n)] += 1
        samples.append(rates(weights))

    def interval(values):
        values = sorted(v for v in values if not math.isnan(v))
        if not values:
            return math.nan, math.nan
        return (values[int(0.025 * (len(values) - 1))],
                values[int(math.ceil(0.975 * (len(values) - 1)))])

    if args.sv_threshold is None:
        threshold = "/".join(sorted(model_thresholds)) + " (model default)"
    else:
        threshold = f"{args.sv_threshold} (override)"
    model_hours = noise_hours * n
    frr, iar, fa_day = rates([1] * n)
    fa_ci = interval([s[2] for s in samples])
    show_ci = args.bootstrap > 0

    def ci_txt(ci, fmt):
        return f" [{ci[0]:{fmt}}-{ci[1]:{fmt}}]" if show_ci else ""

    def ci_cols(ci, fmt):
        return [f"{ci[0]:{fmt}}", f"{ci[1]:{fmt}}"] if show_ci else ["", ""]

    with open(out / "summary.csv", "w", newline="", encoding="utf-8") as fh:
        writer = csv.writer(fh)
        writer.writerow(["sv_threshold", "condition", "target_trials", "false_rejects",
                         "FRR_pct", "FRR_ci_low", "FRR_ci_high",
                         "imposter_trials", "imposter_accepts",
                         "IAR_pct", "IAR_ci_low", "IAR_ci_high",
                         "false_accepts", "model_hours", "FA_per_day",
                         "FA_ci_low", "FA_ci_high"])
        if args.skip_noise:
            noise_txt = "noise skipped"
        else:
            noise_txt = f"{noise_hours:.2f} h noise ({model_hours:.2f} model-hours)"
        print(f"\n{n} speakers, {noise_txt}, sv-threshold {threshold}")
        if show_ci:
            print(f"95% confidence intervals from {args.bootstrap} bootstrap resamples of speakers")
        print()
        width, fa_width = (34, 22) if show_ci else (16, 8)
        print(f"{'condition':<10} {'FRR':>{width}} {'IAR':>{width}} {'FA/day':>{fa_width}}")
        for c in conditions:
            n_t = sum(n_files[c])
            fr = sum(false_rejects[c])
            n_i = sum(n_files[c]) * (n - 1)
            ia = sum(map(sum, imposter_accepts[c]))
            frr_ci = interval([s[0][c] for s in samples])
            iar_ci = interval([s[1][c] for s in samples])
            if args.skip_noise:
                fa_cols, fa_txt = [""] * 5, f"{'skipped':>8}"
            else:
                fa_cols = [sum(false_accepts), f"{model_hours:.2f}", f"{fa_day:.3f}",
                           *ci_cols(fa_ci, ".3f")]
                fa_txt = f"{fa_day:8.3f}" + ci_txt(fa_ci, ".3f")
            writer.writerow([threshold, c, n_t, fr, f"{frr[c]:.2f}", *ci_cols(frr_ci, ".2f"),
                             n_i, ia, f"{iar[c]:.2f}", *ci_cols(iar_ci, ".2f"), *fa_cols])
            print(f"{c:<10} {frr[c]:6.2f}% ({fr:>3}/{n_t:<4}){ci_txt(frr_ci, '6.2f')} "
                  f"{iar[c]:6.2f}% ({ia:>3}/{n_i:<4}){ci_txt(iar_ci, '6.2f')} {fa_txt}")
    print(f"\nwrote {out / 'summary.csv'}")


def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__,
                                     formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = parser.add_subparsers(dest="command", required=True)

    p = sub.add_parser("check", help="check enroll/test recordings against the audio requirements")
    p.add_argument("--corpus", required=True)
    p.add_argument("--rate", type=int, default=16000,
                   help="expected sample rate in Hz (default 16000)")
    p.add_argument("--min-run-ms", type=float, default=10,
                   help="report runs of identical samples at least this long (default 10)")
    p.set_defaults(func=check)

    p = sub.add_parser("noisify", help="mix noise into test/clean at fixed SNRs")
    p.add_argument("--corpus", required=True)
    p.add_argument("--noise", required=True,
                   help="folder of 16-bit mono noise WAVs (or one WAV), same rate as corpus")
    p.add_argument("--snr", default="30,20,10,5", help="comma-separated SNRs in dBA")
    p.set_defaults(func=noisify)

    p = sub.add_parser("run", help="enroll and score every speaker")
    p.add_argument("--sdk", required=True, help="SDK install root (contains bin/)")
    p.add_argument("--enroller", required=True, help="enroller task model (udt-*/eft-*.snsr)")
    p.add_argument("--corpus", required=True)
    p.add_argument("--out", required=True, help="output folder for models and summary.csv")
    p.add_argument("--sv-threshold", type=float,
                   help="override the sv-threshold stored in each enrolled model")
    p.add_argument("--conditions",
                   help="comma-separated test conditions to score, e.g. clean,snr40 (default: all)")
    p.add_argument("--skip-noise", action="store_true",
                   help="don't score the FAR noise (FA columns left blank)")
    p.add_argument("--bootstrap", type=int, default=1000,
                   help="bootstrap resamples for 95%% confidence intervals (default 1000, 0 = off)")
    p.add_argument("-j", "--jobs", type=int, default=1,
                   help="number of speakers to enroll and score at once (default 1)")
    p.set_defaults(func=run)

    args = parser.parse_args()
    args.func(args)


if __name__ == "__main__":
    main()
```

### 6.3 Running It

```bash
python enrolled_benchmark.py run \
    --sdk C:/Sensory/TrulyNaturalSDK/7.9.0 \
    --enroller C:/Sensory/TrulyNaturalSDK/7.9.0/model/udt-universal-3.67.1.0.snsr \
    --corpus my-corpus --out results -j 8
```

| Option | Description |
|---|---|
| `--sdk` | SDK install root — the folder containing `bin/` |
| `--enroller` | Enroller task model to enroll with — use the one your product ships |
| `--corpus` | Corpus folder laid out as in [Step 2](#4-step-2--lay-out-the-corpus) |
| `--out` | Output folder for the enrolled models (`models/`) and `summary.csv` |
| `--sv-threshold` | Optional. Overrides the `sv-threshold` stored in each enrolled model. Leave it off to benchmark the model as it ships |
| `--conditions` | Optional. Comma-separated test conditions to score, such as `clean,snr20`. Leave it off to score every condition folder |
| `--skip-noise` | Optional. Don't score the FAR noise. The FA columns are reported as `skipped` in the table and left blank in `summary.csv` |
| `--bootstrap` | Number of bootstrap resamples used for the 95% confidence intervals (default `1000`). `0` turns the intervals off — see [8.3](#83-confidence-intervals) |
| `-j`, `--jobs` | Number of speakers to enroll and score at once (default `1`). Set it to about the number of CPU cores — see below |

Enrolled models are saved to `<out>/models/<speaker>.snsr` and reused if they already exist, so you can re-run with `--sv-threshold` without re-enrolling. Delete `<out>/models/` to re-enroll — for example, after changing the enroller model or the enrollment recordings.

Each run overwrites `<out>/summary.csv`, so copy it somewhere else first if you want to compare runs.

Each speaker's model is evaluated against every test recording in the corpus and all of the FAR noise, so run time grows with the square of the number of speakers, plus the number of speakers × hours of noise. Enrollment adds a fixed cost per speaker, roughly 15 seconds each on a desktop PC with `udt-universal-3.67.1.0.snsr`. As a rough guide for scoring, one enrolled model processed an hour of audio in about 4 seconds on the same PC, so the recommended 150 hours of noise takes about 10 minutes per model, or about 8 hours for 50 speakers with `-j 1`. The 15-hour minimum takes about a minute per model.

`-j` divides that time across CPU cores. Each job runs its own `spot-enroll` and `snsr-eval` processes for one speaker at a time, so set `-j` to about the number of cores on the machine. On an 8-core PC, a 16-speaker corpus with an hour of noise took 4 minutes 49 seconds with `-j 1` and 51 seconds with `-j 8`, so the 8-hour, 50-speaker run above drops to roughly 1½ hours on such a machine. The results are identical for any `-j` value: every model is still scored on exactly the same audio; it's just scored alongside other models instead of after them ([1.3](#13-simulate-dont-test-live)).

**Quicker partial runs.** Scoring the FAR noise usually takes most of the run time, so while you're setting up a corpus or trying a different `--sv-threshold`, it can help to leave parts out:

- `--skip-noise` scores FRR and IAR only. Run the noise once you're happy with the rest.
- `--conditions clean` (or any list of condition folders) scores only those conditions. Conditions you leave out aren't scored at all, and the FRR and IAR for the conditions you include are exactly what a full run reports for them.

Because enrolled models are reused, a quick run doesn't waste the enrollment work: a later full run with the same `--out` folder picks up the same models.

**Progress display.** While models are being scored, the script shows one line of progress: the percentage of audio scored so far across all models, how many models have finished, elapsed time, and a rough estimate of the time remaining. The percentage is weighted by seconds of audio and updates while each `snsr-eval` job runs, so a long noise file doesn't make it look stuck. In a terminal the line updates in place; when output is redirected to a file, it prints a new line every 10% instead. Enrollment isn't counted in the percentage, so it stays at 0% until the first model starts scoring.

### 6.4 Reading the Output

The script prints a table and writes the same numbers to `<out>/summary.csv`:

```
enrolling spk1
enrolling spk6
scoring 100.0%  2/2 models  elapsed 00:27  remaining 00:00

2 speakers, 0.01 h noise (0.01 model-hours), sv-threshold 0.49 (model default)
95% confidence intervals from 1000 bootstrap resamples of speakers

condition                                 FRR                                IAR                 FA/day
clean        0.00% (  0/4   ) [  0.00-  0.00]   0.00% (  0/4   ) [  0.00-  0.00]    0.000 [0.000-0.000]
snr20       25.00% (  1/4   ) [  0.00- 50.00]   0.00% (  0/4   ) [  0.00-  0.00]    0.000 [0.000-0.000]
snr10       25.00% (  1/4   ) [  0.00- 50.00]   0.00% (  0/4   ) [  0.00-  0.00]    0.000 [0.000-0.000]
snr5        25.00% (  1/4   ) [  0.00- 50.00]   0.00% (  0/4   ) [  0.00-  0.00]    0.000 [0.000-0.000]
```

(This is the actual output of the [Complete Walkthrough](#7-complete-walkthrough-windows-tnl-sdk-790) — a deliberately tiny two-speaker corpus built from the SDK's sample audio, so the percentages move in steps of 25% and aren't meaningful in themselves. Your elapsed time will differ.)

- **sv-threshold** — the threshold the models were scored at: the value stored in the enrolled models, or your `--sv-threshold` override. If the enrolled models store different values, all of them are listed.
- **FRR** — false rejects ÷ target trials, for that condition.
- **IAR** — imposter accepts ÷ imposter trials, for that condition.
- **FA/day** — false accepts in the noise audio ÷ model-hours × 24. FAR doesn't depend on the test condition, so it repeats on every row.
- **[low-high]** — the 95% confidence interval for each figure, from bootstrapping ([8.3](#83-confidence-intervals)). The true rate for this population of speakers is likely to be inside it. Leave the intervals off with `--bootstrap 0`.

The raw counts are shown in parentheses — always look at them alongside the percentage, since `0.00% (0/4)` and `0.00% (0/4000)` mean very different things. Here the snr20 interval of 0–50% shows how little two speakers can tell you.

`summary.csv` has the same figures, with each interval in a pair of `_ci_low` and `_ci_high` columns.

Even this toy run shows the expected pattern: noise raises FRR. Re-running the same corpus with `--sv-threshold 0.9` raises the clean FRR from 0% to 50% (2/4).

---

## 7. Complete Walkthrough (Windows, TNL SDK 7.9.0)

This section runs the whole procedure in PowerShell on audio that ships with the SDK, so you can confirm the tools and script work before collecting your own data. The SDK's `data/enrollments/` folder contains recordings of a few speakers saying a few phrases, named `<phrase>-<speaker>-<take>.wav`. The phrase "armadillo" has two speakers (1 and 6) with six takes each — enough for a two-speaker smoke test, not a real benchmark.

Examples assume the SDK is installed at `C:\Sensory\TrulyNaturalSDK\7.9.0` — substitute your actual install path. Run everything from a working folder of your choice containing `enrolled_benchmark.py`.

**1. Build the corpus** — takes 0–3 for enrollment, takes 4–5 for testing, and the SDK's two sample audio files as (very short) FAR noise:

```powershell
$sdk = "C:\Sensory\TrulyNaturalSDK\7.9.0"
foreach ($spk in 1, 6) {
    $dir = "corpus\speakers\spk$spk"
    New-Item -ItemType Directory -Force "$dir\enroll", "$dir\test\clean" | Out-Null
    0..3 | ForEach-Object { Copy-Item "$sdk\data\enrollments\armadillo-$spk-$_.wav" "$dir\enroll\" }
    4..5 | ForEach-Object { Copy-Item "$sdk\data\enrollments\armadillo-$spk-$_.wav" "$dir\test\clean\" }
}
New-Item -ItemType Directory -Force corpus\noise | Out-Null
Copy-Item "$sdk\data\audio\*.wav" corpus\noise\
```

**2. Add noise conditions** — using the SDK's music sample as the noise source:

```powershell
python enrolled_benchmark.py noisify --corpus corpus `
    --noise "$sdk\data\audio\voice-genie-music.wav" --snr 20,10,5
```

**3. Enroll and score:**

```powershell
python enrolled_benchmark.py run --sdk $sdk `
    --enroller "$sdk\model\udt-universal-3.67.1.0.snsr" `
    --corpus corpus --out results
```

The output should match [6.4](#64-reading-the-output) exactly — the run is deterministic.

---

## 8. Step 5 — Interpreting Results

### 8.1 Choosing an sv-threshold

There is no single right `sv-threshold`; it depends on what matters more for the product:

- **Voice biometrics / security** — prioritize low IAR, and accept a higher FRR.
- **Personalized wake word** ("respond to me, not the TV or my family") — prioritize low FRR and FAR, and accept a higher IAR.

Start with the `sv-threshold` stored in the enrolled model — the script uses it unless you pass `--sv-threshold`. If the results miss your targets, re-run the same corpus with `--sv-threshold` set higher (to lower IAR and FAR) or lower (to lower FRR), and compare. Change the value in small steps and keep each run's `summary.csv`. To see what threshold your enrolled model starts with, query it — always pass an audio file, since `snsr-eval` with no file listens to the live microphone and won't exit on its own:

```bash
snsr-eval -t results/models/spk1.snsr -q sv-threshold corpus/speakers/spk1/test/clean/armadillo-1-4.wav
```

```
sv-threshold = 0.49
   135    765 spk1
```

The second line is just the detection in the audio file you passed.

Once you've chosen a value, set it on the session in your application exactly as you set it in the benchmark. On THF-Micro, remember the application has to compare `svScore` against `SvThreshold` itself — see [Creating and Using Enrolled Models §5.4](enrolled-models.md#54-biometric-scoring-thftnl-vs-thf-micro).

### 8.2 Recognition Sensitivity vs. Verification

`sv-threshold` only controls the verification step. Recognition sensitivity is tuned separately — with `operating-point` for enrolled models built by newer enroller models (such as `udt-universal-3.67.1.0.snsr`, operating points 6–14, default 10), or `score-offset` for older ones. See [Creating and Using Enrolled Models §5.5](enrolled-models.md#55-recognition-sensitivity-operating-points-and-score-offset).

If FRR is too high even at a low `sv-threshold`, the recognition step is rejecting the phrase before verification sees it — try a more accepting operating point (or a positive `score-offset`) and re-run. Pass it to `snsr-eval` with `-s operating-point=<n>`; to use it with the script, add the setting to the `snsr-eval` command in `evaluate()`. Change one setting at a time so you can tell which one moved the numbers.

Recognition is speaker-biased on its own — the model was trained on one speaker's voice — so some imposters are rejected before verification runs, even at `sv-threshold=0`. Don't read IAR at threshold `0` as "IAR without biometrics."

### 8.3 Confidence Intervals

Every FRR, IAR, and FAR figure is an estimate from a sample of speakers, and a different set of speakers would give somewhat different numbers. The script shows how much they could vary with a 95% confidence interval for each figure, computed by **bootstrapping**:

1. Draw a new set of speakers, the same size as the original, at random **with replacement** — some speakers are drawn more than once, others not at all.
2. Recompute FRR, IAR, and FAR from just those speakers' results.
3. Repeat 1,000 times (`--bootstrap`), and report the range that holds the middle 95% of the recomputed values.

The script resamples whole **speakers**, not individual recordings, because one speaker's recordings and their model's results aren't independent of each other: a speaker who enrolled badly tends to have many false rejects at once. Resampling recordings would ignore that and give intervals that are too narrow. The resampling is seeded, so the same results always produce the same intervals, and it only takes a few seconds even for 100 speakers.

Read the intervals alongside the figures:

- **Wide intervals mean too few speakers.** If two products' or settings' intervals overlap heavily, the benchmark can't tell them apart; add speakers before drawing a conclusion.
- **FAR intervals are usually the widest.** A handful of enrollments tend to cause most of the false accepts, so whether they're drawn swings the result a lot.
- **Like the rule of three below, the intervals assume every model hears independent noise.** All models hear the same noise, so the FAR interval is optimistic.

#### When You See Zero Errors

A bootstrap interval can't help when there are no errors at all: every resample also has zero errors, so the interval is `[0.00-0.00]`, as in the walkthrough output. A result of zero errors doesn't mean the true rate is zero — only that it's too small to observe with that much data. A useful rule of thumb (the "rule of three") gives the 95% upper bound when you see **zero** errors in N trials (or N model-days of noise): about **3 / N**.

| Observation | 95% upper bound |
|---|---|
| 0 false accepts in 110 model-hours (4.6 model-days) | ≈ 3 / 4.6 = 0.65 FA/day |
| 0 imposter accepts in 750 trials | ≈ 3 / 750 = 0.4% IAR |
| 0 false rejects in 100 trials | ≈ 3 / 100 = 3% FRR |

Use this to decide whether you have enough data for the claim you want to make. For example, to show FAR is below 0.1 FA/day with zero observed false accepts, you need about 3 / 0.1 = 30 model-days (720 model-hours) of noise. With 50 speakers:

| Hours of noise | Model-days (50 models) | 95% upper bound with 0 false accepts |
|---|---|---|
| 2 | 4.2 | ≈ 0.72 FA/day |
| 15 (minimum) | 31 | ≈ 0.096 FA/day |
| 150 (recommended) | 312 | ≈ 0.0096 FA/day |

Treat these bounds as optimistic. The rule of three assumes independent trials, but every model hears the same noise, so a stretch of noise that sounds like the phrase can trigger many models at once. That's another reason more hours of *different* noise matter more than more models listening to the same noise ([3.4](#34-non-target-audio-for-far)).

### 8.4 Report Every Speaker

It's tempting to drop the few speakers with the worst results — someone with a 40% FRR probably had a poor enrollment, and in real life they might simply enroll again. Removing them can make the headline numbers look much better. Don't do it:

- **Real users don't always re-enroll.** Many people who have a bad experience decide the feature doesn't work and stop using it. Those users are part of what the product will deliver, so they belong in the results.
- **Any cutoff is arbitrary.** Excluding the worst 5% instead of 10%, or every speaker above 30% FRR instead of 40%, changes the result, and the choice then has to be justified and reported alongside it. Results with different cutoffs can't be compared.
- **The enrollment checks are the filter.** Enrollment already rejects recordings that are too noisy, clipped, or inconsistent ([Creating and Using Enrolled Models §4.1](enrolled-models.md#41-the-two-step-enrollment-process)). Every speaker who gets through enrollment is a speaker the product would accept, so report all of them, including those with a high FRR or FAR.

If a few speakers dominate the errors, that's a finding worth reporting in its own right — look at their enrollment recordings to see what went wrong, and consider whether the product should guide users to re-enroll. The confidence intervals ([8.3](#83-confidence-intervals)) already reflect how much these speakers swing the result.

### 8.5 Example: Synthetic 55-Speaker Results

For reference, this is a benchmark of one enrolled wake word phrase on Sensory-generated **synthetic** speakers — 55 speakers, each with 4 enrollment recordings and its own test recordings, noise added digitally:

| Test condition (SNR) | FRR | IAR |
|---|---|---|
| Clean | 9.83% | 3.07% |
| 30 dB | 5.98% | 4.78% |
| 20 dB | 5.98% | 5.16% |
| 10 dB | 7.84% | 3.33% |
| 5 dB | 13.94% | 1.88% |

FAR was **0 false accepts over 110 model-hours** (55 models × a 2-hour conversational noise file) — under about 0.65 FA/day by the rule of three ([8.3](#when-you-see-zero-errors)). Two hours is well below the 15-hour minimum in [3.4](#34-non-target-audio-for-far), so this FAR is only a rough indication.

Treat the IAR figures with caution: as described in [3.5](#35-real-vs-synthetic-speakers), a limited cloned-voice library can make synthetic speakers more alike than real ones, and real-speaker tests have shown IARs much closer to zero. This is exactly why a benchmark should include real speakers before the numbers are used for a product decision.

---

## 9. Common Pitfalls

- **Too few speakers.** A test with a handful of colleagues says little about the population — use at least 50 speakers, and 100+ where possible ([3.1](#31-how-much-data-is-enough)).
- **Testing on enrollment recordings.** Scoring a model against the audio it was trained on makes FRR look artificially low ([3.3](#33-test-recordings)).
- **Imposters saying a different phrase.** IAR measures other speakers saying the **same** target phrase. A different phrase is mostly rejected by the recognition step and makes IAR look better than it is — keep each corpus to one phrase.
- **Target phrase in the FAR noise.** A genuine occurrence of the phrase in TV or conversation audio is counted as a false accept — screen the noise first ([3.4](#34-non-target-audio-for-far)).
- **Digital silence in the recordings.** Runs of zero samples never occur on a real device and skew results, especially in the `clean` condition — run `check` before benchmarking ([Step 2](#4-step-2--lay-out-the-corpus)).
- **Testing live instead of in simulation.** Live testing is slow, hard to repeat, and gives the same result as simulating the same audio ([1.3](#13-simulate-dont-test-live)).
- **Over-the-air playback through a generic speaker.** Use clean, unmodified recordings through a speaker that accurately reproduces a human voice ([3.6](#36-if-you-must-record-over-the-air)).
- **Relying only on synthetic speakers.** Synthetic data may overstate IAR; confirm with real speakers ([3.5](#35-real-vs-synthetic-speakers)).
- **Excluding the worst-performing speakers.** Every speaker who passes enrollment is a real user the product would accept; dropping the ones with the highest FRR or FAR flatters the results and makes them depend on an arbitrary cutoff ([8.4](#84-report-every-speaker)).
- **Benchmarking with a different enroller than the product uses.** Results only carry over to the enroller model — and the enrollment settings, such as `req-enroll` and `accuracy` — that the device actually uses.
- **Putting `-s` before `-t` on the `snsr-eval` command line.** `snsr-eval` rejects it with `ERROR: set -t task before -f or -s options` — the task must come first.
- **Running `snsr-eval -q` with no audio file.** With no file, `snsr-eval` listens to the live microphone and runs until you press Ctrl-C ([8.1](#81-choosing-an-sv-threshold)).
- **Reading "0 errors" as "0% error rate."** Check the rule-of-three bound before drawing conclusions ([8.3](#when-you-see-zero-errors)).
- **Comparing results without their confidence intervals.** A difference between two runs means little if their intervals overlap heavily ([8.3](#83-confidence-intervals)).

---

## 10. Quick Reference — Command Summary

```bash
# Enroll one speaker from 4 quiet recordings
spot-enroll -t udt-universal-3.67.1.0.snsr -o spk1.snsr +spk1 \
    enroll-1.wav enroll-2.wav enroll-3.wav enroll-4.wav

# Score one model against a list of files, one result line per file
snsr-eval -t spk1.snsr -i list.txt -o results.txt

# Same, overriding the model's sv-threshold (-s must come after -t)
snsr-eval -t spk1.snsr -s sv-threshold=0.6 -i list.txt -o results.txt

# Show detections with their speaker-verification scores
snsr-eval -t spk1.snsr -v test-1.wav

# Query a model's default sv-threshold (always pass an audio file)
snsr-eval -t spk1.snsr -q sv-threshold test-1.wav

# Check enrollment and test recordings against the audio requirements
python enrolled_benchmark.py check --corpus my-corpus

# Add noise from a folder of real noise recordings to every test/clean recording
# at 30/20/10/5 dBA SNR
python enrolled_benchmark.py noisify --corpus my-corpus --noise mix-noise --snr 30,20,10,5

# Enroll every speaker and score at each model's own sv-threshold
python enrolled_benchmark.py run --sdk <sdk-root> \
    --enroller <sdk-root>/model/udt-universal-3.67.1.0.snsr \
    --corpus my-corpus --out results -j 8

# Re-score the same models at a different sv-threshold
python enrolled_benchmark.py run --sdk <sdk-root> \
    --enroller <sdk-root>/model/udt-universal-3.67.1.0.snsr \
    --corpus my-corpus --out results -j 8 --sv-threshold 0.6

# Quick check: clean condition only, no FAR noise
python enrolled_benchmark.py run --sdk <sdk-root> \
    --enroller <sdk-root>/model/udt-universal-3.67.1.0.snsr \
    --corpus my-corpus --out results -j 8 --conditions clean --skip-noise
```

---

## 11. References

- TNL SDK 7.9 Docs: https://doc.sensory.com/tnl/7.9/
- `spot-enroll` reference: https://doc.sensory.com/tnl/7.9/tools/spot-enroll/
- `snsr-eval` reference: https://doc.sensory.com/tnl/7.9/tools/snsr-eval/
- Enrollment model types: https://doc.sensory.com/tnl/7.9/models/types/enroll/
- Setting keys reference: https://doc.sensory.com/tnl/7.9/api/setting-keys/
- THF-Micro Docs: https://doc.sensory.com/thf-micro/latest/
- [Creating and Using Enrolled Models](enrolled-models.md) — enrollment, `sv-threshold`, operating points, and `score-offset`
- [How to Save Debug Audio and Data in THF/TNL](debug-audio-capture.md) — capturing on-device audio to reuse as simulation test data
- [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md) — measuring an enrolled model's runtime cost, as opposed to its accuracy
