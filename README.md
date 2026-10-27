# RTX 2060 12 GB — Linux Turing unlock reproduction

[繁體中文](README.zh-TW.md) · [Method](docs/REPRODUCE.zh-TW.md) · [Results](docs/RESULTS.md) · [Recovery](docs/RECOVERY.zh-TW.md) · [Credits](docs/REFERENCES.md)

A measured reproduction of [nlzy/turing-unlock](https://github.com/nlzy/turing-unlock) on **one RTX 2060 12 GB / TU106 / PCI 10de:1f03**. This repository records our integration, verification and recovery method. The GPU exploit and unlock implementation belong to the upstream authors.

**Verified:** one experimental unlock boot, a return to ordinary Ubuntu, and a separate persistent-default unlock boot. NVIDIA, SSH and the graphical login service were healthy; the numerical checks described below passed. **No reliable application speedup or long-term stability claim.**

| Measured configuration | Value |
| --- | --- |
| GPU | RTX 2060 12 GB, TU106, sm_75, PCI 10de:1f03 |
| OS / kernel | Ubuntu 26.04.1 / 7.0.0-38-generic |
| NVIDIA driver / power limit | 595.99.02 / 184 W |
| Secure Boot | Disabled |
| Upstream commit | [1d379b724454d6fb7e8a7a8dd0ea359febc05ca3](https://github.com/nlzy/turing-unlock/commit/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3) |
| Unlock readback | SS0=0x88888888, SS1=0x8, readout=0 |
| Integration | Dedicated dracut initramfs, targeted pre-udev insmod, separate GRUB entry |
| Persistence | Reapply at boot; original Ubuntu kernel/initramfs remain selectable |

The dedicated image contains the module and six firmware assets. We did not install a global DKMS/module setup, patch the NVIDIA driver or flash vBIOS. The permanent entry pins the tested kernel; future kernel/driver changes need validation.

## What is established

- The supported-ID list includes this desktop TU106 device. This is evidence for our measured board/configuration, not every RTX 2060 or laptop.
- The module reported the expected selector values before NVIDIA bound the card.
- Five completed experimental GEMM cases passed 64 CPU-reference samples each, had no non-finite output, and matched the baseline's full-output rolling hashes.
- The persistent boot's 2048³ FP16/FP32-accumulate and pedantic FP32 smoke checks passed.
- A normal restoration boot was actually observed before permanent installation. Original boot-file hashes matched after trial cleanup.

The original trial was **not a clean performance A/B**: a model application autoloaded about 10.5 GiB after reboot, and the final 8192³ FP32 allocation failed with OOM. Raw timings are retained in [Results](docs/RESULTS.md) for transparency and must not be advertised as an isolated unlock gain.

## Use the record

1. Read the [measured procedure](docs/REPRODUCE.zh-TW.md) and [recovery limits](docs/RECOVERY.zh-TW.md).
2. Build the pinned upstream source and obtain its hash-checked firmware locally.
3. Adapt the [.in templates](examples/01-turing-unlock-trial.sh.in) to your own PCI address and boot configuration.
4. Validate one-shot boot and restoration before considering a persistent default.

This is an experimental operations record, not a universal installer. Firmware execution, DMA and resets are privileged operations; failure can require physical power removal, and upstream warns of possible hardware damage from higher tensor power demand. A GRUB fallback or Linux timer cannot recover a machine that never reaches an operable boot state.

## Publication scope

Original notes, source templates and redacted evidence only. No firmware binaries, kernel-module binaries, initramfs, ROM dump, credentials, internal IPs or host UUIDs are distributed. Original material is [MIT licensed](LICENSE); upstream code and NVIDIA firmware keep their own terms. See [references and provenance](docs/REFERENCES.md) and [publication checks](docs/VALIDATION.md).
