# Credits, pinned sources and license scope

## Implementation credit

This repository records our reproduction and Linux boot integration on one RTX 2060 12 GB. We did not discover the underlying GPU vulnerability or author the upstream unlock payload.

- [nlzy/turing-unlock, measured commit 1d379b724454d6fb7e8a7a8dd0ea359febc05ca3](https://github.com/nlzy/turing-unlock/tree/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3): the actual kernel module, firmware generation, GPU state handling and Turing payload used here.
- [Pinned upstream README](https://github.com/nlzy/turing-unlock/blob/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3/README.md): supported operation, original test board, limitations and warnings.
- [Pinned PCI-ID list](https://github.com/nlzy/turing-unlock/blob/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3/kernel/turing_unlock/main.c): includes TU106 device 0x1f03 and opt-in target selection.
- [Pinned firmware provenance notice](https://github.com/nlzy/turing-unlock/blob/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3/firmware/NOTICE.md): NVIDIA assets and distribution scope.
- [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock): the upstream work explicitly credited by nlzy's Turing adaptation.
- [Architecture differences](https://github.com/nlzy/turing-unlock/blob/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3/docs/Difference%20between%20Ampere%20and%20Turing.md): why payload layouts cannot simply be interchanged.

## Later reports, not our measurements

As reviewed on 2026-10-27:
- [TU106 RTX 2070 report #4](https://github.com/nlzy/turing-unlock/issues/4): same-card instruction/GEMM and sampled correctness claims, with explicit short-test limits.
- [RTX 2070 SUPER report #6](https://github.com/nlzy/turing-unlock/issues/6): GEMM gain and a much smaller real MoE prompt-processing gain; token generation largely unchanged.
- [RTX 2060 Mobile report #15](https://github.com/nlzy/turing-unlock/issues/15): locally added laptop PCI IDs; not a blanket laptop-support guarantee.
- [Arch packaging changes after our source commit](https://github.com/nlzy/turing-unlock/compare/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3...417f333263ff169e8f449c931b70db9dfd7b8ef4): two commits changed mkinitcpio packaging/pkgrel, not the kernel unlock code.

These are external self-reports and mutable discussions. Their results are not incorporated into our measured evidence and were not independently reproduced by us.

## License scope

The MIT license in this repository applies to our original documentation, locally authored CUDA test template, boot-hook template and redacted evidence presentation. It does not relicense upstream code, the GPU exploit implementation, NVIDIA firmware or any third-party material.

The upstream kernel module/packaging is GPL-2.0-only, and NVIDIA firmware retains the terms described upstream. Build those assets locally from the pinned upstream source and retain its notices. We distribute no upstream kernel source, firmware binary, .ko, initramfs, ROM dump or CUDA runtime library here.
