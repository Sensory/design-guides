# Measuring Word Error Rate (WER) for TrulyNatural STT Models

**Product:** TrulyNatural (TNL) SDK
**Models:** TNL STT models (e.g. `stt-enUS-general-*.snsr`, `stt-enUS-automotive-*.snsr`)
**Version:** TNL SDK 7.9.0+ recommended (`snsr-eval-batch`); 7.8.0 also supported with the caveats in [2.1](#21-sdktool-version)
**Document Version:** 1.0.0
**Audience:** Field Application Engineers, Customer Integration Engineers
**Status:** Released under NDA — Do not distribute

*Copyright © 2026 Sensory Inc. All rights reserved. This document is confidential and proprietary to Sensory Inc. It may not be reproduced, distributed, or disclosed to any third party without prior written permission from Sensory Inc.*

---

## Table of Contents

1. [Overview](#1-overview)
   - [1.1 What WER Measures](#11-what-wer-measures)
   - [1.2 Why Measure It Per-Deployment](#12-why-measure-it-per-deployment)
2. [Prerequisites](#2-prerequisites)
   - [2.1 SDK/Tool Version](#21-sdktool-version)
   - [2.2 STT Model](#22-stt-model)
   - [2.3 Test Audio Corpus](#23-test-audio-corpus)
3. [Step 1 — Obtain a Standalone Base STT Model File](#3-step-1--obtain-a-standalone-base-stt-model-file)
   - [3.1 Downloading the Model Directly](#31-downloading-the-model-directly)
   - [3.2 Extracting the STT Model from an Assembled Pipeline](#32-extracting-the-stt-model-from-an-assembled-pipeline)
4. [Step 2 — Prepare the Test Corpus](#4-step-2--prepare-the-test-corpus)
   - [4.1 Corpus Layout and Reference Transcripts](#41-corpus-layout-and-reference-transcripts)
   - [4.2 Converting FLAC to WAV (SDK 7.8.0 Only)](#42-converting-flac-to-wav-sdk-780-only)
5. [Step 3 — Build the CSV Manifest](#5-step-3--build-the-csv-manifest)
   - [5.1 Manifest Format](#51-manifest-format)
   - [5.2 Generating the Manifest with a Script](#52-generating-the-manifest-with-a-script)
6. [Step 4 — Run snsr-eval-batch](#6-step-4--run-snsr-eval-batch)
   - [6.1 Command](#61-command)
   - [6.2 Flags Reference](#62-flags-reference)
   - [6.3 Reading the Output](#63-reading-the-output)
7. [Complete Walkthrough (Windows, TNL SDK 7.9.0)](#7-complete-walkthrough-windows-tnl-sdk-790)
   - [7.1 Download the STT Model](#71-download-the-stt-model)
   - [7.2 Download and Extract the Test Corpus](#72-download-and-extract-the-test-corpus)
   - [7.3 Build the CSV Manifest](#73-build-the-csv-manifest)
   - [7.4 Run snsr-eval-batch and Review Results](#74-run-snsr-eval-batch-and-review-results)
8. [Step 5 — Interpreting Results](#8-step-5--interpreting-results)
   - [8.1 Normalization Caveats](#81-normalization-caveats)
   - [8.2 Multithreading with -j](#82-multithreading-with--j)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Quick Reference — Command Summary](#10-quick-reference--command-summary)
11. [References](#11-references)

---

## 1. Overview

### 1.1 What WER Measures

Word Error Rate (WER) is the standard accuracy metric for speech recognition. It compares a model's hypothesis transcript against a known-correct reference transcript for the same audio, using word-level edit distance:

```
WER = (Substitutions + Insertions + Deletions) / (Total Reference Words)
```

- **Substitution** — the model output a wrong word in place of the reference word.
- **Insertion** — the model output an extra word not present in the reference.
- **Deletion** — the model failed to output a word that was present in the reference.

WER is reported as a percentage; lower is better. Unlike RTF or memory usage (see [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md)), WER is a measure of recognition *accuracy*, not runtime resource cost — the two should be evaluated separately and neither substitutes for the other.

For languages without word delimiters (e.g. Japanese, Chinese), `snsr-eval-batch` automatically reports **character error rate (CER)** instead, using the same substitution/insertion/deletion accounting at the character level.

### 1.2 Why Measure It Per-Deployment

The WER figures Sensory publishes for a given Speech-to-Text (STT) model are measured against Sensory's own internal test sets and don't necessarily reflect a specific customer's acoustic conditions, vocabulary, or accent distribution. Re-measuring WER against a representative, customer-relevant audio corpus is the only way to know how a model will actually perform in a given deployment, and gives you a quantitative baseline to compare against when evaluating a model size change, a new SDK version, or a field accuracy complaint (see [How to Save Debug Audio and Data in THF/TNL](debug-audio-capture.md) for capturing real field audio to expand that corpus).

---

## 2. Prerequisites

### 2.1 SDK/Tool Version

This guide uses `snsr-eval-batch`, a dedicated batch-evaluation tool distinct from `snsr-eval` (used in [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md) for single-file profiling). `snsr-eval-batch` scores a model against many audio/reference pairs in one run and reports WER directly.

- **TNL SDK 7.9.0+ (recommended)** — `snsr-eval-batch` accepts WAV or FLAC audio directly.
- **TNL SDK 7.8.0** — also supported, but `snsr-eval-batch` in this version accepts **WAV audio only**. If your corpus is distributed as FLAC (as the corpus in [2.3](#23-test-audio-corpus) is), convert it to WAV first — see [4.2](#42-converting-flac-to-wav-sdk-780-only).

Ensure `snsr-eval-batch` is on your `PATH`. If you only have an assembled pipeline `.snsr` file rather than a standalone STT model, you'll also need `snsr-edit` — see [3.2](#32-extracting-the-stt-model-from-an-assembled-pipeline).

### 2.2 STT Model

Download the TNL STT model(s) you want to evaluate from the SDK model downloads page:

- SDK 7.9.0+: https://doc.sensory.com/tnl/7.9/models/downloads/
- SDK 7.8.0: https://doc.sensory.com/tnl/7.8/models/downloads/

Save the downloaded model into your SDK installation's `model/` directory, alongside the other models shipped with the SDK. This guide uses `stt-enUS-general-small-2.2.5-pnc.snsr` as its running example, saved as `model/stt-enUS-general-small-2.2.5-pnc.snsr` — the small size is used specifically because, unlike the medium model, it doesn't already ship with the SDK, so the download step in this guide is actually exercised rather than a no-op. The procedure is identical for any other STT model on the downloads page (other sizes, languages, or the automotive variants) — substitute the filename.

For models in the SDK 7.8.0 STT list, you'll need to extract the base STT model from an assembled `opt-vg-vad-stt-*.snsr` pipeline file before continuing, rather than using a standalone STT model directly — see [2.1](#21-sdktool-version) and [3.2](#32-extracting-the-stt-model-from-an-assembled-pipeline). A 7.8.0-listed model can be evaluated with the 7.9.0 version of `snsr-eval-batch` (newer tooling stays compatible with older models), but not the reverse — don't expect a 7.9.0 model to work with 7.8.0 tooling.

### 2.3 Test Audio Corpus

A labeled audio corpus (audio files with known-correct reference transcripts) is required. The team's shared WER test corpus is here:

https://drive.google.com/drive/u/1/folders/16P_SwmHJnJ4ls5J4NCW1FsWYMpwTEt0x

The corpus is organized into per-language folders named with two-letter ISO 639-1 language codes (e.g. `en` for English, `pl` for Polish, `id` for Indonesian), plus a handful of three-letter ISO 639-3 codes for languages without a 639-1 code (e.g. `yue` for Cantonese). See the [list of ISO 639 language codes](https://en.wikipedia.org/wiki/List_of_ISO_639_language_codes) if you need to look one up. Each language folder is further split by domain (e.g. `general`), containing zipped test sets.

This guide uses one specific test set throughout as its worked example:

```
en > general > stt-16kHz-en-general-quicktest.zip
```

This corpus provides audio as **FLAC files only**, each with a matching plain-text reference transcript file using a `.nrm` (normalized reference) extension — see [4.1](#41-corpus-layout-and-reference-transcripts). This means SDK 7.8.0 users must convert the entire corpus to WAV before running `snsr-eval-batch` — see [4.2](#42-converting-flac-to-wav-sdk-780-only).

---

## 3. Step 1 — Obtain a Standalone Base STT Model File

`snsr-eval-batch` needs a `.snsr` task file for the STT model on its own — not a full wake-word/VAD/STT pipeline. There are two ways to get one, depending on what you're starting from.

### 3.1 Downloading the Model Directly

If you downloaded a plain STT model per [2.2](#22-stt-model) (e.g. `stt-enUS-general-small-2.2.5-pnc.snsr`), it's already a standalone task file — skip to [Step 2](#4-step-2--prepare-the-test-corpus).

### 3.2 Extracting the STT Model from an Assembled Pipeline

If instead you only have an already-assembled pipeline model — for example `opt-vg-vad-stt-enUS-general-medium-2.4.3-pnc_66.snsr`, built from the `tpl-opt-spot-vad-lvcsr` template with a wake word and VAD ahead of the STT/LVCSR model (see [Benchmarking RTF and Avg/Max Memory Usage §3](benchmarking-rtf-memory.md#3-step-1--assemble-the-pipeline-model) for how such a pipeline gets built in the first place) — pull the STT model back out with `snsr-edit`:

```bash
snsr-edit -v -t opt-vg-vad-stt-enUS-general-medium-2.4.3-pnc_66.snsr \
    -e 2.0 stt-enUS-general-medium-2.4.3-pnc_66.snsr
```

- `-t` — the assembled pipeline task file to extract from.
- `-e 2.0 stt-enUS-general-medium-2.4.3-pnc_66.snsr` — extract element `2.0` (the STT/LVCSR element) into the given output filename.
- `-v` — verbose output, useful for confirming what got extracted.

**Naming convention:** by convention, the output filename is the assembled pipeline's own filename with the `opt-vg-vad-` prefix dropped — the remainder (`stt-enUS-general-medium-2.4.3-pnc_66.snsr` in this example) is the base STT model name. Following this convention for other assembled pipeline files keeps the extracted model's filename self-describing.

`stt-enUS-general-medium-2.4.3-pnc_66.snsr` can now be used as the task file in [Step 4](#6-step-4--run-snsr-eval-batch), exactly like a model downloaded directly.

---

## 4. Step 2 — Prepare the Test Corpus

### 4.1 Corpus Layout and Reference Transcripts

Download `stt-16kHz-en-general-quicktest.zip` (the test set identified in [2.3](#23-test-audio-corpus)) from the corpus library, then extract it under your SDK installation's `data/audio/` directory. The zip already contains a top-level `stt-16kHz-en-general-quicktest/` folder, so extract directly into `data/audio/` rather than into a folder of the same name — otherwise you'll end up with a duplicated, nested path:

```bash
unzip stt-16kHz-en-general-quicktest.zip -d data/audio/
```

This produces `data/audio/stt-16kHz-en-general-quicktest/`. Each audio file in the corpus (`.flac`, or `.wav` after conversion per [4.2](#42-converting-flac-to-wav-sdk-780-only)) has a matching reference transcript file with the same base name and a `.nrm` extension, containing the plain-text ground-truth transcript for that utterance. Filenames in this test set are 32-character hex hashes rather than anything descriptive, but the same-base-name pairing still holds:

```
data/audio/stt-16kHz-en-general-quicktest/
  6e54124861d89fbdd5e81137f37f008a.flac
  6e54124861d89fbdd5e81137f37f008a.nrm
  2eb3091c96a6be5ba3c076e05827f618.flac
  2eb3091c96a6be5ba3c076e05827f618.nrm
  ...
  info.yml
```

The `.nrm` extension is just this corpus's convention for "normalized reference" — `snsr-eval-batch` doesn't care about the reference file's extension, only that it's a plain-text file containing the expected transcript.

Each test-set folder also includes an `info.yml` metadata file (for this test set: `character_language: False`) alongside the audio/reference pairs. It's not itself an audio or reference file — the manifest script in [5.2](#52-generating-the-manifest-with-a-script) ignores it automatically, since it only scans for `.wav`/`.flac` — but it's a useful sanity check for whether a given test set should be scored as WER or CER (see [1.1](#11-what-wer-measures)).

### 4.2 Converting FLAC to WAV (SDK 7.8.0 Only)

`snsr-eval-batch` 7.9.0 accepts WAV or FLAC audio directly; 7.8.0 only accepts WAV. So skip this step on SDK 7.9.0+.

On SDK 7.8.0, this step is **required** — the corpus in [2.3](#23-test-audio-corpus) is FLAC-only. Convert each FLAC file to 16 kHz mono WAV first, using either SoX or ffmpeg. To batch-convert the corpus folder with SoX:

```bash
for f in data/audio/stt-16kHz-en-general-quicktest/*.flac; do
    sox "$f" -r 16000 -c 1 -b 16 "${f%.flac}.wav"
done
```

Or with ffmpeg:

```bash
for f in data/audio/stt-16kHz-en-general-quicktest/*.flac; do
    ffmpeg -i "$f" -ar 16000 -ac 1 -sample_fmt s16 "${f%.flac}.wav"
done
```

Keep the `.nrm` reference files as-is — only the audio needs converting, and the manifest script in [5.2](#52-generating-the-manifest-with-a-script) matches on base filename regardless of audio extension.

---

## 5. Step 3 — Build the CSV Manifest

### 5.1 Manifest Format

`snsr-eval-batch`'s `-c` flag takes a CSV file listing audio/reference pairs, one pair per line:

```
file1.wav,file1.nrm
file2.wav,file2.nrm
```

**The format is strict:** no whitespace anywhere on the line — not around the comma, and not embedded in a path — and no quoting. `snsr-eval-batch` does not support enclosing a filespec in quotes to work around embedded spaces, so any audio or reference path containing a space simply cannot be used in the manifest; such files need to be renamed first. UTF-8 is supported for both the file paths and the transcript contents.

The manifest also won't contain an absolute path — just bare filenames. That means `snsr-eval-batch` has to be run with the corpus folder itself as its working directory, so it can find the files named in the manifest — see [6.1](#61-command).

### 5.2 Generating the Manifest with a Script

Rather than hand-writing this CSV, generate it from the corpus directory. Because the manifest must contain bare filenames (see [5.1](#51-manifest-format)), run the script **from inside the corpus folder itself**, passing `.` as the directory to scan. The script matches each audio file to its same-named `.nrm` reference file, skips (with a warning) any file with no match or with a space/comma in its name, and writes plain `filename,filename` lines with no added whitespace or quoting. Pick whichever of the three fits your environment — they produce identical manifests. This guide uses the Python version in its worked examples.

**Python:**

```python
#!/usr/bin/env python3
"""Build a snsr-eval-batch CSV manifest from a corpus directory of
audio files (.wav/.flac) and matching .nrm reference transcripts.
Writes bare filenames only — run from inside the corpus folder."""

import sys
from pathlib import Path

AUDIO_EXTS = (".wav", ".flac")

def is_safe(name: str) -> bool:
    return " " not in name and "," not in name

def build_manifest(corpus_dir: Path, manifest_path: Path) -> None:
    lines = []
    for audio_path in sorted(corpus_dir.iterdir()):
        if audio_path.suffix.lower() not in AUDIO_EXTS:
            continue
        ref_path = audio_path.with_suffix(".nrm")
        if not ref_path.exists():
            print(f"warning: no reference for {audio_path.name}, skipping", file=sys.stderr)
            continue
        if not (is_safe(audio_path.name) and is_safe(ref_path.name)):
            print(f"warning: {audio_path.name} or its reference has a space/comma "
                  f"in its name (unsupported by snsr-eval-batch), skipping", file=sys.stderr)
            continue
        lines.append(f"{audio_path.name},{ref_path.name}")

    manifest_path.write_text("\n".join(lines) + "\n", encoding="utf-8")
    print(f"wrote {len(lines)} pairs to {manifest_path}")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        sys.exit(f"usage: {sys.argv[0]} <corpus_dir> <manifest.csv>")
    build_manifest(Path(sys.argv[1]), Path(sys.argv[2]))
```

```bash
cd data/audio/stt-16kHz-en-general-quicktest
python build_manifest.py . manifest.csv
```

**PowerShell:**

```powershell
param(
    [Parameter(Mandatory)] [string]$CorpusDir,
    [Parameter(Mandatory)] [string]$ManifestPath
)

$lines = Get-ChildItem -Path $CorpusDir -File |
    Where-Object { $_.Extension -in ".wav", ".flac" } |
    Sort-Object Name |
    ForEach-Object {
        $audioName = $_.Name
        $refName = [IO.Path]::ChangeExtension($audioName, ".nrm")
        $refPath = Join-Path $CorpusDir $refName

        if (-not (Test-Path -LiteralPath $refPath)) {
            Write-Warning "no reference for $audioName, skipping"
            return
        }
        if ($audioName -match '[ ,]' -or $refName -match '[ ,]') {
            Write-Warning "$audioName or its reference has a space/comma in its name, skipping"
            return
        }
        "$audioName,$refName"
    }

Set-Content -Path $ManifestPath -Value $lines -Encoding utf8
Write-Host "wrote $($lines.Count) pairs to $ManifestPath"
```

```powershell
cd data\audio\stt-16kHz-en-general-quicktest
.\build-manifest.ps1 -CorpusDir . -ManifestPath manifest.csv
```

**Bash:**

```bash
#!/usr/bin/env bash
# Build a snsr-eval-batch CSV manifest from a corpus directory of
# audio files (.wav/.flac) and matching .nrm reference transcripts.
# Writes bare filenames only — run from inside the corpus folder.
set -euo pipefail

corpus_dir="$1"
manifest_path="$2"

: > "$manifest_path"
count=0

for audio_path in "$corpus_dir"/*.wav "$corpus_dir"/*.flac; do
    [ -e "$audio_path" ] || continue
    audio_name="$(basename "$audio_path")"
    ref_name="${audio_name%.*}.nrm"
    ref_path="$corpus_dir/$ref_name"

    if [ ! -f "$ref_path" ]; then
        echo "warning: no reference for $audio_name, skipping" >&2
        continue
    fi
    case "$audio_name$ref_name" in
        *' '*|*','*)
            echo "warning: $audio_name or its reference has a space/comma in its name, skipping" >&2
            continue
            ;;
    esac

    echo "$audio_name,$ref_name" >> "$manifest_path"
    count=$((count + 1))
done

echo "wrote $count pairs to $manifest_path"
```

```bash
cd data/audio/stt-16kHz-en-general-quicktest
./build-manifest.sh . manifest.csv
```

On SDK 7.8.0, run whichever script you choose **after** the FLAC→WAV conversion in [4.2](#42-converting-flac-to-wav-sdk-780-only), so the manifest picks up the converted `.wav` files rather than the original `.flac` files.

---

## 6. Step 4 — Run snsr-eval-batch

### 6.1 Command

Because the manifest lists bare filenames rather than absolute paths (see [5.1](#51-manifest-format)), `snsr-eval-batch` must be run **with the corpus folder as its working directory**, so it can find the files named in `-c`. Reference the STT model with a relative path back up to the SDK's `model/` directory — three levels up from `data/audio/stt-16kHz-en-general-quicktest/`:

```bash
cd data/audio/stt-16kHz-en-general-quicktest
snsr-eval-batch -t ../../../model/stt-enUS-general-small-2.2.5-pnc.snsr \
    -c manifest.csv -w -n -j 6
```

### 6.2 Flags Reference

| Flag | Description |
|---|---|
| `-t task` | STT task file to evaluate (required) — the standalone model from [Step 1](#3-step-1--obtain-a-standalone-base-stt-model-file) |
| `-c filename` | CSV manifest of audio/reference pairs from [Step 3](#5-step-3--build-the-csv-manifest) |
| `-a` | Wraps the standalone STT model with `tpl-vad-lvcsr` so VAD segments a single long audio file into multiple utterances — for a corpus of one utterance per file (this guide's setup), leave it off. See [9](#9-common-pitfalls) |
| `-w` | Calculate word-error rate on the in-vocabulary audio in the manifest |
| `-n` | Normalize both hypothesis and reference before scoring (lowercase, strip punctuation) — see [8.1](#81-normalization-caveats) |
| `-j threads` | Number of files to process concurrently (default `1`) — see [8.2](#82-multithreading-with--j) |
| `-s setting=value` | Override an individual task setting, same as `snsr-eval` |
| `-l filename` | Log output path (default `<task>.log`) |
| `-v` | Increase verbosity (stackable) |

### 6.3 Reading the Output

A run over the corpus reports totals across all files:

```
1000 files, 1.282 hr, 9089 Words, 1186 Substitutions, 160 Insertions, 138 Deletions, 16.327% WER, 22.7 xRT
```

(This is the actual output of the [Complete Walkthrough](#7-complete-walkthrough-windows-tnl-sdk-790) in [Section 7](#7-complete-walkthrough-windows-tnl-sdk-790) — `stt-enUS-general-small-2.2.5-pnc.snsr` against the full `stt-16kHz-en-general-quicktest` corpus.)

- **Words** — total reference words scored across the corpus (the WER denominator).
- **Substitutions / Insertions / Deletions** — the three error categories from [1.1](#11-what-wer-measures), summed across all files.
- **WER** — `(Substitutions + Insertions + Deletions) / Words`, as a percentage.
- **xRT** — processing speed as a multiple of real-time (higher is faster); this reflects `snsr-eval-batch`'s batch throughput, not the target-device RTF measured in [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md) — don't conflate the two.

---

## 7. Complete Walkthrough (Windows, TNL SDK 7.9.0)

This section is a condensed, copy-pasteable run-through of [Steps 1–4](#3-step-1--obtain-a-standalone-base-stt-model-file) above, in PowerShell, for a Windows machine with the TNL 7.9.0 SDK installed — using this guide's running example throughout (`stt-enUS-general-small-2.2.5-pnc.snsr` against the `stt-16kHz-en-general-quicktest` test set). See the referenced sections for the reasoning behind each command.

Examples assume the SDK is installed at `C:\Sensory\TrulyNaturalSDK\7.9.0` — substitute your actual install path.

### 7.1 Download the STT Model

1. Open https://doc.sensory.com/tnl/7.9/models/downloads/ in a browser and download `stt-enUS-general-small-2.2.5-pnc.snsr` (or your model of choice — see [2.2](#22-stt-model)).
2. Move it into the SDK's `model\` folder:

```powershell
Move-Item "$env:USERPROFILE\Downloads\stt-enUS-general-small-2.2.5-pnc.snsr" `
    "C:\Sensory\TrulyNaturalSDK\7.9.0\model\stt-enUS-general-small-2.2.5-pnc.snsr"
```

### 7.2 Download and Extract the Test Corpus

Open the corpus library — https://drive.google.com/drive/u/1/folders/16P_SwmHJnJ4ls5J4NCW1FsWYMpwTEt0x (see [2.3](#23-test-audio-corpus)) — navigate to `en > general`, and download `stt-16kHz-en-general-quicktest.zip`. Extract it into the SDK's `data\audio\` folder. The zip already contains its own `stt-16kHz-en-general-quicktest\` folder, so extract straight into `data\audio\` — not into a subfolder of the same name (see [4.1](#41-corpus-layout-and-reference-transcripts)) — leaving you with `data\audio\stt-16kHz-en-general-quicktest\`, containing the 1,000 `.flac`/`.nrm` pairs plus an `info.yml` metadata file.

### 7.3 Build the CSV Manifest

Save the PowerShell manifest script from [5.2](#52-generating-the-manifest-with-a-script) as `build-manifest.ps1`, then run it from inside the corpus folder (`data\audio\stt-16kHz-en-general-quicktest\`), passing `.` as the directory to scan: `.\build-manifest.ps1 -CorpusDir . -ManifestPath manifest.csv`.

### 7.4 Run snsr-eval-batch and Review Results

Still in the corpus folder, invoke the SDK's `bin\snsr-eval-batch.exe`, referencing the model three levels up:

```powershell
..\..\..\bin\snsr-eval-batch.exe -t ..\..\..\model\stt-enUS-general-small-2.2.5-pnc.snsr `
    -c manifest.csv -w -n -j 6
```

See [6.3](#63-reading-the-output) for how to read the resulting Words/Substitutions/Insertions/Deletions/WER/xRT summary.

---

## 8. Step 5 — Interpreting Results

### 8.1 Normalization Caveats

The `-n` flag lowercases and strips punctuation from both the hypothesis and the reference before diffing. Without it, a correct recognition can still be scored as a substitution error purely over casing or punctuation (e.g. `"Set."` vs `"set"`). Use `-n` for a WER figure representative of semantic correctness; omit it only if you specifically need to audit the model's casing/punctuation output (relevant for `-pnc` models).

`-n` only matters for languages that have case and/or punctuation in the first place — it has nothing to normalize (and no effect) for languages without those concepts.

Whichever you choose, apply it consistently across runs you intend to compare — a WER measured with `-n` is not directly comparable to one measured without it.

### 8.2 Multithreading with -j

`-j` parallelizes across corpus files, not within a single file, so it speeds up the batch run without changing the recognition result for any individual file. Set it to roughly the number of available CPU cores on your workstation. This only affects wall-clock time for the batch run — it has no effect on the reported WER.

---

## 9. Common Pitfalls

- Running SDK 7.8.0's `snsr-eval-batch` against FLAC audio without converting to WAV first — see [4.2](#42-converting-flac-to-wav-sdk-780-only).
- Passing an assembled wake-word/VAD/STT pipeline file as `-t` instead of a standalone STT task — extract the STT slot first ([3.2](#32-extracting-the-stt-model-from-an-assembled-pipeline)), or use a directly-downloaded STT model.
- Using `-a` with a corpus of one utterance per audio file, as in this guide — `-a` is only appropriate for a single, very long audio file containing repeated utterances that VAD needs to segment. Using it against a one-utterance-per-file corpus doesn't match the corpus layout and will skew results.
- Using `-a` with a character-based language (e.g. Japanese, Chinese) — don't combine it with CER scoring.
- Comparing WER figures where one run used `-n` and the other didn't ([8.1](#81-normalization-caveats)).
- Treating the reported `xRT` as a target-device RTF figure — it reflects workstation batch throughput, not the embedded-platform real-time factor from [Benchmarking RTF and Avg/Max Memory Usage](benchmarking-rtf-memory.md).
- Drawing conclusions from a small or unrepresentative corpus — WER on a handful of clean utterances won't predict field performance; use a corpus that reflects the target deployment's vocabulary, accents, and noise conditions (see [1.2](#12-why-measure-it-per-deployment)).

---

## 10. Quick Reference — Command Summary

```bash
# 0. Save the downloaded STT model under the SDK's model/ directory
#    -> model/stt-enUS-general-small-2.2.5-pnc.snsr

# Optional: extract STT model from an assembled pipeline instead (e.g. for SDK 7.8.0)
snsr-edit -v -t opt-vg-vad-stt-enUS-general-medium-2.4.3-pnc_66.snsr \
    -e 2.0 stt-enUS-general-medium-2.4.3-pnc_66.snsr

# 1. Download and unzip the test corpus (en > general > stt-16kHz-en-general-quicktest.zip)
#    The zip already contains its own stt-16kHz-en-general-quicktest/ folder
unzip stt-16kHz-en-general-quicktest.zip -d data/audio/

# Required for SDK 7.8.0 only: convert corpus FLAC to WAV
for f in data/audio/stt-16kHz-en-general-quicktest/*.flac; do
    sox "$f" -r 16000 -c 1 -b 16 "${f%.flac}.wav"
done

# 2. Build the CSV manifest — run from inside the corpus folder, bare filenames only
cd data/audio/stt-16kHz-en-general-quicktest
python build_manifest.py . manifest.csv

# 3. Run the WER evaluation from that same folder (omit -a: one utterance per file)
snsr-eval-batch -t ../../../model/stt-enUS-general-small-2.2.5-pnc.snsr \
    -c manifest.csv -w -n -j 6
```

---

## 11. References

- TNL SDK 7.9 Docs: https://doc.sensory.com/tnl/7.9/
- `snsr-eval-batch` reference: https://doc.sensory.com/tnl/7.9/tools/snsr-eval-batch/
- `snsr-edit` reference: https://doc.sensory.com/tnl/7.9/tools/snsr-edit/
- STT model downloads (7.9.0+): https://doc.sensory.com/tnl/7.9/models/downloads/
- STT model downloads (7.8.0): https://doc.sensory.com/tnl/7.8/models/downloads/
- `tpl-opt-spot-vad-lvcsr` template: https://doc.sensory.com/tnl/7.9/models/tpl/tpl-opt-spot-vad-lvcsr/
- Team WER test corpus (Google Drive): https://drive.google.com/drive/u/1/folders/16P_SwmHJnJ4ls5J4NCW1FsWYMpwTEt0x
- ISO 639 language codes: https://en.wikipedia.org/wiki/List_of_ISO_639_language_codes
