# Configuration-Stable PairHMM Score Reduction

Measured stability of an aggregate haplotype-score vector under a canonical compensated reduction policy

Eric Waller  
LuxiEdge  
Measurements recorded 17 June 2026 · this document 28 August 2026 · citation metadata 28 August 2026

## Abstract

In a GATK-style PairHMM forward kernel, each read–haplotype pair yields a log-probability. Those values may be reduced into an aggregate haplotype score. Floating-point addition is not associative, so an unordered parallel or atomic reduction can change the bits of that score vector when the thread count or launch configuration changes.

This document reports a measured alternative: the same forward scores are reduced under a canonical compensated reduction policy. On a 5.12-billion-cell synthetic workload and on 5.82 billion cells of public GIAB HG002 MHC reads, the unordered CPU reduction produced different SHA-256 output fingerprints at different thread counts. The canonical policy produced one fingerprint at every tested thread count. No throughput penalty was observed in the recorded runs. On an NVIDIA H200, the canonical GPU path processed the same GIAB cell count in 80.3 ms and kept one fingerprint across block sizes 128, 256, 512, and 1024.

Scope of the result: configuration-stable aggregate haplotype scores in the tested binaries and arithmetic environments. This is not a HaplotypeCaller replacement, not a demonstration of GATK-equivalent likelihoods, and not a claim of identical bits across CPU and GPU.


## How to cite

Waller, E. (2026). *Configuration-Stable PairHMM Score Reduction* (Version 1.0.0). Zenodo. https://doi.org/10.5281/zenodo.22152053

```
@misc{waller2026pairhmm,
  author       = {Waller, Eric},
  title        = {Configuration-Stable {PairHMM} Score Reduction},
  year         = {2026},
  month        = aug,
  howpublished = {Zenodo},
  doi          = {10.5281/zenodo.22152053},
  url          = {https://doi.org/10.5281/zenodo.22152053},
  note         = {Methods note and closed Linux evaluation binary. No source.}
}
```

Correspondence: Eric Waller, LuxiEdge, e@ewaller.com.

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
5dcb5d21a7df156c8433e7b2c3d0811003d1a8b9c0df8d18d21ed5f57256288b
```

DET-GPU fingerprint at every tested block size:

```
e678bd4eb00b96e7a6976f88055c21f511ff321703632017ad3d458fce011e94
```

On this full-scale GIAB launch the unordered GPU path did not change fingerprint across block sizes. The unordered method can drift, as Section 2.3 shows. It did not drift in Section 2.4.

DET-GPU processed 5,819,596,800 cells in 80.3 ms on this H200 run, about 72 billion cells per second. CPU DET-f32 at eight threads on the same bundle was 5,205 ms, about 1.1 billion cells per second.

The CPU DET-f32 fingerprint in Section 2.2 and the GPU DET fingerprint in this section are different. Cross-device bit identity was not obtained in these binaries.

## 3. What the result is

In the tested CPU binaries, an unordered reduction of PairHMM aggregate haplotype scores is not configuration-stable across thread counts 1, 2, 4, and 8, on both the synthetic workload and the GIAB HG002 MHC window. A canonical compensated reduction policy is configuration-stable across those thread counts. No throughput penalty was observed in the recorded runs.

In the tested H200 binaries, the canonical path is configuration-stable across block sizes 128–1024 on both the smoke workload and the GIAB window, and finishes the GIAB window in 80.3 ms. The unordered GPU path is not always unstable: it drifted in the smoke launch and did not drift on the full GIAB launch.

The result does not establish identical scores across CPU and GPU, across GPU architectures, across compilers or optimization flags, across FMA versus non-FMA arithmetic, across flush-to-zero or subnormal settings, or across math-library implementations. It does not establish GATK-equivalent likelihoods or unchanged variant calls.

## 4. Data

GIAB HG002 reads and the MHC haplotype slice are public Genome in a Bottle resources. Synthetic inputs are generated from a fixed seed and contain no private sequence. Workload sizes, region coordinates, cell counts, wall-clock times, and output fingerprints are stated in Section 2 so the recorded runs can be compared against a later independent implementation without requiring the original source tree.

## 5. Reproducibility of the published binary

Independent check of this result does not require engine source.

The Linux x86_64 evaluation binary in `bin/` is a closed artifact. Evaluation use is limited to running that binary and comparing output fingerprints to this document (see LICENSE).

```
chmod +x pairhmm-determinism-linux-x86_64
shasum -a 256 -c pairhmm-determinism-linux-x86_64.sha256
./pairhmm-determinism-linux-x86_64 --synthetic
```

`--synthetic` is the 5.12-billion-cell workload of Section 2.1. If DET-f32 prints

```
ee49d42b77b331a3d03960491ad27b91a34211da11712f3dd66273d906df767d
```

the reduction on that run matches the recorded June 17 measurement. If it does not, the mismatch is the result. Correspondence: e@ewaller.com.

The published Linux binary was built 28 August 2026. The times and fingerprints in Section 2 are the 17 June 2026 recorded runs. GPU and macOS binaries are not in this repository.

`--giab` requires a local `data/giab_mhc` directory assembled from public GIAB resources. Those reads are not redistributed here.

## 6. Data availability

GIAB HG002 (NA24385) Illumina 300× WGS reads and the MHC haplotype slice are public Genome in a Bottle resources [2]. Synthetic inputs are generated from a fixed seed inside the closed binary and contain no private sequence. Workload sizes, region coordinates, cell counts, wall-clock times, and output fingerprints are stated in Section 2 so a later independent implementation can be compared without the original source tree.

## 7. Software and intellectual property

This repository does not contain source code. Running the binary does not convey source, and it does not license the LuxiEdge engine, LuxiQuant, the Luxi Inference Engine, or any other Luxi product.

What is public here is the measurement, the named reduction *policy* (unordered parallel reduction versus a canonical compensated reduction in fixed read-index order), and a closed evaluation binary that prints the fingerprints. Compensation of floating-point sums is a public numerical technique [3]. The LuxiEdge engine that produced the recorded runs is not published and is not licensed by this document.

This is not a HaplotypeCaller [1], not a GATK-equivalent likelihood claim, and not a genotype caller.

## References

1. Poplin, R., Ruano-Rubio, V., DePristo, M. A., Fennell, T. J., Carneiro, M. O., Van der Auwera, G. A., Kling, D. E., Gauthier, L. D., Levy-Moonshine, A., Roazen, D., Shakir, K., Thibault, J., Chandran, S., Whelan, C., Lek, M., Gabriel, S., Daly, M. J., Neale, B., MacArthur, D. G., & Banks, E. (2018). Scaling accurate genetic variant discovery to tens of thousands of samples. *bioRxiv*. https://doi.org/10.1101/201178

2. Zook, J. M., Catoe, D., McDaniel, J., Vang, L., Spies, N., Sidow, A., Weng, Z., Liu, Y., Mason, C. E., Alexander, N., Henaff, E., McIntyre, A. B. R., Chandramohan, D., Chen, F., Jaeger, E., Moshrefi, A., Pham, K., Stedman, W., Liang, T., … Salit, M. (2016). Extensive sequencing of seven human genomes to characterize benchmark reference materials. *Scientific Data, 3*, 160025. https://doi.org/10.1038/sdata.2016.25

3. Kahan, W. (1965). Pracniques: further remarks on reducing truncation errors. *Communications of the ACM, 8*(1), 40. https://doi.org/10.1145/363707.363723

