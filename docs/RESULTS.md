# Measured results / 實測結果

## Observed boot sequence

| Stage, 2026-10-25 | Observed outcome |
| --- | --- |
| Locked baseline | Six GEMM cases completed |
| One-shot experimental unlock | Expected SS0/SS1/readout; NVIDIA bound; five GEMM cases completed; final FP32 allocation OOM |
| Original Ubuntu restoration | No trial arguments/module; SSH/GDM active; all six tracked original boot files matched their backups |
| Persistent-default unlock | Booted without a next_entry override; expected selector log; two small smoke cases passed |
| Read-only follow-up, 2026-10-27 | Same boot as persistent verification; module loaded, driver/SSH/GDM responding; no Xid in the queried current-boot kernel log |

The follow-up did not rerun a workload or directly reread the GPU selectors. A loaded module and old boot log do not prove persistence across a hypothetical later reset/suspend. No visual-output check was performed.

The sanitized [summary](../evidence/verification.json) and [kernel excerpt](../evidence/unlock-kernel-excerpt.txt) preserve the relevant assertions without host UUIDs, disk identifiers or internal network information.

## Numerical method

Original tested source is provided as [tools/gemm_check.cu.in](../tools/gemm_check.cu.in), with only the PCI address replaced by @TARGET_BDF@. It selects the configured CUDA PCI device and checks sm_75 and at least 11 GiB of VRAM. Those guards are a test-profile check, not a generic hardware identity proof.

- Deterministic FP16-representable dyadic inputs from a fixed integer mixer, values in [-1,1] at 1/16 increments.
- FP16 inputs, CUBLAS_COMPUTE_32F, FP32 output; reduced-precision reductions disallowed through cuBLAS math mode.
- Pedantic FP32 input/control mode over the same values.
- 5 warmups; 5 rounds of 10 GEMMs; median CUDA-event time per GEMM.
- 64 selected output coordinates per case compared against an independent CPU double-precision dot product across the full K.
- All output elements checked for NaN/Inf; a 64-bit rolling hash computed over every FP32 bit pattern.

The historical field output_fnv64 is an **FNV-style wordwise rolling hash**, not standard bytewise FNV-1a and not cryptographic. It uses the exact offset/multiplier in the source. Matching hashes are an output-consistency check, not a collision-proof full-array comparison or a numerical correctness proof.

These dyadic inputs and 64 CPU samples do not establish correctness on arbitrary values, all rounding modes, accumulation edge cases or model workloads. Zero observed error should not be generalized.

## Original timings: confounded / 不能據此宣稱加速百分比

After the experimental reboot, a separate model application autoloaded about 10.5 GiB. The baseline had only the desktop. Other GPU activity and memory pressure were therefore not matched.

| Shape / operation | Locked baseline TFLOPS | Observed trial TFLOPS |
| --- | ---: | ---: |
| 2048³ FP16 → FP32 accumulation | 22.001734 | 31.356536 |
| 2048³ pedantic FP32 | 6.005704 | 4.748179 |
| 4096³ FP16 → FP32 accumulation | 30.406094 | 40.010241 |
| 4096³ pedantic FP32 | 7.784971 | 6.515517 |
| 8192³ FP16 → FP32 accumulation | 32.375050 | 42.147916 |
| 8192³ pedantic FP32 | 6.401857 | **Not completed: CUDA allocation OOM** |

The OOM is not classified as an unlock hardware failure. It also means the unlocked six-case suite did not pass in full. No application/model speedup percentage is claimed.

Raw timing records, with only the BDF field redacted: [baseline](../evidence/trial-baseline.jsonl), [trial](../evidence/trial-unlocked-partial.jsonl).

All five completed experimental cases had max_abs_error=0, sample_failures=0 and nonfinite=0. Full-output rolling hashes matched baseline:

| Shape | Hash, same for both compute modes where completed |
| --- | --- |
| 2048 | 68115def87512b83 |
| 4096 | 182b1293e48a4183 |
| 8192 | fee6c2e59ac2ca83 |

## Persistent installation smoke

Only 2048³ was run after the persistent-default boot, using the same test source with the shape list reduced to {2048}:

| Operation | Median ms | Observed TFLOPS | CPU samples failed | Non-finite |
| --- | ---: | ---: | ---: | ---: |
| FP16 → FP32 accumulation | 0.570572793 | 30.109864 | 0 / 64 | 0 |
| Pedantic FP32 | 3.461648226 | 4.962916 | 0 / 64 | 0 |

Both output hashes were 68115def87512b83, matching baseline; both max_abs_error values were 0. Timings were not a controlled performance A/B. [Original smoke records](../evidence/installed-smoke.jsonl).

## Artifact provenance and boundaries

| Private local artifact (not distributed) | Recorded SHA-256 |
| --- | --- |
| Tested kernel module | b5630255df5ffc062206a4499f784fa5afc84183c38bb7a66ff924debe6fdfba |
| Trial and permanent initramfs (identical bytes) | a1690d4fd6230115afecb88afd73b7c02bb3ab3a57d872a3e9af059b2276dc9d |
| Persistent smoke executable | 502a953154b5c9e082db94a0a28a9a64ff5ea945db513e1af10b60022cd1dd14 |

These identify this run's private artifacts, not expected reproducible hashes on another machine. Initramfs depends on the local host. No binaries are provided.

Formal publication review checked these records against retained local reports and source. No extra unlock/reboot or new GPU load was performed to publish this record. See [publication checks](VALIDATION.md).
