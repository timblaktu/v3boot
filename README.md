# V3000 Bootloader Project

Minimal Rust bootloader for AMD Ryzen Embedded V3000 (V3C18I) platform, following Oxide Computer's proven phbl architecture.

## Project Goals

- **Boot time**: <1 second to U-Boot entry
- **Code size**: <5,000 lines Rust
- **Security**: 99% attack surface reduction vs UEFI
- **Reliability**: Memory-safe Rust, no heap allocation

## Documentation

### Core Documents

| Document | Description |
|----------|-------------|
| [NDA_INDEPENDENT_WORK_PLAN.md](NDA_INDEPENDENT_WORK_PLAN.md) | Actionable implementation plan (85-90% completable without NDA) |
| [PLATFORM_COMPARISON.md](PLATFORM_COMPARISON.md) | EPYC vs V3000 architecture analysis |
| [ARCHITECTURE_DIAGRAMS.md](ARCHITECTURE_DIAGRAMS.md) | Visual system documentation |

### Reference Documents

| Document | Description |
|----------|-------------|
| [CLAUDE-phbl.md](CLAUDE-phbl.md) | phbl repository context |
| [CLAUDE-amd-host-image-builder.md](CLAUDE-amd-host-image-builder.md) | Flash image builder context |
| [CLAUDE-helios.md](CLAUDE-helios.md) | Boot protocol reference |
| [CLAUDE-hubris.md](CLAUDE-hubris.md) | Embedded patterns reference |
| [CLAUDE-v3000-bootloader.md](CLAUDE-v3000-bootloader.md) | Implementation repository context |

### Research Documents

| Document | Description |
|----------|-------------|
| [PHBL_RESEARCH_REPORT.md](PHBL_RESEARCH_REPORT.md) | Comprehensive phbl analysis |
| [PHBL_QUICK_REFERENCE.md](PHBL_QUICK_REFERENCE.md) | phbl quick lookup guide |
| [HELIOS_BOOT_PROTOCOL_RESEARCH.md](HELIOS_BOOT_PROTOCOL_RESEARCH.md) | Boot handoff protocol details |
| [HELIOS_QUICK_REFERENCE.md](HELIOS_QUICK_REFERENCE.md) | Protocol specification |
| [RESEARCH_SUMMARY.md](RESEARCH_SUMMARY.md) | Executive overview |

## Key Findings

### Code Reusability

- **98% of phbl code is reusable** - Only ~35 lines need V3000-specific changes
- **FCH addresses are standardized** - UART (0xFEDC_9000), GPIO (0xFED8_0000)
- **CPUID is known** - Family 0x19, Model 0x40-0x4F (Rembrandt-based)

### Hard Blockers (NDA Required)

1. PSP firmware blobs (EmbeddedPi branch)
2. AGESA binary (FP7r2)
3. APCB template from AMD FAE

### Work That Can Proceed Now

- Build system setup
- CPU mode transitions (16→32→64-bit)
- Page table construction
- ELF loading and CPIO handling
- UART driver (use standard FCH addresses)
- QEMU testing

## Quick Start

```bash
# Clone Oxide repositories (forks)
git clone https://github.com/timblaktu/phbl
git clone https://github.com/timblaktu/amd-host-image-builder

# Set up Rust toolchain
rustup target add x86_64-unknown-none

# Study the key files
cat phbl/src/iomux.rs      # CPUID detection
cat phbl/src/uart.rs       # UART driver
cat phbl/src/mmu.rs        # Page tables
```

## Architecture Overview

```
Power On
    │
    ▼
┌─────────────────┐
│ PSP Boot ROM    │  ARM Cortex-A5
│ (immutable)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ IPL → ABL0-7    │  AGESA memory init
│ (flash)         │
└────────┬────────┘
         │ DRAM ready
         ▼
┌─────────────────┐
│ V3000           │  x86_64, Rust
│ Bootloader      │  Mode transitions
│ (this project)  │  Page tables, UART
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ U-Boot          │  System bootloader
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Linux           │  Operating system
└─────────────────┘
```

## References

- [Oxide phbl](https://github.com/oxidecomputer/phbl)
- [Oxide amd-host-image-builder](https://github.com/oxidecomputer/amd-host-image-builder)
- [PSPTool](https://github.com/PSPReverse/PSPTool)
- [Coreboot AMD](https://doc.coreboot.org/soc/amd/)
- [SolidRun HoneyComb](https://solidrun.atlassian.net/wiki/spaces/developer/)

## License

See individual Oxide repositories for license terms (MPL-2.0).
