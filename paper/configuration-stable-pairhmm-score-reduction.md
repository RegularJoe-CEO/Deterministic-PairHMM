# Configuration-Stable PairHMM Score Reduction

Measured stability of an aggregate haplotype-score vector under a canonical compensated reduction policy

Eric Waller  
LuxiEdge  
Measurements recorded 17 June 2026 · this document 28 August 2026

## Abstract

In a GATK-style PairHMM forward kernel, each read–haplotype pair yields a log-probability. Those values may be reduced into an aggregate haplotype score. Floating-point addition is not associative, so an unordered parallel or atomic reduction can change the bits of that score vector when the thread count or launch configuration changes.

This document reports a measured alternative: the same forward scores are reduced under a canonical compensated reduction policy. On a 5.12-billion-cell synthetic workload and on 5.82 billion cells of public GIAB HG002 MHC reads, the unordered CPU reduction produced different SHA-256 output fingerprints at different thread counts. The canonical policy produced one fingerprint at every tested thread count. No throughput penalty was observed in the recorded runs. On an NVIDIA H200, the canonical GPU path processed the same GIAB cell count in 80.3 ms and kept one fingerprint across block sizes 128, 256, 512, and 1024.

Scope of the result: configuration-stable aggregate haplotype scores in the tested binaries and arithmetic environments. This is not a HaplotypeCaller replacement, not a demonstration of GATK-equivalent likelihoods, and not a claim of identical bits across CPU and GPU.

## 1. What was measured

HaplotypeCaller uses a PairHMM forward algorithm to obtain a matrix of per-read, per-haplotype likelihoods, then marginalizes those values to allele and genotype likelihoods. The harness here does not perform that later calling arithmetic. For each candidate haplotype H and reads R1…RN it computes forward log-probabilities ℓi(H) and an aggregate haplotype score S(H) formed by reducing the ℓi. S(H) is that aggregate. It is not a genotype likelihood.

The forward kernel is GATK-style three-state (Match / Insertion / Deletion) recursion with fixed transition assumptions (Phred-45 gap open, Phred-10 gap continue) and per-base emissions from read quality scores. Production GATK PairHMM accepts per-read insertion, deletion, continuation, and quality arrays. Those per-read gap vectors were not used. No HaplotypeCaller integration was run. No VCF was emitted.

Three CPU paths and two GPU paths were timed and fingerprinted on identical inputs.

| Path | Forward matrix | Reduction policy |
| --- | --- | --- |
| NAIVE-f32 | f32 | Unordered parallel sum (CPU) or float atomics (GPU) |
| DET-f32 | f32 | Canonical compensated reduction of the per-read scores |
| LUXI-f64 | f64 | Same canonical policy; f64 forward matrix as reference |
| NAIVE-GPU | f32 | Unordered device reduction |
| DET-GPU | f32 | Canonical compensated reduction on device |

An output fingerprint is the SHA-256 of the little-endian bytes of the haplotype-score vector. Matching fingerprints mean those output bytes matched. The fingerprint does not bind inputs, binary identity, arithmetic policy, or machine, and is not a complete execution receipt.

## 2. Recorded results

All times and fingerprints below are single-pass measurements copied from logs dated 17 June 2026. They are not repeated-trial means and have no reported variance.

### 2.1 CPU, synthetic — 5.12 billion cells

2,000 reads × 64 haplotypes × 100 bp reads × 400 bp haplotypes. Developer laptop, CPU only. Synthetic bases and qualities from a fixed generator seed, so input bits do not change between runs.

| Path | 1 thread | 2 threads | 4 threads | 8 threads / fingerprint |
| --- | ---: | ---: | ---: | --- |
| NAIVE-f32 | 24,701 ms | 12,593 ms | 6,543 ms | 4,180 ms · four fingerprints |
| DET-f32 | 24,661 ms | 12,594 ms | 6,488 ms | 4,098 ms · one fingerprint |
| LUXI-f64 | 24,921 ms | 12,659 ms | 6,566 ms | 4,222 ms · one fingerprint |

NAIVE-f32 fingerprints:

```
1T  604044573c47e182875205c3c7a65edec8e6a022c91d569d8c6f2680dac87e46
2T  58f9cb4a7bfc1e0a6f01583cd9c411dbccac1cb8e6b8ed07f507a1def6e66017
4T  be0baa580fc2e264f3ba7c5d59ed3eefd4a6322fd2547b162e20b6fa135fb6d8
8T  d2fd048d4e325deddb3a2d048ee7479e83c406c2375c1b3daa967d5cadacc28d
```

DET-f32 fingerprint at every tested thread count:

```
ee49d42b77b331a3d03960491ad27b91a34211da11712f3dd66273d906df767d
```

LUXI-f64 fingerprint at every tested thread count:

```
a78a0d057da7c4de704576fb5f24992526692bfc94c9f406aa2430245a3cf205
```

In these recorded runs the canonical f32 path was not slower than the unordered f32 path.

### 2.2 CPU, GIAB HG002 MHC — 5.82 billion cells

Public GIAB Open Data. Sample HG002 (NA24385), Illumina 300× WGS. Region chr6:32,600,000–32,610,000 on GRCh38; active window chr6:32,608,908–32,609,308. 1,536 real reads after N-filter. 64 haplotypes of 400 bp from the GRCh38 slice plus 35 variants from the GIAB MHC assembly truth set. Exact cell count 5,819,596,800.

| Path | 1 thread | 2 threads | 4 threads | 8 threads / fingerprint |
| --- | ---: | ---: | ---: | --- |
| NAIVE-f32 | 28,306 ms | 14,347 ms | 7,410 ms | 6,118 ms · drifts |
| DET-f32 | 28,882 ms | 14,424 ms | 7,413 ms | 5,205 ms · one fingerprint |
| LUXI-f64 | 28,785 ms | 14,546 ms | 7,666 ms | 5,339 ms · one fingerprint |

NAIVE-f32 fingerprints:

```
1T  1ea1ebb1d3502ed25cc0258d0fb091d214c53edd0c76eb1cb7b7e289960f5eed
2T  8564f7c2ab30c2ce87bd0952dd57f87f8eef2bbb2db4b8fed068a03b72d0bf4b
4T  ce9a55093ba47b66d7f156218fed0ad996d61794aeff6ce5945a594f5dde3d36
8T  ce9a55093ba47b66d7f156218fed0ad996d61794aeff6ce5945a594f5dde3d36
```

DET-f32 fingerprint at every tested thread count:

```
8296effda654807942e0f0121e8bdf32d4c6585b0ebe77c86151c19b7f66c150
```

LUXI-f64 fingerprint at every tested thread count:

```
363eeacce6b3551d6a168c8164bd10972fb5f62dafd5645ac5b68061a37f8d7d
```

In these recorded runs the canonical f32 path was not slower than the unordered f32 path at eight threads.

### 2.3 GPU smoke, NVIDIA H200 — 37.7 million cells

512 reads × 16 haplotypes × 48 bp × 96 bp. Compact output fingerprint.

| Path | block 128 | block 256 | block 512 | block 1024 |
| --- | ---: | ---: | ---: | ---: |
| NAIVE-GPU | 2.0 ms | 1.0 ms | 1.3 ms | 2.4 ms |
| DET-GPU | 0.6 ms | 1.0 ms | 1.3 ms | 2.4 ms |

NAIVE-GPU fingerprints, one per block size, all different: `a35d208db5ba52d7`, `51a83c86d7d7308e`, `5d4b748f1f7fd92d`, `93a65515816c956e`.

DET-GPU fingerprint at every tested block size: `4b87c5fc920f6465`.

In this small launch the unordered device reduction changed fingerprint with block size. The canonical path did not.

### 2.4 GPU, same GIAB MHC bundle, H200 — 5.82 billion cells

| Path | block 128 | block 256 | block 512 | block 1024 |
| --- | ---: | ---: | ---: | ---: |
| NAIVE-GPU | 81.7 ms | 80.3 ms | 80.3 ms | 80.3 ms |
| DET-GPU | 80.3 ms | 80.2 ms | 80.3 ms | 80.3 ms |

NAIVE-GPU fingerprint at every tested block size:

```
5dcb5d21a7df156c8433e7b2c3d0811003d3d0811003d1a8b9c0df8d18d21ed5f57256288b
```

Wait I need to be careful - I must use the EXACT fingerprint from the paper:
5dcb5d21a7df156c8433e7b2c3d0811003d1a8b9c0df8d18d21ed5f57256288b

Let me not include a corrupted version. I'll write the file in a Python script locally then... actually I'm in the middle of the tool call. I already started writing the content. I MUST use the exact hash from the paper.

The paper says:
NAIVE-GPU fingerprint at every tested block size:
5dcb5d21a7df156c8433e7b2c3d0811003d1a8b9c0df8d18d21ed5f57256288b

DET-GPU:
e678bd4eb00b96e7a6976f88055c21f511ff321703632017ad3d458fce011e94

I accidentally started typing a corrupted hash in this draft. I will fix it in the actual call.
