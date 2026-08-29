# Deterministic-PairHMM

Public paper and **closed evaluation binaries** for a configuration-stable GATK-style PairHMM aggregate haplotype-score reduction.

This is **not** the LuxiEdge engine. The binaries let you run the published workload and check the SHA-256 fingerprint. They do not include, and do not license, LuxiEdge, LuxiQuant, or the Luxi Inference Engine. Source is not in this repository.

When you reduce per-read PairHMM log-probabilities into a haplotype score vector, unordered parallel or atomic addition can change the bits of that vector when thread count or GPU block size changes. The paper reports a measured alternative: the same forward scores, reduced under a canonical compensated reduction policy.

**Paper:** [Configuration-Stable PairHMM Score Reduction](paper/configuration-stable-pairhmm-score-reduction.md)  
Eric Waller, LuxiEdge. Measurements 17 June 2026. This document 28 August 2026.

**License:** [LICENSE](LICENSE) — all rights reserved. Evaluation use of the binaries only.

## What this is

- A GATK-style three-state PairHMM forward kernel (Match / Insertion / Deletion) plus an aggregate haplotype score `S(H)` formed by reducing the per-read log-probabilities.
- A recorded result: on a 5.12-billion-cell synthetic workload and on 5.82 billion cells of public GIAB HG002 MHC reads, the unordered CPU reduction produced different fingerprints at different thread counts. The canonical policy produced one fingerprint at every tested thread count. No throughput penalty was observed in the recorded runs.
- On an NVIDIA H200, the canonical GPU path processed the same GIAB cell count in 80.3 ms and kept one fingerprint across block sizes 128, 256, 512, and 1024.

## What this is not

- Not a HaplotypeCaller replacement.
- Not a demonstration of GATK-equivalent likelihoods.
- Not a claim of identical bits across CPU and GPU.
- Not a genotype caller. No VCF is emitted. Production GATK per-read gap vectors were not used.
- Not open source. Not the LuxiEdge engine.

## Binaries

Closed evaluation binaries live in [`bin/`](bin/).

| File | Platform | SHA-256 |
| --- | --- | --- |
| [pairhmm-determinism-linux-x86_64](bin/pairhmm-determinism-linux-x86_64) | Linux x86_64 CPU | `ef7a33add60e86e729d981dfc2ef5556914cc3968727ffd5fdb3ade9131afc83` |

This Linux build was made 28 August 2026. It is a closed evaluation binary, not engine source. GPU and macOS builds are separate artifacts.

## Contact

Eric Waller · [e@ewaller.com](mailto:e@ewaller.com)
