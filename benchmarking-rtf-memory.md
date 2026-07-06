# Benchmarking Real-Time Factor (RTF) and Avg/Max Memory Usage for Sensory TrulyHandsfree/TrulyNatural Models

**Product:** TrulyHandsfree (THF) / TrulyNatural (TNL) SDK  
**Models:** Voice Genie wake word + Automotive STT pipeline  
**Version:** 7.6.1 / 7.7.0 / 7.8.0+  
**Document Version:** 1.0.0  
**Audience:** Field Application Engineers, Customer Integration Engineers  
**Status:** Released under NDA — Do not distribute

*Copyright © 2026 Sensory Inc. All rights reserved. This document is confidential and proprietary to Sensory Inc. It may not be reproduced, distributed, or disclosed to any third party without prior written permission from Sensory Inc.*

---

## Table of Contents

1. [Overview](#1-overview)
   - [Real-Time Factor (RTF)](#real-time-factor-rtf)
   - [Real-Time Requirements by Technology Type](#real-time-requirements-by-technology-type)
   - [Memory Usage](#memory-usage)
2. [Prerequisites](#2-prerequisites)
   - [2.1 Test Platform](#21-test-platform)
   - [2.2 SDK Installation](#22-sdk-installation)
   - [2.3 Model Files](#23-model-files)
   - [2.4 Test Audio Files](#24-test-audio-files)
3. [Step 1 — Assemble the Pipeline Model](#3-step-1--assemble-the-pipeline-model)
4. [Step 2 — Run snsr-eval with Pipeline Profiling](#4-step-2--run-snsr-eval-with-pipeline-profiling)
   - [4.1 Profiling Flags](#41-profiling-flags)
   - [4.2 Basic Profiling Command](#42-basic-profiling-command)
   - [4.3 Reading Audio from stdin](#43-reading-audio-from-stdin)
   - [4.4 Basic Profiling Output Format (-p)](#44-basic-profiling-output-format--p)
   - [4.5 The Effect of Partial Results on RTF](#45-the-effect-of-partial-results-on-rtf)
   - [4.6 Detailed Per-Element Profiling with -pp (Advanced)](#46-detailed-per-element-profiling-with--pp-advanced)
5. [Step 3 — Measuring Worst-Case Memory Usage with Valgrind](#5-step-3--measuring-worst-case-memory-usage-with-valgrind)
   - [5.1 Running the Massif Profiler](#51-running-the-massif-profiler)
   - [5.2 Reading the Results](#52-reading-the-results)
   - [5.3 Benchmark Results (Raspberry Pi 4, SDK 7.8.0)](#53-benchmark-results-raspberry-pi-4-sdk-780)
   - [5.4 Reducing Heap Usage by Embedding the Model in Code Space](#54-reducing-heap-usage-by-embedding-the-model-in-code-space)
6. [Step 4 — Average MIPS, RTF, and Memory for Continuous Wake Word Listening](#6-step-4--average-mips-rtf-and-memory-for-continuous-wake-word-listening)
   - [6.1 The Noise Test File](#61-the-noise-test-file)
   - [6.2 Average RTF Measurement](#62-average-rtf-measurement)
   - [6.3 Average MIPS Measurement with perf stat](#63-average-mips-measurement-with-perf-stat)
   - [6.4 Average Memory Measurement](#64-average-memory-measurement)
   - [6.5 Benchmark Results (Raspberry Pi 4 8 GB, SDK 7.8.0)](#65-benchmark-results-raspberry-pi-4-8-gb-sdk-780)
7. [Platform Notes: ARM NEON Extensions](#7-platform-notes-arm-neon-extensions)
8. [Pipeline Architecture Reference](#8-pipeline-architecture-reference)
9. [Quick Reference — Command Summary](#9-quick-reference--command-summary)
10. [References](#10-references)

---

## 1. Overview

This guide explains how to benchmark **real-time factor (RTF)**, **MIPS**, and **memory usage** for Sensory TrulyHandsfree and TrulyNatural speech recognition models using the `snsr-eval` command-line tool and standard Linux utilities. The techniques here allow developers and integration engineers to characterize CPU and memory requirements for a given model pipeline on a target platform, and to reproduce the benchmark figures published in Sensory's SDK documentation.

### Real-Time Factor (RTF)

RTF is the standard metric for measuring how much CPU time the SDK uses relative to the duration of the audio being processed:

```
RTF = (CPU processing time) / (audio duration)
```

An RTF of **1.0** means the engine consumes exactly as much CPU time as the audio lasts — i.e., just barely real-time. An RTF of **0.3** means the engine uses 30% of real time, leaving 70% CPU headroom.

Pipeline profiling via `snsr-eval` is supported across all TrulyHandsfree (THF) and TrulyNatural (TNL) technology types — wake words, phrase-spotted commands, LVCSR, and STT. This guide focuses on **wake words and STT** as they represent the two ends of the pipeline and the primary performance concerns for an automotive voice assistant integration.

### Real-Time Requirements by Technology Type

Not all pipeline stages have the same real-time constraint:

- **Wake words (continuous listening)** — the wake word detector runs continuously and must keep pace with the incoming audio stream at all times. An RTF > 1.0 means the detector falls behind and will miss triggers. **RTF ≤ 1.0 is a hard requirement** for wake word models.

- **STT, LVCSR, and phrase-spotted commands** — these technologies operate over a **finite, bounded listening window** that begins after the wake word fires. Because the audio duration is known in advance and the input can be buffered, processing does not need to complete in real time. An RTF > 1.0 is acceptable — the engine will simply take longer than the utterance duration to return a result, adding latency but not causing missed recognitions. This allows Sensory to be deployed on less CPU-capable platforms than a strict real-time constraint would require.

In practice, minimizing RTF for the STT stage is still desirable to reduce end-to-end response latency for the user.

### Memory Usage

This guide covers two distinct memory measurements:

- **Peak (worst-case) heap memory** — measured using Valgrind `massif` while processing a voice command utterance. This is the maximum heap allocation the pipeline will ever reach and is used for platform memory budgeting.

- **Average steady-state memory** — measured using `top` while the wake word detector runs continuously on live audio with no speech present. This is the baseline resident memory during idle listening and is the more relevant figure for long-running deployments.

---

## 2. Prerequisites

### 2.1 Test Platform

Benchmarks in this guide were performed on a **Raspberry Pi 4** running Raspberry Pi OS (64-bit). The Pi 4 was chosen as the reference platform because it is widely available, inexpensive, and highly standardized — benchmarks run on one Pi 4 should be repeatable on any other. It also includes ARM NEON SIMD support (see Section 7), making it representative of the class of embedded ARM platforms targeted by Sensory's automotive and IoT customers.

A **Raspberry Pi 4 or later** is recommended for reproducing the results in this guide.

### 2.2 SDK Installation

Ensure the TNL SDK 7.7.0+ is installed and the following are accessible on your PATH:

- `snsr-eval` — model evaluation / profiling tool
- `snsr-edit` — model pipeline assembly tool

### 2.3 Model Files

The following model files from the SDK distribution are used in this guide:

| File | Role |
|---|---|
| `tpl-spot-vad-lvcsr-3.23.0.snsr` | Pipeline template: wake word → VAD → STT |
| `spot-voicegenie-enUS-6.5.1-m.snsr` | Voice Genie wake word (slot 0) |
| `stt-enUS-automotive-medium-2.3.15-pnc.snsr` | Automotive STT model (slot 1) |

By convention, model files are found under `model/` in the SDK installation directory.

### 2.4 Test Audio Files

For reproducible RTF measurements, use **pre-recorded WAV files** rather than live audio. This eliminates audio capture jitter and allows the same utterances to be re-run identically across platforms.

The SDK includes a reference audio file specifically for this purpose:

| File | Location | Content |
|---|---|---|
| `voice-genie-set-cruise-control.wav` | `data/audio/` | "Voice Genie, set the cruise control to 55 miles per hour." |

This file covers the full pipeline — it triggers the Voice Genie wake word in slot 0, activates VAD segmentation, and drives the Automotive STT model in slot 1 through a realistic command utterance.

Audio file format (for reference, if you supply additional test files):
- Format: 16-bit PCM WAV, little-endian
- Sample rate: 16000 Hz (16 kHz), mono
- Content: utterances of the form **"Voice Genie, \<command\>"**

---

## 3. Step 1 — Assemble the Pipeline Model

Before profiling, combine the three model files into a single pipeline `.snsr` file using `snsr-edit`. This only needs to be done once.

```bash
cd ~/Sensory/TrulyNaturalSDK/7.8.0

bin/snsr-edit -o vg-stt.snsr \
    -t model/tpl-spot-vad-lvcsr-3.23.0.snsr \
    -f 0 model/spot-voicegenie-enUS-6.5.1-m.snsr \
    -f 1 model/stt-enUS-automotive-medium-2.3.15-pnc.snsr \
    -s include-wake-word-audio=1
```

**What this does:**

- `-t` — specifies the **template** (`tpl-spot-vad-lvcsr`), which defines the pipeline topology: wake word (slot 0) → VAD endpoint detection → STT recognizer (slot 1)
- `-f 0` — loads the Voice Genie wake word model into slot 0
- `-f 1` — loads the Automotive STT model into slot 1
- `-s include-wake-word-audio=1` — passes the wake word audio through to the STT decoder so the full utterance (including "Voice Genie") can be included in the transcript if the model supports it

**Output:** `vg-stt.snsr` — the assembled pipeline model, ready for evaluation.

---

## 4. Step 2 — Run `snsr-eval` with Pipeline Profiling

### 4.1 Profiling Flags

`snsr-eval` supports two levels of pipeline profiling:

| Flag | Description |
|---|---|
| `-p` | Enable pipeline profiling — reports aggregate timing per pipeline stage |
| `-pp` | Enable verbose pipeline profiling — reports per-utterance timing in addition to aggregate |

> **Note:** Pipeline profiling is marked **experimental** in the SDK documentation. Results are reliable for comparative benchmarking across platforms but should not be treated as production-level instrumentation.

### 4.2 Basic Profiling Command

Run the assembled pipeline against the SDK reference audio file:

```bash
bin/snsr-eval -p -t vg-stt.snsr data/audio/voice-genie-set-cruise-control.wav
```

For more detail on per-utterance timing:

```bash
bin/snsr-eval -pp -t vg-stt.snsr data/audio/voice-genie-set-cruise-control.wav
```

With increased verbosity for full recognition output alongside profiling:

```bash
bin/snsr-eval -v -pp -t vg-stt.snsr data/audio/voice-genie-set-cruise-control.wav
```

### 4.3 Reading Audio from stdin

If you want to pipe audio from another source (e.g., a file conversion utility):

```bash
sox input.wav -t raw -r 16000 -e signed -b 16 - | bin/snsr-eval -p -t vg-stt.snsr -
```

The `-` argument tells `snsr-eval` to read headerless 16-bit PCM from stdin.

### 4.4 Basic Profiling Output Format (`-p`)

With `-p`, `snsr-eval` appends an RTF (real-time factor) summary after the recognition output. RTF is reported as a **percentage** for the overall pipeline and per slot. The following is actual output from `voice-genie-set-cruise-control.wav` run on a Raspberry Pi 4:

```
P   3010   3530 Said the cruis
P   3050   3930 Set. The cruise control
P   3090   4170 Set. The cruise control to
P   3090   4730 Set the cruise control to fifty
P   3090   5170 Set the cruise control to fifty five minut
P   3090   5610 Set the cruise control to fifty five miles back
NLU intent: set_cruise_control (0.9968) = set the cruise control to 55 miles per hour
NLU entity:   number (0.9937) = 55
NLU entity:   speed_unit (0.9936) = miles per hour
  3090   5770 Set the cruise control to fifty five miles per hour.
Total:   108345 samples,  4.959 seconds, 73.23% rtf
   0.:    49440 samples,  0.133 seconds,  4.32% rtf
   1.:    73680 samples,  4.794 seconds, 104.09% rtf
```

**Interpreting the profiling summary:**

| Line | Description |
|---|---|
| `Total:` | Overall pipeline: total audio samples processed, total CPU time, and aggregate RTF |
| `0.:` | Slot 0 (wake word): samples and CPU time consumed by the phrase spotter |
| `1.:` | Slot 1 (STT): samples and CPU time consumed by the STT decoder |

- **RTF is expressed as a percentage.** 73.23% rtf means the pipeline used 73% of the audio duration in CPU time on this platform.
- **Slot 0 (wake word) RTF is very low** (4.32%) — as expected for a fixed-phrase spotter running continuously.
- **Slot 1 (STT) RTF at 104.09%** — this exceeds 100%, which means the STT decoder took slightly longer than the utterance duration to process. As described in Section 1, this is acceptable for STT because the audio is buffered over a bounded listening window; it adds latency but does not cause missed recognitions.
- The `P` lines are **partial result hypotheses**, emitted during decoding as the model's best guess evolves. These are distinct from the final transcript on the last line.

### 4.5 The Effect of Partial Results on RTF

Partial results have a **significant impact on CPU cost.** By default, `snsr-eval` emits a partial result every 1000 ms of audio processed. Disabling partial results dramatically reduces RTF:

```
bin/snsr-eval -p -t vg-stt.snsr -s partial-result-interval=0 \
    data/audio/voice-genie-set-cruise-control.wav
```

```
NLU intent: set_cruise_control (0.9968) = set the cruise control to 55 miles per hour
NLU entity:   number (0.9937) = 55
NLU entity:   speed_unit (0.9936) = miles per hour
  3090   5770 Set the cruise control to fifty five miles per hour.
Total:   108345 samples,  1.856 seconds, 27.41% rtf
   0.:    49440 samples,  0.131 seconds,  4.25% rtf
   1.:    73680 samples,  1.697 seconds, 36.85% rtf
```

**Comparison (Raspberry Pi 4):**

| Configuration | Total RTF | Slot 1 (STT) RTF |
|---|---|---|
| `partial-result-interval=1000` (default) | 73.23% | 104.09% |
| `partial-result-interval=0` (disabled) | 27.41% | 36.85% |

Disabling partial results reduced total CPU load by nearly **3×** on this platform, and brought the STT slot well under 100% RTF.

**Design guidance:**

- Partial results are computationally expensive. If your application does not need streaming hypothesis updates (e.g., displaying live transcription to the user), set `partial-result-interval=0`.
- If your application needs partial results but is experiencing high CPU load, latency, or audio buffer overruns, increase the interval (e.g., `-s partial-result-interval=2000`) to reduce the frequency of partial decoding passes. Sensory recommends using the **largest interval your UX can tolerate** under resource-constrained conditions.
- When benchmarking for platform feasibility, measure both configurations and report them separately, as they represent meaningfully different use cases.

### 4.6 Detailed Per-Element Profiling with `-pp` (Advanced)

Adding a second `-p` (i.e., `-pp`) produces a full hierarchical breakdown of CPU time for every element in the pipeline. This is useful for understanding exactly where time is spent within each stage — for example, identifying whether the bottleneck in the STT slot is the neural network, the language model, or the decoder search.

```bash
bin/snsr-eval -pp -t vg-stt.snsr -s partial-result-interval=0 \
    data/audio/voice-genie-set-cruise-control.wav
```

**Column definitions:**

| Column | Description |
|---|---|
| `ELEMENT` | Pipeline component name. Prefixed by slot number (`0.`, `1.`, `vad.`) or `:total:` for the overall pipeline |
| `MAX` | Maximum time observed for a single invocation of this element (useful for latency analysis) |
| `TIME` | Cumulative CPU time spent in this element across the entire audio file |
| `BIN` | Percentage of this element's *parent container's* total time |
| `TOTAL` | Percentage of the overall pipeline total time |
| `CPU` | Percentage of the audio file duration — equivalent to the RTF contribution of this element |

**Actual output (Raspberry Pi 4, SDK 7.8.0, partial results disabled):**

```
Processed 108345 samples at 16000 Hz for a total 6771.562 ms

              ELEMENT        MAX            TIME       BIN    TOTAL      CPU
               :total:                  1849.695 ms  100.0 %  100.0 %   27.3 %
                    1:   481.435 ms     1690.494 ms   91.4 %   91.4 %   25.0 %
                    0:     1.305 ms      129.896 ms    7.0 %    7.0 %    1.9 %
                  vad:     0.478 ms       26.290 ms    1.4 %    1.4 %    0.4 %
            :overhead:                     1.957 ms    0.1 %    0.1 %    0.0 %
                  arb:     0.032 ms        0.608 ms    0.0 %    0.0 %    0.0 %
                  seg:     0.108 ms        0.293 ms    0.0 %    0.0 %    0.0 %
                  dmx:     0.002 ms        0.144 ms    0.0 %    0.0 %    0.0 %
              reverse:     0.013 ms        0.013 ms    0.0 %    0.0 %    0.0 %

             :1 total:                  1690.494 ms  100.0 %   91.4 %   25.0 %
               1.onnx:   449.495 ms     1517.258 ms   89.8 %   82.0 %   22.4 %
               1.bert:    43.552 ms      100.641 ms    6.0 %    5.4 %    1.5 %
                1.mod:     0.405 ms       36.844 ms    2.2 %    2.0 %    0.5 %
          :1 overhead:                    11.857 ms    0.7 %    0.6 %    0.2 %
                1.fex:     0.365 ms        8.855 ms    0.5 %    0.5 %    0.1 %
             1.search:     2.713 ms        8.564 ms    0.5 %    0.5 %    0.1 %
                1.pnc:     5.402 ms        5.402 ms    0.3 %    0.3 %    0.1 %
             1.result:     0.626 ms        0.626 ms    0.0 %    0.0 %    0.0 %
           1.textnorm:     0.185 ms        0.185 ms    0.0 %    0.0 %    0.0 %
            ...

             :0 total:                   129.896 ms  100.0 %    7.0 %    1.9 %
                0.net:     0.732 ms      116.526 ms   89.7 %    6.3 %    1.7 %
                0.fex:     0.080 ms        6.694 ms    5.2 %    0.4 %    0.1 %
             0.search:     0.602 ms        2.759 ms    2.1 %    0.1 %    0.0 %
            ...

           :vad total:                    26.290 ms  100.0 %    1.4 %    0.4 %
              vad.fex:     0.074 ms        8.590 ms   32.7 %    0.5 %    0.1 %
             vad.net0:     0.069 ms        7.818 ms   29.7 %    0.4 %    0.1 %
              vad.vad:     0.389 ms        4.297 ms   16.3 %    0.2 %    0.1 %
            ...
```

**Key observations from the detailed profile:**

- **STT (slot 1) dominates at 91.4% of total CPU time.** Within slot 1, the ONNX transformer neural network (`1.onnx`) accounts for 89.8% of that slot's time — it is by far the primary cost driver. The BERT language model (`1.bert`) is second at 6.0%. Everything else (feature extraction, CTC search, NLU, punctuation normalization) is negligible by comparison.

- **Wake word (slot 0) is lightweight at 7.0% of total CPU**, with its own neural network (`0.net`) accounting for 89.7% of slot 0's time. At only 1.9% of audio duration, the wake word detector imposes minimal continuous load on the system.

- **VAD contributes just 1.4% of total CPU time (0.4% of audio duration).** The VAD is essentially free from a resource planning perspective. Its feature extraction (`vad.fex`) and two-stage neural network (`vad.net0`, `vad.net1`) together account for the majority of its modest cost.

- **The `MAX` column reveals worst-case latency per invocation.** The ONNX network in slot 1 has a maximum single-call time of 449 ms — relevant if the application is sensitive to jitter in partial result delivery.

- **Punctuation and normalization (`1.pnc`) runs only once** at the end of the utterance (5.4 ms total, same as MAX), confirming it is a post-processing step on the final hypothesis only.

---

## 5. Step 3 — Measuring Worst-Case Memory Usage with Valgrind

Valgrind's `massif` heap profiler tracks every allocation and deallocation over the course of a run, giving a reliable worst-case heap figure for the pipeline. This is more accurate than a simple RSS snapshot, and is the right tool for establishing a memory budget for embedded or resource-constrained deployments.

> **Important:** Do **not** use `-p` or `-pp` when running under massif. The profiling instrumentation adds its own allocations and will inflate the memory figures.

### 5.1 Running the Massif Profiler

Run the pipeline twice — once with partial results disabled, once with the default interval — to confirm that `partial-result-interval` does not materially affect memory requirements:

```bash
cd ~/Sensory/trulynatural/7.8.0

# Run 1: partial results disabled
valgrind --tool=massif --time-unit=B --massif-out-file=vg-stt.massif \
    bin/snsr-eval -t vg-stt.snsr -s partial-result-interval=0 \
    data/audio/voice-genie-set-cruise-control.wav
ms_print vg-stt.massif > vg-stt-pri0.txt

# Run 2: default partial-result-interval (1000 ms)
valgrind --tool=massif --time-unit=B --massif-out-file=vg-stt.massif \
    bin/snsr-eval -t vg-stt.snsr \
    data/audio/voice-genie-set-cruise-control.wav
ms_print vg-stt.massif > vg-stt-pri1000.txt
```

**Options used:**

| Option | Description |
|---|---|
| `--tool=massif` | Selects the heap profiler |
| `--time-unit=B` | Measures time in bytes allocated rather than wall time, giving the clearest picture of allocation behavior |
| `--massif-out-file=vg-stt.massif` | Writes profiling data to this named file instead of the default `massif.out.<pid>` |

Valgrind runs significantly slower than native execution (typically 10–50×), so expect the `snsr-eval` run to take considerably longer than usual. This does not affect the memory measurements.

### 5.2 Reading the Results

The `ms_print` output begins with an ASCII chart of heap usage over time (x-axis: bytes allocated; y-axis: MB in use), followed by a snapshot table. The key figure is the **peak total heap** at the top of the chart, and the corresponding row in the snapshot table marked `(peak)`.

### 5.3 Benchmark Results (Raspberry Pi 4, SDK 7.8.0)

**Partial results disabled (`partial-result-interval=0`):**

```
    MB
174.5^                                                  #                     
     |                                                  #@                    
     |                                                ::#@@@                  
     |                                              ::: #@@ :                 
     |                                        @ :  :: : #@@ :::               
     |                              :::::    @@:::::: : #@@ ::                
     |                           :@@:::: ::::@@:::::: : #@@ :: :              
     |                          ::@ :::: : : @@:::::: : #@@ :: :              
     |              ::::::::::::::@ :::: : : @@:::::: : #@@ :: ::@@@          
     |           ::::::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@            
     |          ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@            
     |          ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :         
     |        ::::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@        
     |        : ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@:       
     |        : ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@::::::: 
     |     :::: ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@::      
     |     :  : ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@::      
     |     :  : ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@::      
     |     :  : ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@::      
     |     :  : ::: :::: ::: :: ::@ :::: : : @@:::::: : #@@ :: ::@  :@::      
   0 +----------------------------------------------------------------------->MB
     0                                                                   614.8
Number of snapshots: 53
 Detailed snapshots: [19, 27, 28, 36 (peak), 37, 38, 43, 45, 49]
```

**Default partial results (`partial-result-interval=1000`):**

```
    MB
167.2^                                                   @                    
     |                                                  #@@@                  
     |                                                  #@@ @                 
     |                                             @@:@@#@@ @:                
     |                                       @@::::@ :@ #@@ @::               
     |                             ::::::::::@ :: :@ :@ #@@ @::               
     |                           ::::: :::: :@ :: :@ :@ #@@ @::::             
     |               ::::::::::::: ::: :::: :@ :: :@ :@ #@@ @:::              
     |             :::::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :::          
     |           ::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: ::           
     |          :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: ::           
     |          :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: ::        
     |        :::::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @       
     |        : :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @       
     |        : :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @:::::: 
     |     :::: :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @:      
     |     :  : :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @:      
     |     :  : :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @:      
     |     :  : :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @:      
     |     :  : :::: :::::: : :: : ::: :::: :@ :: :@ :@ #@@ @::: :: : @:      
   0 +----------------------------------------------------------------------->MB
     0                                                                   624.0
Number of snapshots: 96
 Detailed snapshots: [27, 32, 34, 36 (peak), 37, 38, 39, 46, 49, 59, 69, 79, 89]
```

**Comparison:**

| Configuration | Peak Snapshot | Peak Total Heap | Peak Useful Heap |
|---|---|---|---|
| `partial-result-interval=0` | 36 | 183,016,824 B (**174.5 MB**) | 176,852,617 B (168.7 MB) |
| `partial-result-interval=1000` (default) | 36 | 174,809,632 B (**166.7 MB**) | 168,644,469 B (160.8 MB) |

**Key observations:**

- Peak heap memory is **~175 MB** in the worst case on this platform, regardless of partial result configuration.
- The difference between the two configurations is only **~7.8 MB (~4%)**, confirming that `partial-result-interval` does not materially affect memory usage. The default (1000 ms) configuration actually showed slightly *lower* peak usage in this particular test, though this should not be assumed to hold in general.
- Both runs peaked at snapshot 36, suggesting the peak occurs at the same point in execution — during STT model inference — regardless of partial result settings.

### 5.4 Reducing Heap Usage by Embedding the Model in Code Space

The heap figures above include the `vg-stt.snsr` model file loaded into heap during initialization. The assembled model is **95,101,108 bytes (~90.7 MB)**, which accounts for the majority of the peak heap usage seen in the massif output.

On embedded platforms where heap is at a premium, this can be avoided by converting the model to a C array using `snsr-edit -c` and linking it directly into the executable's code (read-only data) segment. When the model lives in code space rather than heap, it is excluded from the heap figures entirely, reducing the effective runtime heap requirement by the model size.

```bash
# Convert the assembled pipeline model to a C array
bin/snsr-edit -c vg-stt-model.c -t vg-stt.snsr
```

The resulting `.c` file defines a byte array that can be compiled and linked into your application. The model is then passed to the SDK via the data API rather than loaded from a file path at runtime.

Developers are encouraged to study the **`spot-data.c`** sample (included in the SDK under `sample/c/`) for guidance on using embedded model data in application code.

---

## 6. Step 4 — Average MIPS, RTF, and Memory for Continuous Wake Word Listening

The peak RTF and memory figures in Sections 4 and 5 are measured during active speech recognition — the most computationally demanding moment in the pipeline. However, for continuous-listening wake word deployments, the more relevant question is: **what is the average CPU and memory load during idle listening?**

Wake word detectors may run for hours between triggers. The peak figures captured during a voice command utterance are not representative of this steady-state cost. A more useful metric for platform power and thermal budgeting is the load during normal background-noise conditions with no speech present.

> **Note:** MIPS and steady-state RTF measurements only apply to **continuous-listening wake words**. STT, LVCSR, and phrase-spotted command models are windowed technologies — they only run when triggered and consume CPU resources as needed up to the limits of the system. There is no meaningful steady-state CPU floor to measure for those technologies.

### 6.1 The Noise Test File

The correct input for these measurements is a short audio file containing **low-level white noise at −80 dBfs** — quiet enough to not trigger the wake word, but present enough to keep the detector actively processing.

The SDK does not include such a file, but one can be generated with SoX:

```bash
# 10-second file — used for RTF measurement
sox -n -r 16000 -b 16 -c 1 data/audio/noise-10s.wav synth 10 whitenoise vol -80dB

# 100-second file — required for MIPS measurement (see Section 6.3)
sox -n -r 16000 -b 16 -c 1 data/audio/noise-100s.wav synth 100 whitenoise vol -80dB
```

### 6.2 Average RTF Measurement

```bash
bin/snsr-eval -p -t vg-stt.snsr data/audio/noise-10s.wav
```

**Actual output (Raspberry Pi 4, SDK 7.8.0):**

```
Total:   163303 samples,  0.440 seconds,  4.31% rtf
   0.:   163303 samples,  0.434 seconds,  4.26% rtf
```

Note that **slot 1 does not appear** in the output. Because the wake word never fires on noise input, the VAD and STT stages are never invoked and contribute no CPU time. The 4.31% total RTF reflects the continuous cost of wake word detection alone — this is the steady-state load the system will carry during idle listening.

### 6.3 Average MIPS Measurement with `perf stat`

`perf stat` is a Linux performance counter tool that measures the total number of CPU instructions executed over the lifetime of a process. For a continuous-listening wake word running against a known-duration audio file, this gives a direct MIPS figure.

> **Important:** Run `perf stat` against the **wake word model file directly** (`spot-voicegenie-enUS-6.5.1-m.snsr`), not the assembled pipeline (`vg-stt.snsr`). The pipeline includes the STT model loaded into memory even when idle, which adds initialization overhead not relevant to the steady-state wake word measurement.

Use the **100-second noise file** rather than the 10-second file. `perf stat` counts all instructions including model loading and initialization, which are a one-time fixed cost. A longer audio file dilutes this initialization overhead and gives a more accurate picture of the steady-state instruction rate.

```bash
perf stat bin/snsr-eval -t model/spot-voicegenie-enUS-6.5.1-m.snsr \
    data/audio/noise-100s.wav
```

**Actual output (Raspberry Pi 4, SDK 7.8.0):**

```
 Performance counter stats for 'bin/snsr-eval -t model/spot-voicegenie-enUS-6.5.1-m.snsr data/audio/noise-100s.wav':

          4,685.93 msec task-clock:u             #    0.994 CPUs utilized
                 0      context-switches:u        #    0.000 /sec
                 0      cpu-migrations:u          #    0.000 /sec
               599      page-faults:u             #  127.830 /sec
     8,189,811,590      cycles:u                  #    1.748 GHz
    10,546,585,073      instructions:u            #    1.29  insn per cycle
   <not supported>      branches:u
         9,089,640      branch-misses:u

       4.715627546 seconds time elapsed
       4.651830000 seconds user
       0.035906000 seconds sys
```

**Calculating MIPS:**

The key figure is `instructions:u` — the total instruction count for the entire run. Divide by the audio file duration in seconds, then by one million:

```
MIPS = instructions / audio_duration_seconds / 1,000,000
     = 10,546,585,073 / 100 / 1,000,000
     = 105.4 MIPS
```

**Interpreting the output:**

| Field | Value | Notes |
|---|---|---|
| `task-clock:u` | 4,685.93 msec | Total CPU time consumed by the process |
| `CPUs utilized` | 0.994 | Effectively single-threaded — expected |
| `cycles:u` | 8,189,811,590 | Raw clock cycles |
| `instructions:u` | 10,546,585,073 | **The key figure for MIPS calculation** |
| `insn per cycle` | 1.29 | IPC — reflects NEON SIMD utilization |
| `seconds time elapsed` | 4.716 | Wall clock time |
| `seconds user` | 4.652 | User-space CPU time |

- The **~4.7 seconds of CPU time** to process **100 seconds of audio** is consistent with the ~4.3% RTF measured in Section 6.2.
- The **IPC of 1.29** reflects efficient SIMD utilization via ARM NEON extensions. A lower IPC (closer to 1.0) on a platform without NEON would indicate the same work is taking more cycles.
- `branches:u` showing `<not supported>` is normal on some ARM configurations — it does not affect the instruction count or MIPS calculation.

### 6.4 Average Memory Measurement

Processing a WAV file offline is unsuitable for this measurement: `snsr-eval` runs WAV files as fast as the CPU allows, so even a long file is consumed in a few seconds of wall time. Instead, use **live audio mode** — run `snsr-eval` with no audio file argument in one terminal window so it reads from the default capture device and runs indefinitely, then measure memory in a second terminal.

**Step 1 — Start `snsr-eval` in live audio mode** (Terminal 1):

```bash
bin/snsr-eval -t vg-stt.snsr
```

Wait a few seconds for initialization to complete. You will see the message `Using live audio from default capture device. ^C to stop.`

**Step 2 — Get total system RAM** (Terminal 2):

```bash
grep MemTotal /proc/meminfo
```

Example output:
```
MemTotal:        7998716 kB
```

**Step 3 — Get `snsr-eval` memory as a percentage of total RAM** (Terminal 2):

```bash
top -p $(pgrep snsr-eval)
```

Note the value in the `%MEM` column for the `snsr-eval` process. Allow a few update cycles for the value to stabilize after initialization.

**Step 4 — Calculate absolute memory usage:**

```
Average RAM = %MEM × MemTotal
```

**Step 5 — Stop `snsr-eval`** (Terminal 1):

Press `^C`.

### 6.5 Benchmark Results (Raspberry Pi 4 8 GB, SDK 7.8.0)

| Metric | Value |
|---|---|
| Average RTF (wake word, noise input) | **4.31%** |
| Average MIPS (wake word, 100s noise) | **105.4 MIPS** |
| `MemTotal` | 7,998,716 kB |
| `snsr-eval` `%MEM` | 2.0% |
| **Average steady-state RAM** | 2.0% × 7,998,716 kB = **~156 MB** |

The STT model is loaded into memory but its inference-time buffers are not allocated until a wake word trigger actually fires, so the steady-state RAM figure is lower than the peak heap values measured in Section 5.

---

## 7. Platform Notes: ARM NEON Extensions

Sensory's wake word and STT models are optimized to use **ARM NEON SIMD extensions** when present on the host CPU. NEON acceleration is detected and enabled automatically at runtime — no configuration is required.

**NEON can reduce RTF by as much as 75%** compared to a non-NEON ARM build. This has a significant impact on platform feasibility assessments: a system that appears marginal without NEON may be well within headroom with it.

The **Raspberry Pi 4** (Cortex-A72) includes NEON support, and the benchmark results in this guide reflect NEON-accelerated execution. When reproducing these measurements on another ARM platform, verify that the target CPU includes NEON (all ARMv7 and AArch64 cores do) and that the SDK binary was built with NEON enabled.

> **Note:** The benchmarks in this guide should be reproduced on a Raspberry Pi 4 or comparable ARMv8 platform using the ARM Linux SDK binary distributed by Sensory. Results on other architectures — including x86-64 — will differ and should be evaluated separately using the appropriate SDK build for that platform.

---

## 8. Pipeline Architecture Reference

The `tpl-spot-vad-lvcsr` template implements the following sequential pipeline:

```
Audio Input
    │
    ▼
┌─────────────────────────────────┐
│  Slot 0: Wake Word (phrasespot) │  ← spot-voicegenie-enUS-6.5.1-m.snsr
│  "Voice Genie"                  │
└─────────────────────────────────┘
    │ on detection
    ▼
┌─────────────────────────────────┐
│  VAD (Voice Activity Detector)  │  ← built into tpl-spot-vad-lvcsr
│  Segments post-wake-word audio  │
└─────────────────────────────────┘
    │ on speech endpoint
    ▼
┌─────────────────────────────────┐
│  Slot 1: STT Recognizer (lvcsr) │  ← stt-enUS-automotive-medium-2.3.15-pnc.snsr
│  Transcribes command utterance  │
└─────────────────────────────────┘
    │
    ▼
 Transcript + NLU intent/entity output
```

**Key behavioral notes:**

- The STT model does not produce a final hypothesis until the VAD detects a speech endpoint (`^end`) or the audio stream ends. Partial results are emitted via `^result-partial`.
- With `include-wake-word-audio=1`, the audio buffer passed to the STT decoder includes the wake word. Whether the wake word appears in the transcript depends on the specific STT model configuration.
- The `-pp` profiling flag reports timing for the full pipeline including the VAD segmentation phase.

---

## 9. Quick Reference — Command Summary



```bash
# 1. Assemble the pipeline model (one time)
bin/snsr-edit -o vg-stt.snsr \
    -t model/tpl-spot-vad-lvcsr-3.23.0.snsr \
    -f 0 model/spot-voicegenie-enUS-6.5.1-m.snsr \
    -f 1 model/stt-enUS-automotive-medium-2.3.15-pnc.snsr \
    -s include-wake-word-audio=1

# 2. Basic profiling — aggregate RTF per slot (default partial-result-interval=1000ms)
bin/snsr-eval -p -t vg-stt.snsr data/audio/voice-genie-set-cruise-control.wav

# 3. Basic profiling — partial results disabled (lower-bound RTF)
bin/snsr-eval -p -t vg-stt.snsr -s partial-result-interval=0 \
    data/audio/voice-genie-set-cruise-control.wav

# 4. Detailed per-element profiling — partial results disabled
bin/snsr-eval -pp -t vg-stt.snsr -s partial-result-interval=0 \
    data/audio/voice-genie-set-cruise-control.wav

# 5. Detailed profiling with full recognition output
bin/snsr-eval -v -pp -t vg-stt.snsr -s partial-result-interval=0 \
    data/audio/voice-genie-set-cruise-control.wav

# 6. Measure worst-case heap memory — partial results disabled
valgrind --tool=massif --time-unit=B --massif-out-file=vg-stt.massif \
    bin/snsr-eval -t vg-stt.snsr -s partial-result-interval=0 \
    data/audio/voice-genie-set-cruise-control.wav
ms_print vg-stt.massif > vg-stt-pri0.txt

# 7. Measure worst-case heap memory — default partial results (1000 ms)
valgrind --tool=massif --time-unit=B --massif-out-file=vg-stt.massif \
    bin/snsr-eval -t vg-stt.snsr \
    data/audio/voice-genie-set-cruise-control.wav
ms_print vg-stt.massif > vg-stt-pri1000.txt

# 8. Generate noise test files (requires SoX)
sox -n -r 16000 -b 16 -c 1 data/audio/noise-10s.wav synth 10 whitenoise vol -80dB
sox -n -r 16000 -b 16 -c 1 data/audio/noise-100s.wav synth 100 whitenoise vol -80dB

# 9. Average RTF — wake word idle / continuous listening
bin/snsr-eval -p -t vg-stt.snsr data/audio/noise-10s.wav

# 10. Average MIPS — wake word (100s noise file, standalone wake word model)
perf stat bin/snsr-eval -t model/spot-voicegenie-enUS-6.5.1-m.snsr \
    data/audio/noise-100s.wav
#   Calculate: MIPS = instructions:u / 100 / 1,000,000

# 11. Average steady-state memory (run in Terminal 1, then steps below in Terminal 2)
bin/snsr-eval -t vg-stt.snsr
#   Terminal 2, Step 1 — get total RAM:  grep MemTotal /proc/meminfo
#   Terminal 2, Step 2 — get %MEM:       top -p $(pgrep snsr-eval)
#   Calculate:  Average RAM = %MEM × MemTotal
```

---

## 10. References

- TNL SDK 7.8 Docs: https://doc.sensory.com/tnl/7.8/
- `snsr-eval` reference: https://doc.sensory.com/tnl/7.8/tools/snsr-eval/
- `tpl-spot-vad-lvcsr` template: https://doc.sensory.com/tnl/7.8/models/tpl/tpl-spot-vad-lvcsr/
- STT model type: https://doc.sensory.com/tnl/7.8/models/types/stt/
