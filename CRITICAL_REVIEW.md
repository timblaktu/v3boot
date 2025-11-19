# Critical Review and Corrections

This document identifies assumptions requiring verification, corrects potential errors, and provides foundational explanations for team members new to AMD boot architecture.

---

## 1. Assumptions Requiring Verification

### HIGH RISK - May Be Incorrect

| Assumption | In Document | Concern | Verification Method |
|------------|-------------|---------|---------------------|
| **V3000 CPUID is 0x19/0x40-0x4F** | Multiple | Based on Rembrandt mobile. V3000 embedded variant may have different model ID | Check AMD product brief or CPUID on hardware |
| **~98% code reusable** | NDA_INDEPENDENT | Too optimistic. More realistic: 80% generic x86 | Review actual phbl code line by line |
| **UART at 0xFEDC_9000** | Multiple | Assumed same as EPYC FCH. Embedded FCH may differ | Dump firmware with PSPTool or check datasheet |
| **GPIO at 0xFED8_0000** | Multiple | Same concern as UART | Same verification method |
| **Pins 135-138 for UART** | Multiple | SP3/SP5 socket pins. FP7r2 BGA likely different | Schematic or datasheet required |
| **Boot timing estimates** | ARCHITECTURE_DIAGRAMS | Based on EPYC. V3000 has simpler memory (2 ch vs 8-12) | Measure on hardware |

### MEDIUM RISK - Likely Correct But Unconfirmed

| Assumption | Basis | Confidence |
|------------|-------|------------|
| Same PSP→ABL→x86 boot flow | All AMD Zen platforms use this | High |
| DDR5 support | V3000 product brief states DDR5 | High |
| EmbeddedPi AGESA branch | Based on Rembrandt family | Medium |
| 16550A UART IP | Standard across AMD FCH | High |
| IO mux offset +0x0D00 | Consistent in phbl code | Medium |

### CORRECTED ESTIMATES

| Metric | Previous Claim | Corrected Estimate | Reasoning |
|--------|----------------|-------------------|-----------|
| Code reusability | 98% | **80-85%** | Generic x86 is 80%, platform-specific is 15-20% |
| Lines to change | ~35 | **100-300** | More conservative; includes testing/debugging |
| Time to QEMU boot | 2-3 weeks | **4-6 weeks** | More realistic for careful development |
| Integration time | 3-5 days | **1-2 weeks** | Includes debugging on real hardware |

---

## 2. Foundational Concepts (For Teams New to AMD)

### 2.1 Why Does AMD Have a PSP?

**Problem**: Modern CPUs need complex initialization before x86 code can run:
- Memory controller must be trained (DDR timing calibration)
- Power management must be configured
- Security features must be established
- Silicon errata must be worked around

**Solution**: AMD includes a separate ARM processor (Platform Security Processor) that:
- Runs before x86 cores wake up
- Initializes DRAM so x86 has memory to run from
- Provides cryptographic services (fTPM, key storage)
- Cannot be disabled or replaced

**Implications for V3000 bootloader**:
- We cannot bypass PSP - it must run first
- DRAM is already initialized when our code runs
- We need AMD firmware blobs for PSP to function

### 2.2 What is AGESA?

**AGESA** = AMD Generic Encapsulated Software Architecture

Think of it as AMD's "BIOS SDK" - proprietary code that:
- Initializes CPU, memory controller, and chipset
- Runs as ABL (AGESA Boot Loader) stages 0-7 on the PSP
- Writes results to APOB (AGESA PSP Output Block)

**Why can't we write our own?**
- Memory training algorithms are AMD trade secrets
- Requires detailed knowledge of DDR PHY internals
- Would take years to develop and test
- AMD provides binaries under NDA

**Future**: AMD openSIL (2026+) will open-source this for newer platforms.

### 2.3 What is FCH?

**FCH** = Fusion Controller Hub

The "southbridge" integrated into AMD SoCs containing:
- UART controllers (serial ports)
- GPIO banks (general purpose I/O)
- USB, SATA, SPI controllers
- I2C/SMBus interfaces

**Why it matters**: Our bootloader communicates via FCH UART. The FCH register addresses are what we need from AMD documentation.

**EPYC vs V3000**: EPYC has a separate I/O Die; V3000 has FCH integrated into the monolithic SoC. The registers may be at the same addresses but this is not guaranteed.

### 2.4 What is the Bit 11 Protocol?

**Context**: When the bootloader hands off to the kernel, it provides page tables mapping virtual to physical memory.

**Problem**: The kernel needs to know which page table entries it "owns" vs which are temporary bootloader mappings it can reclaim.

**Solution**: Bit 11 in Page Table Entries (PTEs)
- This is an "available" bit (hardware ignores it)
- Bootloader sets bit 11 = 1 on all kernel pages
- Kernel knows: bit 11 set = mine, bit 11 clear = reclaim

**Why this matters**: If we don't set bit 11 correctly, the kernel may:
- Fail to boot
- Corrupt its own page tables
- Have memory leaks

### 2.5 What is APCB vs APOB?

**APCB** (AMD PSP Customization Block):
- **Input** to AGESA
- Board-specific configuration (memory topology, GPIO settings)
- Created by board manufacturer
- Stored in flash, read by PSP

**APOB** (AGESA PSP Output Block):
- **Output** from AGESA
- Results of memory training
- Written by PSP to reserved memory region
- Can be cached to flash for faster subsequent boots

**Why this matters**: We need an APCB template from AMD FAE for our board. Without it, AGESA won't know how to configure memory.

### 2.6 PSP Directory vs BHD Directory

Both are tables in flash that tell PSP where to find firmware:

**PSP Directory** ($PSP, $PL2):
- Contains PSP-specific firmware
- Boot loaders, security OS, SMU firmware
- Runs on ARM core

**BHD Directory** (BIOS Header Directory):
- Contains x86-related components
- APCB, APOB location, microcode patches
- Our bootloader goes here (Reset Image entry)

**Simplified view**:
```
Flash Image
├── EFS Header (tells PSP where directories are)
├── PSP Directory
│   ├── PSP Boot Loader
│   ├── ABL0-7 (AGESA stages)
│   └── SMU Firmware
└── BHD Directory
    ├── APCB (board config)
    ├── APOB (training results)
    ├── Microcode
    └── Reset Image (OUR BOOTLOADER)
```

---

## 3. Document-Specific Corrections

### 3.1 NDA_INDEPENDENT_WORK_PLAN.md

**Issue**: Claims "85-90% completable without NDA"

**Correction**: More accurate is 70-80%. While generic x86 code works, we still need:
- Correct CPUID model (must verify)
- Correct pin configuration (must verify)
- Memory map details for optimization

**Issue**: Claims integration is "3-5 days"

**Correction**: First hardware bringup typically takes 1-2 weeks due to:
- Debug cycles with serial console
- Unexpected platform differences
- APCB tuning

### 3.2 PLATFORM_COMPARISON.md

**Issue**: States "V3000 is Family 19h, Model 0x40-0x4F (Rembrandt-based)"

**Correction**: Add caveat: "Based on Rembrandt mobile silicon. Embedded V3000 may have different model ID. Verify with hardware CPUID or AMD documentation."

**Issue**: "~35 lines to change"

**Correction**: This underestimates testing, error handling, and debug code. More realistic: 100-300 lines including platform detection, error paths, and configuration.

### 3.3 ARCHITECTURE_DIAGRAMS.md

**Issue**: Boot timing estimates (PSP ~50-100ms, etc.)

**Correction**: Add note: "Timing estimates are approximate and based on EPYC platforms. V3000 may differ due to simpler memory configuration (2 channels vs 8-12)."

**Issue**: Memory map addresses shown as exact values

**Correction**: Mark addresses with "(verify for V3000)" where they're assumed from EPYC.

### 3.4 COMMUNITY_INSIGHTS.md

**Issue**: Generation detection bits (Turin=0xe3, Milan=0xfc, Genoa=0xfe)

**Correction**: Add explanation of what these bits are:
"These are processor generation identifier bits from CPUID/EFS that help the firmware distinguish platforms. They're not CPUID family/model but a separate encoding used by AMD tools. V3000's bits are unknown and must be determined from firmware analysis or documentation."

---

## 4. Missing Explanations to Add

### 4.1 Why Minimal Bootloader?

Teams may ask: "Why not use UEFI or coreboot?"

**Answer**:
- **UEFI**: ~1.5M lines of code, huge attack surface, slow boot
- **Coreboot**: Still needs AGESA blob, complex build system
- **Minimal bootloader**: <5K lines, fast boot, memory-safe Rust, minimal attack surface

**Our scope is deliberately narrow**:
- CPU mode transitions (standard x86)
- Page tables (standard x86)
- UART console (simple driver)
- Load next stage (ELF parsing)
- That's it. Everything else deferred to U-Boot/Linux.

### 4.2 Why U-Boot Instead of Direct Linux?

**Options**:
1. Bootloader → Linux directly
2. Bootloader → U-Boot → Linux

**We chose option 2 because**:
- U-Boot handles device tree, boot arguments
- U-Boot has network boot, A/B redundancy
- U-Boot is well-tested on embedded platforms
- Smaller scope for our bootloader

### 4.3 What Exactly Does Our Bootloader Do?

**Complete list**:
1. Start in 16-bit real mode (where PSP left us)
2. Set up GDT (Global Descriptor Table)
3. Switch to 32-bit protected mode
4. Enable PAE (Physical Address Extension)
5. Build initial page tables
6. Switch to 64-bit long mode
7. Initialize UART for console output
8. Decompress CPIO archive (contains U-Boot)
9. Parse U-Boot ELF, load segments
10. Set bit 11 on kernel page table entries
11. Jump to U-Boot entry point

**What it does NOT do**:
- Memory initialization (PSP/AGESA did this)
- Device enumeration (U-Boot/Linux does this)
- Power management (U-Boot/Linux does this)
- Any hardware drivers besides UART

---

## 5. Glossary for New Team Members

| Term | Meaning |
|------|---------|
| **ABL** | AGESA Boot Loader - runs on PSP |
| **AGESA** | AMD Generic Encapsulated Software Architecture |
| **APCB** | AMD PSP Customization Block - board config input |
| **APOB** | AGESA PSP Output Block - training results output |
| **BHD** | BIOS Header Directory - x86 firmware components |
| **BSP** | Boot Strap Processor - first CPU core to run |
| **CPUID** | CPU Identification instruction |
| **EFS** | Embedded Firmware Structure - flash layout |
| **FCH** | Fusion Controller Hub - integrated I/O |
| **GDT** | Global Descriptor Table - x86 segmentation |
| **IOD** | I/O Die - separate chip in EPYC MCM |
| **IPL** | Initial Program Loader - first PSP stage |
| **MCM** | Multi-Chip Module - chiplet design |
| **MMIO** | Memory-Mapped I/O |
| **MSR** | Model-Specific Register |
| **PAE** | Physical Address Extension |
| **PML4** | Page Map Level 4 - top of page tables |
| **PSP** | Platform Security Processor - ARM core |
| **PTE** | Page Table Entry |
| **ROMSIG** | ROM Signature - flash header magic |
| **SMU** | System Management Unit - power/thermal |
| **SoC** | System on Chip |
| **UMC** | Unified Memory Controller |

---

## 6. Recommended Document Structure

For a team without AMD experience, suggest reading order:

1. **This document first** (CRITICAL_REVIEW.md) - understand the caveats
2. **PLATFORM_COMPARISON.md** - understand EPYC vs V3000
3. **ARCHITECTURE_DIAGRAMS.md** - visual understanding
4. **NDA_INDEPENDENT_WORK_PLAN.md** - what we can do now
5. **COMMUNITY_INSIGHTS.md** - ecosystem context
6. **PHBL_RESEARCH_REPORT.md** - deep dive (when needed)

---

## 7. Key Risks to Communicate

### Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| V3000 addresses differ from EPYC | Medium | High | PSPTool analysis, FAE support |
| APCB structure incompatible | Medium | High | Get template from AMD |
| Boot regression during development | High | Medium | Test after every change |
| Memory training issues | Low | High | AGESA handles, but need correct APCB |

### Schedule Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| NDA/documentation delays | Medium | High | Proceed with analysis and generic code |
| Hardware availability | Low | High | QEMU development first |
| Unexpected platform differences | Medium | Medium | Build flexibility into code |

---

## 8. Action Items from This Review

### Immediate

1. **Add caveats** to all documents where assumptions are made
2. **Create glossary** section in README.md
3. **Revise estimates** to be more conservative
4. **Add "Why" explanations** for key design decisions

### Before Hardware Bringup

1. **Verify CPUID** on actual V3000 hardware
2. **Dump existing firmware** with PSPTool
3. **Get APCB template** from AMD FAE
4. **Confirm UART/GPIO addresses** from documentation

### During Development

1. **Document all assumptions** with verification status
2. **Test after every change** (boot regression is common)
3. **Keep detailed logs** of what works/fails

---

## Summary

This critical review identifies that:

1. **Several key assumptions need verification** before we can have high confidence
2. **Previous estimates were too optimistic** - revised to more realistic numbers
3. **Foundational explanations are needed** for team members new to AMD
4. **Reading order matters** - start with caveats, then architecture, then details

The project is still highly feasible, but we should:
- Present estimates as ranges, not exact numbers
- Clearly label assumptions vs confirmed facts
- Provide context for why things are the way they are
- Test assumptions as early as possible
