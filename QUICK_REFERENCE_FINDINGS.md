# Platform Extension Quick Reference

## Key Findings

### Design Intent: YES, Designed for Extension

Oxide's projects explicitly support adding new processor generations through:
- **Enumeration-based architecture** (ProcessorGeneration enum)
- **Configuration-driven parameters** (JSON5/TOML files)
- **Runtime platform detection** (CPUID-based dispatch)
- **Clear extension points** (match statements on processor type)

### License: MPL-2.0 Permissive

- Can modify for internal use without restrictions
- Can distribute modified versions (must disclose changes)
- No requirement to upstream
- Must preserve license headers

### Current Coverage: EPYC Only

Supports: Naples, Rome, Milan, Genoa, Turin (all EPYC families)
Missing: V3000 (Ryzen Embedded)

---

## Critical Code Locations

### amd-host-image-builder

**Flash Layout by Processor** (`/tmp/amd-host-image-builder/src/static_config.rs:67-75`):
```rust
pub(crate) const fn EFH_BEGINNING(pg: ProcessorGeneration) -> Location {
    match pg {
        ProcessorGeneration::Naples => 0x2_0000,
        ProcessorGeneration::Rome | ProcessorGeneration::Milan => 0xFA_0000,
        ProcessorGeneration::Genoa | ProcessorGeneration::Turin => 0x2_0000,
    }
}
```
ACTION: Add V3000 case here

**Configuration Validation** (`/tmp/amd-host-image-builder/ahib-config/src/lib.rs:491-567`):
```rust
impl<'a> TryFrom<RawSerdeConfig<'a>> for SerdeConfig<'a> {
    fn try_from(raw: RawSerdeConfig<'a>) -> Result<Self> {
        match raw.processor_generation {
            ProcessorGeneration::Naples => { /* validation */ }
            ProcessorGeneration::Rome | ProcessorGeneration::Milan => { /* validation */ }
            ProcessorGeneration::Genoa | ProcessorGeneration::Turin => { /* validation */ }
        }
    }
}
```
ACTION: Add V3000 validation case

### phbl

**GPIO Pin Configuration** (`/tmp/phbl/src/iomux.rs:39-59`):
```rust
fn mux_settings() -> Option<&'static [(isize, GpioX)]> {
    match cpuinfo()? {
        (0x17, 0x00..=0x0f, _, _) => Some(&[(135, GpioX::F0), ...])  // Naples
        (0x19, 0x00..=0x0f, _, _) => Some(&[(135, GpioX::F0), ...])  // Milan
        (0x1a, 0x00..=0x1f, _, Some(SP5)) => Some(&[...])  // Turin
        _ => None,
    }
}
```
ACTION: Add V3000 CPUID match arm with V3000-specific pins

**UART Address** (`/tmp/phbl/src/uart.rs:251-252`):
```rust
const UART_MMIO_BASE_ADDR: usize = 0xFEDC_9000;
```
RISK: If V3000 UART address differs, needs refactoring for parameterization

---

## Minimum Changes Required

### For Functional V3000 Support

1. **Update amd-efs library** (external dependency):
   - Add `ProcessorGeneration::V3000` enum variant
   - Define `EfhV3000SpiMode` struct (if needed)

2. **Update amd-host-image-builder** (~20 lines):
   - `/tmp/amd-host-image-builder/src/static_config.rs`: Add 2-3 lines
   - `/tmp/amd-host-image-builder/ahib-config/src/lib.rs`: Add 5-10 lines
   - Create `/tmp/amd-host-image-builder/apps/v3000-ruby-1.0.0.a.toml`
   - Create `/tmp/amd-host-image-builder/etc/v3000-ruby-1.0.0.a.efs.json5`

3. **Update phbl** (~10 lines):
   - `/tmp/phbl/src/iomux.rs`: Add CPUID match arm (5-10 lines)
   - `/tmp/phbl/src/uart.rs`: Add UART base if address differs (0-10 lines)

### Total Code Changes: ~40-60 lines (if UART address is same)

---

## Parameters Needed from AMD (NDA Required)

### Critical for Boot to Work

1. **CPUID Values**
   - Family ID: `0x??`
   - Model range: `0x?? - 0x??`
   - Stepping range: `0x?? - 0x??`
   - Socket/package type: (if applicable)

2. **Memory Addresses**
   - UART MMIO base: `0x???? ????` (or confirm 0xFEDC_9000)
   - GPIO/IOMUX base: `0x???? ????` (or confirm 0xFED8_0000)
   - EFH flash location: `0x???? ????`

3. **Pin Assignments for UART**
   - Pin numbers for RXD, TXD, CTS, RTS

### Important but Workaroundable

1. **SPI Mode Capabilities**
   - Supported modes (Bulldozer, Zen Rome, ESPI, custom)
   - Flash timing parameters

2. **Memory Layout**
   - PSP reserved regions
   - APOB location
   - CPIO/ramdisk region size (if different from 128 MiB)

---

## Extension Pattern Recommendation

### The "Fork and Customize" Approach

These projects are designed for forking, not drop-in library use:

```
Original Repos (Oxide Maintained)
  ├─ amd-efs (library, core abstractions)
  ├─ amd-host-image-builder (configuration tool)
  └─ phbl (bootloader)
         ↓ Fork for new platform
Company Internal Repos (V3000 Variants)
  ├─ amd-efs@v3000 (add ProcessorGeneration::V3000)
  ├─ amd-host-image-builder@v3000 (add static config)
  └─ phbl@v3000 (add GPIO detection)
```

### Update Strategy

1. **Minor Version Patch Approach**
   - Fork both repositories
   - Add V3000 support in separate branch
   - Tag as `v0.1.3-v3000` or similar
   - Keep in sync with upstream main branch

2. **Upstreaming Consideration**
   - If V3000 support is broadly useful, propose PR to Oxide
   - ProcessorGeneration enum is already extensible
   - Minimal disruption to existing code

---

## Build System Integration

### phbl Build Command

```bash
cd /tmp/phbl
cargo xtask build --cpioz=./phase1.cpio.z --release
# Output: target/x86_64-oxide-none-elf/release/phbl
```

### amd-host-image-builder Command

```bash
cd /tmp/amd-host-image-builder
cargo xtask gen \
    --payload ./target/x86_64-oxide-none-elf/release/phbl \
    --amd-firmware ./amd-firmware/blobs/V3000 \
    --app apps/v3000-ruby-1.0.0.a.toml \
    --image out/v3000-flash.bin
```

### Schema Validation

```bash
cd /tmp/amd-host-image-builder
cargo xtask schema  # Generates JSON schema for validation
```

---

## Risk Assessment

| Area | Risk | Mitigation |
|------|------|-----------|
| UART Address Mismatch | Medium | CPUID-based runtime detection |
| GPIO Pin Assignments | Low | Add new match arm in iomux.rs |
| Flash Layout | Low | Add to static_config.rs |
| SPI Mode Compatibility | Medium | Update validation logic |
| Memory Layout | Low | Can use same 128 MiB CPIO region |
| Firmware Blob Compatibility | High | Requires AMD firmware binaries |
| PSP Boot Sequence | High | Requires AMD PSP documentation |

---

## Testing Checklist

### Pre-Hardware

- [ ] Configuration files parse correctly
- [ ] Flash layout calculations are correct
- [ ] CPUID detection works (via testing on V3000 CPU)
- [ ] PSPTool can parse generated image
- [ ] Firmware blobs extract correctly

### Hardware Bring-up

- [ ] UART output appears at correct baud rate (115200)
- [ ] PSP boot messages visible
- [ ] Bootloader message appears
- [ ] Kernel loads successfully
- [ ] Boot handoff parameters correct

---

## Expected Time Estimates

| Phase | Time | Notes |
|-------|------|-------|
| Information Gathering | 1-2 weeks | Awaiting AMD NDA docs |
| Code Implementation | 1-2 days | ~50-60 lines of code |
| Configuration Creation | 1 day | TOML/JSON5 files |
| Build & Unit Testing | 1 day | Verify compilation |
| Hardware Testing | 1-2 weeks | Depends on hardware availability |
| **Total** | **3-5 weeks** | Most time is waiting/hardware access |

---

## Common Pitfalls

1. **Assuming UART address is universal**
   - Different AMD processors sometimes have different FCH designs
   - Must verify for V3000 specifically

2. **Hardcoding processor-specific values**
   - Use runtime detection (CPUID) where possible
   - Makes code more portable

3. **Forgetting to validate all match statements**
   - Pattern matching is exhaustive
   - Adding enum variant requires updating ALL match arms

4. **Incorrect PSP directory offsets**
   - PSP directory must be 4-KiB aligned
   - Must account for erasable block sizes

5. **Missing firmware blobs**
   - PSP needs specific blob types in specific order
   - Wrong blob or missing blob = boot failure

---

## Absolute File Paths Summary

### amd-host-image-builder Key Files
```
/tmp/amd-host-image-builder/src/static_config.rs          [Flash layout]
/tmp/amd-host-image-builder/src/main.rs                    [Config validation]
/tmp/amd-host-image-builder/ahib-config/src/lib.rs         [Config structures]
/tmp/amd-host-image-builder/apps/                          [Platform configs]
/tmp/amd-host-image-builder/etc/                           [Platform definitions]
/tmp/amd-host-image-builder/Cargo.toml                     [Dependencies]
```

### phbl Key Files
```
/tmp/phbl/src/iomux.rs                                     [GPIO detection]
/tmp/phbl/src/uart.rs                                      [UART driver]
/tmp/phbl/src/phbl.rs                                      [Initialization]
/tmp/phbl/src/mem.rs                                       [Memory constants]
/tmp/phbl/x86_64-oxide-none-elf.json                       [Target spec]
/tmp/phbl/xtask/src/main.rs                                [Build system]
```

### amd-efs Library (External)
```
amd-efs/src/lib.rs                                         [ProcessorGeneration enum]
```

---

## Success Criteria

V3000 support is successful when:

1. [ ] Compilation succeeds without warnings
2. [ ] `cargo xtask gen` produces valid flash image
3. [ ] PSPTool can parse generated image
4. [ ] UART outputs bootloader message at 115200 baud
5. [ ] Bootloader successfully loads kernel
6. [ ] Kernel receives correct boot parameters

