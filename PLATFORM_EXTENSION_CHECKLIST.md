# Oxide Computer Platform Extension Checklist

## Repository Structure

### amd-host-image-builder
- **Source Root**: `/tmp/amd-host-image-builder/`
- **License**: `/tmp/amd-host-image-builder/LICENSE.txt` (MPL-2.0)
- **README**: `/tmp/amd-host-image-builder/README.md`

### phbl
- **Source Root**: `/tmp/phbl/`
- **License**: `/tmp/phbl/LICENSE.txt` (MPL-2.0)
- **README**: `/tmp/phbl/README.md`

---

## Critical Files for Platform Extension

### amd-host-image-builder

| File | Purpose | Lines | Status |
|------|---------|-------|--------|
| `/tmp/amd-host-image-builder/src/static_config.rs` | Platform flash layout constants | 67-75 | **KEY FILE** |
| `/tmp/amd-host-image-builder/src/main.rs` | Configuration validation, entry point | 491-567 | **KEY FILE** |
| `/tmp/amd-host-image-builder/ahib-config/src/lib.rs` | Configuration structures (RawSerdeConfig) | 410-453 | **KEY FILE** |
| `/tmp/amd-host-image-builder/apps/milan-gimlet-b-1.0.0.a.toml` | Example configuration | Full file | Reference |
| `/tmp/amd-host-image-builder/.cargo/config.toml` | Cargo build configuration | Full file | Check for customization |

### phbl

| File | Purpose | Lines | Status |
|------|---------|-------|--------|
| `/tmp/phbl/src/iomux.rs` | GPIO/pin configuration, CPUID detection | 23-71 | **KEY FILE** |
| `/tmp/phbl/src/uart.rs` | UART driver, MMIO addresses | 251-252 (constants), 415-420 (Device enum) | **KEY FILE** |
| `/tmp/phbl/src/phbl.rs` | Platform initialization, memory layout | 123-126 (CPIO size), 81-106 (init function) | Review |
| `/tmp/phbl/src/mem.rs` | Memory address types and validation | 1-30 (constants) | Review |
| `/tmp/phbl/x86_64-oxide-none-elf.json` | Target specification | Full file | May need V3000 variant |
| `/tmp/phbl/xtask/src/main.rs` | Build system configuration | 1-100+ | Review for build patterns |
| `/tmp/phbl/Cargo.toml` | Dependencies | Full file | Check for platform-specific deps |

---

## Extension Checklist

### Phase 1: Information Gathering

- [ ] Obtain V3000 BKDG (BIOS and Kernel Developers Guide) from AMD NDA
- [ ] Document CPU Family/Model/Stepping ID from `CPUID` output
- [ ] Identify UART base address for V3000 FCH UART
- [ ] Identify GPIO base address for V3000 FCH GPIO
- [ ] Document pin assignments for UART signals
- [ ] Determine EFH (Embedded Firmware Header) location in flash
- [ ] Identify SPI flash configuration mode (Bulldozer, Zen Rome, ESPI, custom)
- [ ] Map PSP reserved memory regions
- [ ] Analyze existing V3000 firmware with PSPTool

### Phase 2: amd-efs Library Update

Dependencies: These files are in external `amd-efs` crate (needs fork)

- [ ] Add `ProcessorGeneration::V3000` enum variant
- [ ] Define `EfhV3000SpiMode` struct if needed
- [ ] Add V3000 validation logic
- [ ] Update all match statements covering processor generations

### Phase 3: amd-host-image-builder Configuration

- [ ] Create `/tmp/amd-host-image-builder/apps/v3000-ruby-1.0.0.a.toml`
  - Set `processor_generation: 'V3000'`
  - List V3000-specific firmware blobs
  - Set flash size (typically 16 MiB)

- [ ] Create `/tmp/amd-host-image-builder/etc/v3000-ruby-1.0.0.a.efs.json5`
  - Configure PSP directory entries
  - Configure BHD directory entries
  - Set V3000-specific SPI mode

- [ ] Update `/tmp/amd-host-image-builder/src/static_config.rs`
  - Add V3000 case to `EFH_BEGINNING()` function
  - Add V3000 case to `EFH_SIZE` if different

- [ ] Update `/tmp/amd-host-image-builder/ahib-config/src/lib.rs` TryFrom impl
  - Add V3000 processor generation validation
  - Verify SPI mode combinations

### Phase 4: phbl UART/GPIO Configuration

- [ ] Update `/tmp/phbl/src/iomux.rs`
  - Add V3000 CPUID pattern matching
  - Define GPIO pin assignments for V3000
  - Add V3000 case to `mux_settings()` function

- [ ] Review `/tmp/phbl/src/uart.rs`
  - Check if UART base address differs for V3000
  - If different, consider parameterization options:
    - Option A: Runtime detection via CPUID
    - Option B: Compile-time feature flag
    - Option C: Device parameter in Device enum

- [ ] Update `/tmp/phbl/src/mem.rs` if needed
  - Adjust memory layout constants for V3000
  - Verify canonical address space assumptions

- [ ] Update `/tmp/phbl/src/phbl.rs` if needed
  - Adjust CPIO archive size if V3000 has different memory
  - Adjust ramdisk region initialization

### Phase 5: Build System

- [ ] Test build with phbl:
  ```bash
  cd /tmp/phbl
  cargo xtask build --cpioz=<compressed_cpio_path>
  ```

- [ ] Test amd-host-image-builder generation:
  ```bash
  cd /tmp/amd-host-image-builder
  cargo xtask gen \
    --payload /path/to/phbl \
    --amd-firmware /path/to/amd-firmware/blobs \
    --app apps/v3000-ruby-1.0.0.a.toml \
    --image out/v3000-ruby.img
  ```

### Phase 6: Validation

- [ ] Validate flash image structure
- [ ] Extract and inspect PSP directory
- [ ] Extract and inspect BHD directory
- [ ] Verify firmware blob sizes match configuration
- [ ] Check checksum calculations

### Phase 7: Hardware Testing

- [ ] Flash image to V3000 test system
- [ ] Verify UART output at boot
- [ ] Check PSP boot messages
- [ ] Verify kernel loading
- [ ] Test boot parameters handoff

---

## Key Code Patterns to Understand

### Pattern 1: Processor Generation Match Statement

**Location**: Multiple files

```rust
match processor_generation {
    ProcessorGeneration::Naples => { /* Naples-specific logic */ }
    ProcessorGeneration::Rome | ProcessorGeneration::Milan => { /* Rome/Milan */ }
    ProcessorGeneration::Genoa | ProcessorGeneration::Turin => { /* Genoa/Turin */ }
    // ADD V3000 HERE
}
```

### Pattern 2: CPUID-Based Detection

**Location**: `/tmp/phbl/src/iomux.rs`

```rust
fn cpuinfo() -> Option<(u8, u8, u8, Option<u32>)> {
    // Returns (family, model, stepping, socket_type)
}

fn mux_settings() -> Option<&'static [(isize, GpioX)]> {
    match cpuinfo()? {
        (0x17, 0x00..=0x0f, _, _) => { /* Naples */ }
        // ADD V3000 CPUID PATTERN HERE
        _ => None,
    }
}
```

### Pattern 3: Configuration Validation

**Location**: `/tmp/amd-host-image-builder/ahib-config/src/lib.rs`

```rust
ProcessorGeneration::V3000 => {
    // Specify which SPI modes/configurations are valid for V3000
    if /* valid V3000 config */ {
        return Ok(SerdeConfig { /* ... */ });
    }
}
```

---

## Dependencies and External Crates

### Required Forks/Modifications

1. **amd-efs** - External crate
   - Add `ProcessorGeneration::V3000`
   - Likely hosted on: GitHub oxidecomputer/amd-efs

2. **amd-apcb** - External crate
   - May need V3000 APCB structure updates
   - Likely hosted on: GitHub oxidecomputer/amd-apcb

### Independent Modifications

1. **amd-host-image-builder** - Can fork independently
   - No changes required to dependencies if amd-efs updated
   - Add static config and validation

2. **phbl** - Can fork independently
   - Only depends on standard crates (x86, miniz_oxide, etc.)
   - No external Oxide crates needed

---

## File Modification Impact Analysis

### Low Risk (Configuration Only)
- Create new `.toml` files for V3000 in `apps/`
- Create new `.json5` files for V3000 in `etc/`

### Medium Risk (Adding Match Arms)
- Update `/tmp/amd-host-image-builder/src/static_config.rs`
- Update `/tmp/phbl/src/iomux.rs`
- Update amd-efs ProcessorGeneration enum

### High Risk (Architecture Change)
- Parameterizing UART base address in phbl
- Changing memory layout constants
- Modifying CPU detection logic

---

## Testing Strategy

### Unit Tests
- [ ] Test new CPUID detection patterns
- [ ] Test configuration validation
- [ ] Test flash layout calculations

### Integration Tests
- [ ] Build complete flash image
- [ ] Verify image structure with PSPTool
- [ ] Extract and compare blobs

### Hardware Tests
- [ ] Boot on V3000 hardware
- [ ] Capture UART output
- [ ] Verify handoff to kernel

---

## Documentation to Create

1. **V3000 Platform Specification**
   - CPUID information
   - Memory map
   - Hardware addresses
   - Boot sequence

2. **Extension Guide**
   - How to add new platforms to these tools
   - Common pitfalls
   - Testing checklist

3. **Configuration Reference**
   - V3000-specific TOML/JSON5 options
   - PSP directory entry types for V3000
   - BHD directory entry types for V3000

