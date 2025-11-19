# Community Research and Ecosystem Insights

This document synthesizes research from GitHub forks, issues, PRs, and the broader AMD boot ecosystem to provide actionable insights for the V3000 bootloader project.

---

## Executive Summary

Research reveals a more active AMD open-source firmware ecosystem than initially expected:

- **Oxide repositories** have 8 forks of phbl, 3 of amd-apcb, primarily by firmware researchers
- **Coreboot has Rembrandt support** (V3000's base architecture) - direct reference available
- **PSPReverse tools** (PSPTool, PSPEmu) enable complete firmware analysis without NDA
- **AMD openSIL** will replace AGESA by 2026-2027, though V3000 (Zen 3) won't benefit
- **3mdeb's V1000/R1000 work** provides closest precedent for embedded AMD bootloader

---

## 1. Key Technical Insights from GitHub Issues

### Processor Generation Detection

**Issue amd-efs#120** reveals critical detection values:

| Generation | Detection Bits | Notes |
|------------|---------------|-------|
| Turin | `0xe3` | Latest EPYC |
| Milan | `0xfc` | Older Family 19h |
| Genoa | `0xfe` | Newer Family 19h |
| **V3000** | **TBD** | Need to determine |

**AMD Document Reference**: 57299 Table 4

**Implication**: V3000 will need its own generation identifier bits.

### UART/GPIO Configuration

**Issue phbl#48** (SP5 UART0 pin mux):
- Pin mux values on Genoa differ from documented reset values
- Pins default to GPIO instead of UART
- **Must explicitly set IO mux values**

**Critical Insight for V3000**: Cannot rely on reset defaults. Must configure:
```rust
// Example from phbl iomux.rs
(135, GpioX::F0),  // UART0_CTS
(136, GpioX::F0),  // UART0_RXD
(137, GpioX::F0),  // UART0_RTS
(138, GpioX::F0),  // UART0_TXD
```

### APCB Structure Differences

**Issue amd-apcb#157** (DimmInfoSmbusElement):
- AMD reuses same Type ID with different structure layouts
- Milan vs Genoa/Turin have different fields
- New fields include Socket/Channel/DIMM encoding

**Issue amd-apcb#161** (Token value conflicts):
- Milan: Custom 3 MBaud = value 9
- Turin 1.0.0.5: 3 MBaud = value 14
- Previous value 9 = 230400 Baud in newer versions

**Implication**: V3000 may have unique APCB structures and token values. Cannot assume EPYC values work.

### Boot Regression Risk

**Issue phbl#33**: Boot failure after single commit
- Fixed by commit `9e73c23b`

**Best Practice**: Test boot after every change. Even small modifications can cause failures.

---

## 2. Platform Extension Patterns

### Adding New Processor Support

**Turin Support Pattern** (from multiple PRs):

1. **Add processor detection** (amd-efs#118)
2. **Create APCB tokens** (amd-apcb#134, #140)
3. **Add image builder config** (amd-host-image-builder#253)
4. **Iterate on firmware versions** (#223, #225, #247, #251)

**Cosmo Platform Pattern** (amd-host-image-builder#227):

1. Create JSON5 configuration file
2. Test during bringup
3. Refine with hardware
4. Update for future firmware

**V3000 Application**:
1. Add CPUID detection pattern `(0x19, 0x40..=0x4f, _, _)`
2. Create v3000 board configuration TOML
3. Test with SolidRun HoneyComb
4. Iterate as firmware details emerge

### Stable Rust Migration

**Issue amd-apcb#41**: Required nightly Rust due to:
- `#[pre]` attribute
- Generic Associated Types (GAT)

**Resolution** (amd-host-image-builder#231, amd-efs#123, amd-apcb#158):
- All three repos now support stable Rust
- Updated to Rust 2024 edition
- Uses `xtask` build system

**Implication**: V3000 project can use stable Rust toolchain.

---

## 3. Notable Community Members

### Fork Owners

| Owner | Repo | Background |
|-------|------|------------|
| **orangecms** | amd-apcb | Daniel Maslowski, coreboot developer |
| **skade** | phbl | Florian Gilcher, Ferrous Systems (Rust safety-critical) |
| **fiedka** | amd-apcb | Active contributor, 27 commits |
| **daym** | Multiple | Primary maintainer, issues and PRs |
| **citrus-it** | amd-host-image-builder | External PR contributor |

### External Resources

| Person/Org | Contribution | Contact |
|------------|--------------|---------|
| **3mdeb** | V1000/R1000 AGESA+EDK2 | blog.3mdeb.com |
| **TU Berlin** | PSPTool, PSPEmu | Robert Buhren, Alexander Eichner |
| **DayZeroSec** | PSP reverse engineering | dayzerosec.com |
| **Paul Grimes** | AMD openSIL lead | AMD |

---

## 4. Tools and Resources

### Essential Tools

| Tool | Purpose | Usage |
|------|---------|-------|
| **PSPTool** | Firmware analysis | `psptool -E firmware.bin` |
| **PSPEmu** | PSP emulation | Run PSP code without hardware |
| **AMD-SP-Loader** | Binary Ninja plugin | Analyze extracted blobs |
| **amdpsp-re** | PSP notes/tools | Reference material |

### Recommended Workflow

```bash
# 1. Dump existing V3000 firmware
# (from SolidRun board or vendor BIOS)

# 2. Analyze with PSPTool
psptool -E v3000_firmware.bin > analysis.txt
psptool -X -u -o extracted/ v3000_firmware.bin

# 3. Study structure
# Look for:
# - PSP directory entries
# - AGESA version
# - APCB structure
# - UART/GPIO addresses

# 4. Compare with EPYC
# Check differences in:
# - Directory structure
# - Entry types
# - Configuration values
```

### Reference Implementations

| Source | Path/URL | Purpose |
|--------|----------|---------|
| Coreboot Rembrandt | `src/soc/amd/rembrandt` | V3000's base architecture |
| Coreboot Mendocino | `src/soc/amd/mendocino` | Derived from Rembrandt |
| Coreboot Sabrina | - | Rembrandt split source |
| 3mdeb V1000 | blog.3mdeb.com | AGESA integration patterns |

---

## 5. Community Best Practices

### From Oxide PRs

1. **Document RFD references** in code
2. **Use page table provenance** for valid pointer creation
3. **Require type annotations** and `///` documentation
4. **Test boot after every commit**
5. **Use `Cow` for buffers** supporting both std and no_std

### From Issues

1. **Dump existing firmware first** (amd-host-image-builder#229)
2. **Specify minimum Rust version** explicitly
3. **Handle struct size differences** carefully (amd-apcb#134)
4. **Use raw token values** to avoid version conflicts (amd-host-image-builder#243)
5. **Improve error context** (amd-efs#122)

### For External Users

1. **Read RFD 284** - Authoritative phbl technical reference
2. **Use stable Rust** - All repos now support it
3. **Check platform-specific tokens** for your AMD generation
4. **Use JSON5 configuration** for amd-host-image-builder
5. **Expect Gimlet-specific code** that needs adaptation

---

## 6. Common Gotchas

### Build Issues

| Problem | Solution |
|---------|----------|
| Missing `gld` linker | Set `CARGO_TARGET_X86_64_OXIDE_NONE_ELF_LINKER` |
| illumos builds fail | Use `LD=gld cargo xtask ...` |
| serde_yaml enum failure | Use serde_json instead |

### Runtime Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| `IncompatibleExecutable` | `_BL_SPACE` validation too strict | PR #13 (draft) |
| Boot failure | Code regression | Test after every commit |
| UART not working | Wrong pin mux | Explicitly set GPIO functions |

### Platform Issues

| Problem | Impact |
|---------|--------|
| Token values differ | Baud rates, speeds, etc. change between versions |
| APCB structures differ | Same Type ID, different layouts |
| Dump command broken | Doesn't work on Oxide images |

---

## 7. AMD Boot Ecosystem Timeline

### Current State (2024-2025)

```
Binary AGESA Required
       │
       ├── Coreboot: Chromebook platforms (FSP-compatible)
       ├── Oxide: phbl (custom integration)
       ├── 3mdeb: V1000/R1000 EDK2 (commercial)
       └── Vendors: Standard IBV approach
```

### Future State (2026-2027)

```
AMD openSIL (Open Source)
       │
       ├── EPYC Venice (Zen 6): First production
       ├── Ryzen Medusa (Zen 6): Client production H1 2027
       └── Partners: AMI, 9elements, Supermicro
```

### V3000 Strategy

Since V3000 is Zen 3 (not Zen 6), it will use the binary AGESA approach:

1. **2024-2025**: Use binary AGESA with phbl-style integration
2. **Future platforms**: Consider openSIL when available
3. **Architecture design**: Keep compatible with openSIL patterns

---

## 8. Additional Resources

### Key Presentations

| Event | Talk | Key Content |
|-------|------|-------------|
| OSFC 2022 | Bryan Cantrill "Bury the BIOS" | Holistic systems philosophy |
| Black Hat 2020 | "AMD PSP" | PSP emulator, debug bypass |
| 36C3 2019 | "Dissecting AMD PSP" | Complete architecture |
| FOSDEM 2021-2024 | "AMD OSF Status" | Annual updates |
| OCP 2025 | "openSIL" | Venice/Medusa announcement |

### Oxide RFDs

| RFD | Title | Relevance |
|-----|-------|-----------|
| 20 | Host Bootstrap Software Objectives | Design goals |
| 215 | Oxide Machine Architecture | Memory layout, bit 11 |
| 216 | Verified Boot | Security model |
| 241 | Holistic Boot | Overall architecture |
| 284 | Loading Host OS | **Primary phbl reference** |

Access: https://rfd.shared.oxide.computer/

### Hacker News Discussions

- [phbl and OpenSIL](https://news.ycombinator.com/item?id=35571572)
- [Oxide's UEFI replacement](https://news.ycombinator.com/item?id=39025316)
- [Helios discussion](https://news.ycombinator.com/item?id=39178521)

---

## 9. Contribution Opportunities

### Open Issues (Potential Help)

| Issue | Repo | Description |
|-------|------|-------------|
| #229 | amd-host-image-builder | Flash image dump script |
| #248 | amd-host-image-builder | SEC I2C Voltage for Turin |
| #106 | amd-efs | Address mode type splitting |
| #157 | amd-apcb | DimmInfoSmbusElement differences |

### Draft PRs (Stalled)

| PR | Repo | Status |
|----|------|--------|
| #222 | amd-host-image-builder | ApobNvCopy BHD entry |
| #162 | amd-host-image-builder | Improve dump subcommand |
| #13 | amd-host-image-builder | Relax ELF loader space check |

---

## 10. Recommendations

### Immediate Actions

1. **Obtain V3000 firmware** from SolidRun and analyze with PSPTool
2. **Study coreboot Rembrandt** (`src/soc/amd/rembrandt`)
3. **Read RFD 284** thoroughly
4. **Join coreboot mailing list** for Rembrandt insights

### Architecture Decisions

1. **Use Oxide phbl as base** - Proven, well-documented
2. **Use AMD FCH standard addresses** - High confidence correct
3. **Explicitly configure pin mux** - Cannot trust reset defaults
4. **Design for testing** - Boot can break with small changes

### Community Engagement

1. **Open issues** for questions (see phbl#37 positive response)
2. **Attend FOSDEM/OSFC** for AMD OSF community
3. **Contact 3mdeb** for V1000/R1000 experience
4. **Monitor openSIL** for architectural patterns

---

## Summary

The community research reveals:

1. **Rich ecosystem** of tools and knowledge around AMD boot
2. **Clear patterns** for adding new platform support
3. **Critical gotchas** to avoid (pin mux, token values, testing)
4. **Strong reference** in coreboot Rembrandt
5. **Active development** toward open-source (openSIL 2026+)

The V3000 project can leverage significant community resources while following proven patterns from Oxide's EPYC implementations.
