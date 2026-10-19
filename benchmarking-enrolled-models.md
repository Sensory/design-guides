# Benchmarking Enrolled Wake Word Models (FRR, IAR, FAR)

**Product:** TrulyHandsfree (THF) and TrulyNatural (TNL) SDKs  
**Models:** Enrolled models produced by the enroller task models (`udt-*.snsr`, `eft-*.snsr`)  
**Version:** THF / TNL 7.8.0+ (examples run on TNL 7.9.0)  
**Document Version:** 1.0.0  
**Last Modified:** 2026-10-19  
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
   - [8.3 Confidence When You See Zero Errors](#83-confidence-when-you-see-zero-errors)
   - [8.4 Example: Synthetic 55-Speaker Results](#84-example-synthetic-55-speaker-results)
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
| Non-target audio for FAR | 2 hours | As much as you can — 24 hours or more |

Because every speaker serves as an imposter for every other speaker, the number of imposter trials grows with the square of the speaker count: 50 speakers × 15 test recordings each gives 750 target trials but 36,750 imposter trials. More speakers is what improves IAR confidence the most.

### 3.2 Enrollment Recordings

Each speaker enrolls by saying the target phrase **4 times in a quiet environment** — a home or office with no one talking in the background and no fans or other noise running. These recordings train that speaker's enrolled model, exactly as an end user would on the device. Follow the recording guidelines in [Creating and Using Enrolled Models §4.2](enrolled-models.md#42-recording-guidelines), including recording some takes with trailing context speech if your product's enrollment flow does so.

### 3.3 Test Recordings

Each speaker also records **10–20 test recordings** of the same target phrase, separate from their enrollment recordings. Never reuse an enrollment recording as a test recording — the model has already been trained on it, so it will make FRR look better than it really is.

To make the test recordings representative of how the product will actually be used:

- Record them on a different day or in a different session from enrollment if possible, so they capture natural variation in the speaker's voice.
- Vary distance, loudness, and direction the way real use will — for example, a laptop user may be at arm's length or across a desk. Sensory doesn't prescribe fixed distances or angles; match your product's use case, and record each distinct setup as its own condition (see [Step 2](#4-step-2--lay-out-the-corpus)) so you can compare them.
- Cover noise by either recording in noisy conditions or — more commonly and more reproducibly — recording clean and **adding noise digitally afterwards** at fixed signal-to-noise ratios (see [Step 3](#5-step-3--add-noise-conditions)).

### 3.4 Non-Target Audio for FAR

FAR audio is the easiest data to collect, since it doesn't need volunteers: any long recording of realistic sound that doesn't contain the target phrase works — conversations, TV and radio, music, room and household noise. The challenge is quantity. False accepts are rare events, so you need many hours of audio to see enough of them (or to be confident there are few — see [8.3](#83-confidence-when-you-see-zero-errors)). Use **at least 2 hours**; more is better.

Screen the audio for the target phrase (and close soundalikes, if you want to keep them out) before using it — a genuine occurrence of the phrase in the noise would be scored as a false accept.

### 3.5 Real vs. Synthetic Speakers

Sensory can generate **synthesized (voice-cloned) enrollment and test data** for a target phrase, which makes it practical to reach 50+ speakers quickly. Synthetic data is useful, but it shouldn't be the only data:

- A cloned-voice library may not be as diverse as real speakers. Similar-sounding synthetic voices make it easier for one "speaker" to pass as another, which can inflate IAR above what you'd see in the field. In Sensory's experience, tests with real speakers have shown IARs much closer to zero than synthetic tests of the same model.
- Some of the benchmark **must** use real recordings of actual people speaking. Use synthetic data to scale up and to compare configurations against each other; use real speakers to confirm the absolute numbers.

Contact your Sensory FAE about synthetic data for your phrase.

### 3.6 If You Must Record Over the Air

If the test audio has to be played back over the air into the device — for example, to capture it through the device's own microphones — play back **unmodified, clean recordings** through a loudspeaker that accurately reproduces a human voice (such as an artificial mouth or a head-and-torso simulator), not a generic speaker. A small consumer speaker colors the audio and can change the results. Capture what the device hears once, then score the captured audio in simulation ([1.3](#13-simulate-dont-test-live)).

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

---

## 5. Step 3 — Add Noise Conditions

To test in noise, mix noise into each clean test recording at fixed signal-to-noise ratios. Sensory typically uses **30, 20, 10, and 5 dB SNR**, but use whatever range matches your product. The script's `noisify` subcommand reads every `test/clean/*.wav` file, mixes in a random segment of a noise file at each SNR, and writes the results to `test/snr30/`, `test/snr20/`, and so on:

```bash
python enrolled_benchmark.py noisify --corpus my-corpus --noise babble.wav --snr 30,20,10,5
```

- The noise file must be 16-bit mono PCM at the same sample rate as the corpus, and longer than any test recording.
- SNR is computed from the average power of the whole test recording, including any leading and trailing silence. Trim long silences from the test recordings first if you want SNR to reflect the speech alone.
- The random noise segment is seeded, so re-running `noisify` reproduces the same noisy corpus.
- Use a different noise recording here from the FAR noise in `noise/`, so the two tests stay independent.

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
  noisify  mix noise into clean test recordings at fixed SNRs
  run      enroll every speaker, then score every model
"""

import argparse
import array
import csv
import math
import random
import subprocess
import sys
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

def noisify(args) -> None:
    corpus = Path(args.corpus)
    _, noise = read_pcm(Path(args.noise))
    noise_power = power(noise)
    rng = random.Random(0)  # fixed seed: the noisy corpus is reproducible
    snrs = [int(s) for s in args.snr.split(",")]

    for clean in sorted(corpus.glob("speakers/*/test/clean/*.wav")):
        params, speech = read_pcm(clean)
        start = rng.randrange(0, max(len(noise) - len(speech), 1))
        segment = noise[start:start + len(speech)]
        for snr in snrs:
            gain = math.sqrt(power(speech) / (noise_power * 10 ** (snr / 10)))
            mixed = array.array("h", (
                max(-32768, min(32767, int(s + gain * n)))
                for s, n in zip(speech, segment)))
            out = clean.parent.parent / f"snr{snr}" / clean.name
            out.parent.mkdir(exist_ok=True)
            with wave.open(str(out), "wb") as w:
                w.setparams(params)
                w.writeframes(mixed.tobytes())
    print(f"wrote {', '.join(f'snr{s}' for s in snrs)} test conditions")


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


def run(args) -> None:
    sdk = Path(args.sdk)
    corpus = Path(args.corpus)
    out = Path(args.out)
    (out / "models").mkdir(parents=True, exist_ok=True)

    speakers = sorted(p for p in (corpus / "speakers").iterdir() if p.is_dir())
    tests = [(spk.name, f.parent.name, f)
             for spk in speakers for f in sorted(spk.glob("test/*/*.wav"))]
    noise = sorted((corpus / "noise").glob("*.wav"))
    noise_hours = sum(wav_seconds(f) for f in noise) / 3600
    # clean first, then snrNN from least to most noisy
    conditions = sorted({cond for _, cond, _ in tests},
                        key=lambda c: -float(c[3:]) if c.startswith("snr") else -math.inf)

    def score(spk: Path):
        """Enroll one speaker if needed, then score their model on everything."""
        model = out / "models" / f"{spk.name}.snsr"
        if not model.exists():
            print(f"enrolling {spk.name}")
            enroll(sdk, args.enroller, spk, model)
        threshold = None
        if args.sv_threshold is None:
            threshold = sv_threshold(sdk, model, tests[0][2])
        work = out / "work" / spk.name  # each job needs its own list/results files
        work.mkdir(parents=True, exist_ok=True)
        counts = evaluate(sdk, model, [f for _, _, f in tests] + noise, work,
                          args.sv_threshold)
        return spk.name, threshold, counts

    # Each job runs its own spot-enroll/snsr-eval processes, so -j N uses N cores.
    # Results are identical for any -j: each model is still scored on the same audio.
    with ThreadPoolExecutor(max_workers=args.jobs) as pool:
        results = list(pool.map(score, speakers))

    # tally[condition] = [target, fr, imposter, ia]
    tally = {c: [0, 0, 0, 0] for c in conditions}
    false_accepts = 0
    model_thresholds = set()

    for name, threshold, counts in results:
        if threshold is not None:
            model_thresholds.add(threshold)
        for owner, cond, f in tests:
            hit = counts.get(str(f), 0) > 0
            row = tally[cond]
            if owner == name:
                row[0] += 1
                row[1] += not hit
            else:
                row[2] += 1
                row[3] += hit
        false_accepts += sum(counts.get(str(f), 0) for f in noise)

    if args.sv_threshold is None:
        threshold = "/".join(sorted(model_thresholds)) + " (model default)"
    else:
        threshold = f"{args.sv_threshold} (override)"
    model_hours = noise_hours * len(speakers)
    fa_day = 24 * false_accepts / model_hours if model_hours else float("nan")

    with open(out / "summary.csv", "w", newline="", encoding="utf-8") as fh:
        writer = csv.writer(fh)
        writer.writerow(["sv_threshold", "condition", "target_trials", "false_rejects",
                         "FRR_pct", "imposter_trials", "imposter_accepts", "IAR_pct",
                         "false_accepts", "model_hours", "FA_per_day"])
        print(f"\n{len(speakers)} speakers, {noise_hours:.2f} h noise "
              f"({model_hours:.2f} model-hours), sv-threshold {threshold}\n")
        print(f"{'condition':<10} {'FRR':>16} {'IAR':>16} {'FA/day':>8}")
        for c in conditions:
            n_t, fr, n_i, ia = tally[c]
            frr = 100 * fr / n_t if n_t else float("nan")
            iar = 100 * ia / n_i if n_i else float("nan")
            writer.writerow([threshold, c, n_t, fr, f"{frr:.2f}", n_i, ia, f"{iar:.2f}",
                             false_accepts, f"{model_hours:.2f}", f"{fa_day:.3f}"])
            print(f"{c:<10} {frr:6.2f}% ({fr:>3}/{n_t:<4}) "
                  f"{iar:6.2f}% ({ia:>3}/{n_i:<4}) {fa_day:8.3f}")
    print(f"\nwrote {out / 'summary.csv'}")


def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__,
                                     formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = parser.add_subparsers(dest="command", required=True)

    p = sub.add_parser("noisify", help="mix noise into test/clean at fixed SNRs")
    p.add_argument("--corpus", required=True)
    p.add_argument("--noise", required=True, help="16-bit mono noise WAV, same rate as corpus")
    p.add_argument("--snr", default="30,20,10,5", help="comma-separated SNRs in dB")
    p.set_defaults(func=noisify)

    p = sub.add_parser("run", help="enroll and score every speaker")
    p.add_argument("--sdk", required=True, help="SDK install root (contains bin/)")
    p.add_argument("--enroller", required=True, help="enroller task model (udt-*/eft-*.snsr)")
    p.add_argument("--corpus", required=True)
    p.add_argument("--out", required=True, help="output folder for models and summary.csv")
    p.add_argument("--sv-threshold", type=float,
                   help="override the sv-threshold stored in each enrolled model")
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
| `-j`, `--jobs` | Number of speakers to enroll and score at once (default `1`). Set it to about the number of CPU cores — see below |

Enrolled models are saved to `<out>/models/<speaker>.snsr` and reused if they already exist, so you can re-run with `--sv-threshold` without re-enrolling. Delete `<out>/models/` to re-enroll — for example, after changing the enroller model or the enrollment recordings.

Each run overwrites `<out>/summary.csv`, so copy it somewhere else first if you want to compare runs.

Each speaker's model is evaluated against every test recording in the corpus and all of the FAR noise, so run time grows with the square of the number of speakers, plus the number of speakers × hours of noise. Enrollment adds a fixed cost per speaker, roughly 15 seconds each on a desktop PC with `udt-universal-3.67.1.0.snsr`. As a rough guide for scoring, one enrolled model processed an hour of audio in about 4 seconds on the same PC, so 24 hours of noise takes about 100 seconds per model, or about 85 minutes for 50 speakers with `-j 1`.

`-j` divides that time across CPU cores. Each job runs its own `spot-enroll` and `snsr-eval` processes for one speaker at a time, so set `-j` to about the number of cores on the machine. On an 8-core PC, a 16-speaker corpus with an hour of noise took 4 minutes 49 seconds with `-j 1` and 51 seconds with `-j 8`. The results are identical for any `-j` value: every model is still scored on exactly the same audio; it's just scored alongside other models instead of after them ([1.3](#13-simulate-dont-test-live)).

### 6.4 Reading the Output

The script prints a table and writes the same numbers to `<out>/summary.csv`:

```
2 speakers, 0.01 h noise (0.01 model-hours), sv-threshold 0.49 (model default)

condition               FRR              IAR   FA/day
clean        0.00% (  0/4   )   0.00% (  0/4   )    0.000
snr20       25.00% (  1/4   )   0.00% (  0/4   )    0.000
snr10       25.00% (  1/4   )   0.00% (  0/4   )    0.000
snr5        25.00% (  1/4   )   0.00% (  0/4   )    0.000
```

(This is the actual output of the [Complete Walkthrough](#7-complete-walkthrough-windows-tnl-sdk-790) — a deliberately tiny two-speaker corpus built from the SDK's sample audio, so the percentages move in steps of 25% and aren't meaningful in themselves.)

- **sv-threshold** — the threshold the models were scored at: the value stored in the enrolled models, or your `--sv-threshold` override. If the enrolled models store different values, all of them are listed.
- **FRR** — false rejects ÷ target trials, for that condition.
- **IAR** — imposter accepts ÷ imposter trials, for that condition.
- **FA/day** — false accepts in the noise audio ÷ model-hours × 24. FAR doesn't depend on the test condition, so it repeats on every row.

The raw counts are shown in parentheses — always look at them alongside the percentage, since `0.00% (0/4)` and `0.00% (0/4000)` mean very different things.

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

### 8.3 Confidence When You See Zero Errors

A result of zero errors doesn't mean the true rate is zero — only that it's too small to observe with that much data. A useful rule of thumb (the "rule of three") gives the 95% upper bound when you see **zero** errors in N trials (or N model-days of noise): about **3 / N**.

| Observation | 95% upper bound |
|---|---|
| 0 false accepts in 110 model-hours (4.6 model-days) | ≈ 3 / 4.6 = 0.65 FA/day |
| 0 imposter accepts in 750 trials | ≈ 3 / 750 = 0.4% IAR |
| 0 false rejects in 100 trials | ≈ 3 / 100 = 3% FRR |

Use this to decide whether you have enough data for the claim you want to make. For example, to show FAR is below 0.1 FA/day with zero observed false accepts, you need about 3 / 0.1 = 30 model-days (720 model-hours) of noise. With 50 speakers that's about 14.4 hours of noise audio; 2 hours (100 model-hours, 4.2 model-days) only shows FAR is below about 0.72 FA/day.

### 8.4 Example: Synthetic 55-Speaker Results

For reference, this is a benchmark of one enrolled wake word phrase on Sensory-generated **synthetic** speakers — 55 speakers, each with 4 enrollment recordings and its own test recordings, noise added digitally:

| Test condition (SNR) | FRR | IAR |
|---|---|---|
| Clean | 9.83% | 3.07% |
| 30 dB | 5.98% | 4.78% |
| 20 dB | 5.98% | 5.16% |
| 10 dB | 7.84% | 3.33% |
| 5 dB | 13.94% | 1.88% |

FAR was **0 false accepts over 110 model-hours** (55 models × a 2-hour conversational noise file) — under about 0.65 FA/day by the rule of three ([8.3](#83-confidence-when-you-see-zero-errors)).

Treat the IAR figures with caution: as described in [3.5](#35-real-vs-synthetic-speakers), a limited cloned-voice library can make synthetic speakers more alike than real ones, and real-speaker tests have shown IARs much closer to zero. This is exactly why a benchmark should include real speakers before the numbers are used for a product decision.

---

## 9. Common Pitfalls

- **Too few speakers.** A test with a handful of colleagues says little about the population — use at least 50 speakers, and 100+ where possible ([3.1](#31-how-much-data-is-enough)).
- **Testing on enrollment recordings.** Scoring a model against the audio it was trained on makes FRR look artificially low ([3.3](#33-test-recordings)).
- **Imposters saying a different phrase.** IAR measures other speakers saying the **same** target phrase. A different phrase is mostly rejected by the recognition step and makes IAR look better than it is — keep each corpus to one phrase.
- **Target phrase in the FAR noise.** A genuine occurrence of the phrase in TV or conversation audio is counted as a false accept — screen the noise first ([3.4](#34-non-target-audio-for-far)).
- **Testing live instead of in simulation.** Live testing is slow, hard to repeat, and gives the same result as simulating the same audio ([1.3](#13-simulate-dont-test-live)).
- **Over-the-air playback through a generic speaker.** Use clean, unmodified recordings through a speaker that accurately reproduces a human voice ([3.6](#36-if-you-must-record-over-the-air)).
- **Relying only on synthetic speakers.** Synthetic data may overstate IAR; confirm with real speakers ([3.5](#35-real-vs-synthetic-speakers)).
- **Benchmarking with a different enroller than the product uses.** Results only carry over to the enroller model — and the enrollment settings, such as `req-enroll` and `accuracy` — that the device actually uses.
- **Putting `-s` before `-t` on the `snsr-eval` command line.** `snsr-eval` rejects it with `ERROR: set -t task before -f or -s options` — the task must come first.
- **Running `snsr-eval -q` with no audio file.** With no file, `snsr-eval` listens to the live microphone and runs until you press Ctrl-C ([8.1](#81-choosing-an-sv-threshold)).
- **Reading "0 errors" as "0% error rate."** Check the rule-of-three bound before drawing conclusions ([8.3](#83-confidence-when-you-see-zero-errors)).

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

# Add noise to every test/clean recording at 30/20/10/5 dB SNR
python enrolled_benchmark.py noisify --corpus my-corpus --noise babble.wav --snr 30,20,10,5

# Enroll every speaker and score at each model's own sv-threshold
python enrolled_benchmark.py run --sdk <sdk-root> \
    --enroller <sdk-root>/model/udt-universal-3.67.1.0.snsr \
    --corpus my-corpus --out results -j 8

# Re-score the same models at a different sv-threshold
python enrolled_benchmark.py run --sdk <sdk-root> \
    --enroller <sdk-root>/model/udt-universal-3.67.1.0.snsr \
    --corpus my-corpus --out results -j 8 --sv-threshold 0.6
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
