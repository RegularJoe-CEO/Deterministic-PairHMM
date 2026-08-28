# Closed evaluation binaries

Linux x86_64 CPU harness. No source. Evaluation license: see [LICENSE](../LICENSE).

| File | SHA-256 |
| --- | --- |
| `pairhmm-determinism-linux-x86_64` | `ef7a33add60e86e729d981dfc2ef5556914cc3968727ffd5fdb3ade9131afc83` |

```bash
chmod +x pairhmm-determinism-linux-x86_64
shasum -a 256 -c pairhmm-determinism-linux-x86_64.sha256
./pairhmm-determinism-linux-x86_64 --synthetic
```

No flags is the same synthetic 5.12-billion-cell workload as `--synthetic`. `--giab` needs a `data/giab_mhc` directory from public GIAB.

This binary is not the LuxiEdge engine. GPU and macOS builds are not in this folder yet.
