# Oxide Computer Platform Extension Analysis

## Executive Summary

Oxide Computer's projects (`amd-host-image-builder` and `phbl`) are **intentionally designed for extensibility to different platforms**, but with a strongly EPYC-centric architecture. Both projects:

1. Use **enumeration-based platform selection** rather than compile-time conditionals
2. Employ **configuration files (JSON5/TOML) for platform parameters**
3. Are **designed as libraries with clear extension points**
4. Use **MPL-2.0 license** permitting modifications for new platforms

---

## 1. amd-host-image-builder Extension Pattern

### 1.1 Processor Generation Enumeration

**File**: `/tmp/amd-host-image-builder/ahib-config/src/lib.rs` (imports from `amd-efs`)

The core extension mechanism uses an enum:

```rust
pub enum ProcessorGeneration {
    Naples,      // EPYC 7001 series
    Rome,        // EPYC 7002 series  
    Milan,       // EPYC 7003 series
    Genoa,       // EPYC 9004 series
    Turin,       // EPYC 9005 series
}
```

**Location**: Defined in external `amd-efs` crate (Oxide's library)

**Extension Pattern**: To add V3000 support, you would:
1. Add `V3000` variant to `ProcessorGeneration` enum
2. Update all match statements that handle processor generations

### 1.2 Platform-Specific Configuration Structure

The tool uses **JSON5 configuration files** with processor generation as the top-level differentiator:

**File**: `/tmp/amd-host-image-builder/ahib-config/src/lib.rs` (lines 410-453)

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct RawSerdeConfig<'a> {
    pub processor_generation: ProcessorGeneration,
    #[serde(default)]
    pub spi_mode_bulldozer: Option<EfhBulldozerSpiMode>,
    #[serde(default)]
    pub spi_mode_zen_naples: Option<EfhNaplesSpiMode>,
    #[serde(default)]
    pub spi_mode_zen_rome: Option<EfhRomeSpiMode>,
    pub espi0_configuration: Option<EfhEspiConfiguration>,
    pub espi1_configuration: Option<EfhEspiConfiguration>,
    pub psp_main_directory_flash_location: Option<Location>,
    pub bhd_main_directory_flash_location: Option<Location>,
    pub psp: SerdePspDirectoryVariant,
    pub bhd: SerdeBhdDirectoryVariant<'a>,
}
```

**Key Design**: 
- Multiple SPI mode fields for different generations
- Processor-generation-specific ESPI configuration support
- Directory locations parameterized per platform

### 1.3 Platform-Specific Flash Layout Constants

**File**: `/tmp/amd-host-image-builder/src/static_config.rs` (lines 67-75)

```rust
#[allow(non_snake_case)]
pub(crate) const fn EFH_BEGINNING(
    processor_generation: ProcessorGeneration,
) -> Location {
    match processor_generation {
        ProcessorGeneration::Naples => 0x2_0000,
        ProcessorGeneration::Rome | ProcessorGeneration::Milan => 0xFA_0000,
        ProcessorGeneration::Genoa | ProcessorGeneration::Turin => 0x2_0000,
    }
}
```

**Evidence of Extensibility**:
- Function is `const fn` allowing compile-time optimization
- Pattern matching on processor generation determines flash addresses
- EFH (Embedded Firmware Header) location varies: `0x2_0000` vs `0xFA_0000`
- Clear grouping (Naples alone, Rome+Milan together, Genoa+Turin together)

### 1.4 Configuration Validation by Processor Generation

**File**: `/tmp/amd-host-image-builder/ahib-config/src/lib.rs` (lines 491-567)

The config system **validates SPI mode compatibility with processor generation**:

```rust
impl<'a> core::convert::TryFrom<RawSerdeConfig<'a>> for SerdeConfig<'a> {
    type Error = Error;
    fn try_from(raw: RawSerdeConfig<'a>) 
        -> core::result::Result<Self, Self::Error> {
        match raw.processor_generation {
            ProcessorGeneration::Naples => {
                if raw.spi_mode_bulldozer.is_none()
                    && raw.spi_mode_zen_naples.is_some()
                    && raw.spi_mode_zen_rome.is_none() {
                    return Ok(SerdeConfig { /* ... */ });
                }
            }
            ProcessorGeneration::Rome | ProcessorGeneration::Milan => {
                if raw.spi_mode_bulldozer.is_none()
                    && raw.spi_mode_zen_naples.is_none()
                    && raw.spi_mode_zen_rome.is_some() {
                    return Ok(SerdeConfig { /* ... */ });
                }
            }
            ProcessorGeneration::Genoa | ProcessorGeneration::Turin => {
                // Genoa/Turin use ESPI or Bulldozer, not Zen Rome modes
                if raw.spi_mode_zen_naples.is_none()
                    && raw.spi_mode_zen_rome.is_none()
                    && (raw.espi0_configuration.is_some()
                        || raw.espi1_configuration.is_some()) {
                    return Ok(SerdeConfig { /* ... */ });
                }
            }
        }
        Err(Error::Efs(amd_efs::Error::SpiModeMismatch))
    }
}
```

**Key Pattern**: Each processor generation has specific required/forbidden configuration combinations.

### 1.5 Application Configuration Files

**File**: `/tmp/amd-host-image-builder/apps/milan-gimlet-b-1.0.0.a.toml`

```toml
cpu = 'milan'
board = 'gimlet-b'
firmware_version = '1.0.0.a'
size = 32
blobs = [
    'AmdPubKey_gn.tkn',
    'PspBootLoader_gn.sbin',
    'PspRecoveryBootLoader_gn.sbin',
    'SmuFirmwareGn.csbin',
    # ... 30+ firmware blobs ...
]
```

**Extension Pattern**:
- Create new `.toml` file for V3000: `v3000-ruby-1.0.0.a.toml`
- Update `cpu`, `board`, `firmware_version` fields
- Specify V3000-specific firmware blobs

### 1.6 Where Platform Constants Are Defined

**Key Files for Platform Extension**:

| Component | File | How Extended |
|-----------|------|--------------|
| Processor Generation | External `amd-efs` crate | Add enum variant |
| Flash Layout | `src/static_config.rs` | Add match arm to `EFH_BEGINNING()` |
| SPI Mode | `ahib-config/src/lib.rs` | Add `Option<EfhV3000SpiMode>` field |
| Firmware Blobs | `apps/*.toml` files | Create new `.toml` with V3000 blobs |
| Validation Logic | `ahib-config/src/lib.rs` TryFrom impl | Add match arm for V3000 processor gen |

---

## 2. phbl Platform Abstraction

### 2.1 UART Address Parameterization

**File**: `/tmp/phbl/src/uart.rs` (lines 251-252)

```rust
/// The base virtual address of all UARTs.
const UART_MMIO_BASE_ADDR: usize = 0xFEDC_9000;
```

**Critical Design**:
- Single constant for Milan/Rome/Genoa/Turin
- **Hardcoded EPYC assumption**: This address is specific to EPYC FCH (Fusion Controller Hub)
- For V3000 (different FCH design), this would need to change

**Uart Device Enum**:
```rust
#[repr(usize)]
pub enum Device {
    Uart0 = UART_MMIO_BASE_ADDR,
    _Uart1 = UART_MMIO_BASE_ADDR + 0x1000,
    _Uart2 = UART_MMIO_BASE_ADDR + 0x5000,
    _Uart3 = UART_MMIO_BASE_ADDR + 0x6000,
}
```

**Problem**: If V3000 has different UART spacing, this needs refactoring.

### 2.2 GPIO and I/O Mux Configuration

**File**: `/tmp/phbl/src/iomux.rs` (lines 23-60)

```rust
pub unsafe fn init() {
    if let Some(settings) = mux_settings() {
        use core::ptr;
        const GPIO_BASE_ADDR: usize = 0xFED8_0000;  // Hardcoded
        const IOMUX_BASE_ADDR: usize = GPIO_BASE_ADDR + 0x0D00;
        let iomux = ptr::with_exposed_provenance_mut::<u8>(IOMUX_BASE_ADDR);
        for (pin, function) in settings.iter() {
            unsafe {
                ptr::write_volatile(iomux.offset(*pin), *function as u8);
            }
        }
    }
}
```

**Platform-Specific Pin Configuration** via CPUID:

```rust
fn mux_settings() -> Option<&'static [(isize, GpioX)]> {
    const SP5: u32 = 4;
    
    match cpuinfo()? {
        (0x17, 0x00..=0x0f, 0x0..=0xf, _) | // Naples
        (0x17, 0x30..=0x3f, 0x0..=0xf, _) | // Rome
        (0x19, 0x00..=0x0f, 0x0..=0xf, _) | // Milan
        (0x19, 0x10..=0x1f, 0x0..=0xf, Some(SP5)) | // Genoa
        (0x19, 0xa0..=0xaf, 0x0..=0xf, Some(SP5)) | // Bergamo
        (0x1a, 0x00..=0x1f, 0x0..=0xf, Some(SP5)) => { // Turin
            Some(&[
                (135, GpioX::F0),  // UART0 CTS
                (136, GpioX::F0),  // UART0 RXD
                (137, GpioX::F0),  // UART0 RTS
                (138, GpioX::F0),  // UART0 TXD
            ])
        },
        _ => None,
    }
}
```

**CPUID Format**: `(family, model_range, stepping_range, socket_type)`

**For V3000 Support**:
- Family: `0x18` or TBD (from AMD NDA docs)
- Model range: TBD
- Add new match arm with V3000-specific GPIO pins

### 2.3 Memory Layout Constants

**File**: `/tmp/phbl/src/phbl.rs` (lines 123-126)

```rust
fn cpio_addr() -> mem::V4KA {
    const CPIO_LEN: usize = 128 * mem::MIB;  // Fixed 128 MiB
    mem::V4KA::new(saddr().addr() - CPIO_LEN)
}
```

**Issue**: CPIO archive size is fixed. If V3000 needs different memory layout, this impacts page table setup.

### 2.4 CPU Detection and Mode Transition

**File**: `/tmp/phbl/src/iomux.rs` (lines 64-71)

```rust
fn cpuinfo() -> Option<(u8, u8, u8, Option<u32>)> {
    let cpuid = x86::cpuid::CpuId::new();
    let features = cpuid.get_feature_info()?;
    let family = features.family_id();
    let ext = cpuid.get_extended_processor_and_feature_identifiers()?;
    let pkg_type = (family > 0x10).then_some(ext.pkg_type());
    Some((family, features.model_id(), features.stepping_id(), pkg_type))
}
```

**Abstraction Level**: Runtime CPU detection allows same binary to support multiple platforms.

---

## 3. Build System Integration (xtask)

### 3.1 phbl Build System

**File**: `/tmp/phbl/xtask/src/main.rs`

```rust
#[derive(Parser)]
enum Command {
    Build {
        #[clap(flatten)]
        profile: BuildProfile,
        #[clap(long)]
        cpioz: PathBuf,  // Compressed CPIO archive path
    },
    Clippy,
    Disasm,
    Test,
}
```

**Pattern**: Single binary output, parameterized at runtime by:
- CPIO archive (embedded at compile time)
- Target specification file

### 3.2 Custom Target Specification

**File**: `/tmp/phbl/x86_64-oxide-none-elf.json`

```json
{
    "llvm-target": "x86_64-unknown-none",
    "vendor": "oxide",
    "os": "none",
    "linker": "gld",
    "pre-link-args": {
        "ld": ["-nostdlib", "-Tsrc/phbl.ld", "-zmax-page-size=4096"]
    }
}
```

**For V3000 Extension**:
- Could create `x86_64-v3000-none-elf.json` if different linker flags needed
- Or reuse with same architecture (both x86_64)

---

## 4. Intended Usage Patterns

### 4.1 Library vs. Fork Design

**Evidence from amd-host-image-builder**:

1. **Designed for Forking**: README states user should fork and customize
2. **Not a Drop-in Library**: Requires deep understanding of PSP/EFH structure
3. **Monolithic Approach**: Single tool handles all processor generations

**Evidence from phbl**:

1. **Could be Used as Library**: Modular code organization
2. **Currently Monolithic**: No feature flags or conditional compilation
3. **Platform Detection at Runtime**: Supports runtime CPU type detection

### 4.2 License Implications (MPL-2.0)

**Key Points**:
- Can modify for internal use (V3000 version)
- Must preserve Mozilla Public License headers
- Can distribute modified versions with modifications disclosed
- **No requirement to upstream changes** (but encouraged)

**Recommendation for V3000 Extension**:
```
1. Fork both repositories (or create internal branches)
2. Add ProcessorGeneration::V3000 to amd-efs
3. Update phbl iomux.rs with V3000 CPUID detection
4. Create v3000-specific configuration files
5. Add validation logic for V3000 in amd-host-image-builder config
6. Consider upstreaming if V3000 support is generally useful
```

---

## 5. Documentation and Extension Points

### 5.1 Files Needing Modification for V3000

| File | Change | Type |
|------|--------|------|
| `amd-efs/src/lib.rs` | Add `V3000` to `ProcessorGeneration` enum | Required |
| `amd-host-image-builder/src/static_config.rs` | Add V3000 case to `EFH_BEGINNING()` | Required |
| `amd-host-image-builder/ahib-config/src/lib.rs` | Add V3000 SPI mode validation | Required |
| `phbl/src/iomux.rs` | Add V3000 CPUID detection (family 0x18?) | Required |
| `phbl/src/uart.rs` | **CRITICAL**: May need V3000-specific UART base addr | Conditional |
| `phbl/src/mem.rs` | May need V3000-specific memory layout | Conditional |
| `apps/v3000-*.toml` | Create V3000 configuration files | Required |
| `.cargo/config.toml` | May need V3000 target specification | Optional |

### 5.2 Parameters That Vary by Platform

| Parameter | Naples | Rome | Milan | Genoa | Turin | V3000 |
|-----------|--------|------|-------|-------|-------|-------|
| EFH Address | 0x2_0000 | 0xFA_0000 | 0xFA_0000 | 0x2_0000 | 0x2_0000 | ? |
| UART Base | 0xFEDC_9000 | Same | Same | Same | Same | ? |
| GPIO Base | 0xFED8_0000 | Same | Same | Same | Same | ? |
| CPU Family | 0x17 | 0x17 | 0x19 | 0x19 | 0x1a | ? |
| SPI Mode | Bulldozer | Zen Rome | Zen Rome | ESPI | ESPI | ? |

---

## 6. Key Assumptions and Unknowns

### 6.1 EPYC-Specific Assumptions

1. **FCH (Fusion Controller Hub) Design**: UART/GPIO addresses assume EPYC FCH architecture
2. **PSP Boot Sequence**: Assumes AMD PSP bootloader (may be different on V3000)
3. **Memory Initialization**: Assumes PSP initializes RAM before handoff
4. **Flash Organization**: Assumes SPI flash with PSP directory structure
5. **Processor Package Types**: Socket detection via `pkg_type` field (SP5 socket)

### 6.2 V3000-Specific Information Required

From AMD NDA documentation:
- [ ] CPU Family/Model/Stepping ID
- [ ] UART base address (FCH UART MMIO)
- [ ] GPIO base address (FCH GPIO MMIO)
- [ ] I/O mux pin assignments for UART
- [ ] EFH (Embedded Firmware Header) location in flash
- [ ] SPI configuration mode (Bulldozer, Zen Rome, ESPI, or custom)
- [ ] Memory map (PSP reserved regions, APOB location)
- [ ] First-stage loader interface specification
- [ ] Any silicon errata workarounds

---

## 7. Recommendations for V3000 Extension

### Phase 1: Preparation
1. **Document V3000 Constants**: Gather all platform-specific values from datasheets/NDA docs
2. **Analyze PSPTool Output**: Extract firmware blobs from existing V3000 system using PSPTool
3. **Study Rembrandt in Coreboot**: V3000 is based on Rembrandt; coreboot implementation provides hints

### Phase 2: amd-host-image-builder Extension
```bash
# 1. Create new enum variant (needs amd-efs fork/modification)
enum ProcessorGeneration {
    V3000,  // Add this
}

# 2. Add static config
pub(crate) const fn EFH_BEGINNING(pg: ProcessorGeneration) -> Location {
    match pg {
        // ... existing cases ...
        ProcessorGeneration::V3000 => 0x???_?????,  // TBD
    }
}

# 3. Add SPI mode
pub struct EfhV3000SpiMode { /* TBD */ }

# 4. Add validation
ProcessorGeneration::V3000 => {
    // Check which SPI modes are valid for V3000
}

# 5. Create config file
# apps/v3000-ruby-1.0.0.a.toml
cpu = 'v3000'
board = 'v3c18i'
firmware_version = '1.0.0.a'
blobs = [ /* V3000-specific blobs */ ]
```

### Phase 3: phbl Extension
```rust
// 1. Add V3000 to iomux detection
fn mux_settings() -> Option<&'static [(isize, GpioX)]> {
    match cpuinfo()? {
        // ... existing cases ...
        (0x18, 0x??, .., Some(SP5)) => { // V3000 family?
            Some(&[
                (???, GpioX::F0),  // UART0 pins TBD
                // ...
            ])
        }
        _ => None,
    }
}

// 2. If UART address differs, parameterize it:
// Option A: Runtime detection
const UART_MMIO_BASE_ADDR: usize = get_uart_base_addr();
fn get_uart_base_addr() -> usize {
    match cpuinfo() {
        Some((family, ..)) if family == 0x18 => 0x????_?????,  // V3000
        _ => 0xFEDC_9000,  // EPYC default
    }
}

// Option B: Compile-time feature
#[cfg(feature = "platform-v3000")]
const UART_MMIO_BASE_ADDR: usize = 0x????_????;
#[cfg(not(feature = "platform-v3000"))]
const UART_MMIO_BASE_ADDR: usize = 0xFEDC_9000;
```

### Phase 4: Build and Test
```bash
# Build with V3000 support
cargo xtask build --cpioz=$CPIOZ

# Test UART output
cargo xtask test

# Create flash image
cargo run -p amd-host-image-builder -- generate \
    --config apps/v3000-ruby-1.0.0.a.toml \
    --payload target/x86_64-oxide-none-elf/release/phbl \
    --output v3000-flash.bin
```

---

## 8. Concrete Extension Example: V3000 Configuration File

```json5
// apps/v3000-ruby-1.0.0.a.toml
{
    cpu = 'v3000'
    board = 'v3c18i'
    firmware_version = '1.0.0.a'
    size = 16  // V3000 typically has 16 MiB flash
    
    blobs = [
        // V3000-specific firmware blobs from AMD package
        'V3000_AmdPubKey.tkn',
        'V3000_PspBootLoader.sbin',
        'V3000_PSP_Firmware.sbin',
        'V3000_SMU_Firmware.csbin',
        'V3000_AGESA_binary.csbin',
        // ... more blobs ...
    ]
}
```

```json5
// etc/v3000-ruby-1.0.0.a.efs.json5
{
    processor_generation: 'V3000',
    spi_mode_v3000: {
        // TBD based on V3000 PSP requirements
        read_mode: 'Normal33_33MHz',
        fast_speed: 'FastSpeed',
        // ... other V3000-specific SPI settings ...
    },
    psp_main_directory_location: undefined,  // Auto-allocated
    bhd_main_directory_location: undefined,  // Auto-allocated
    
    psp: {
        entries: [
            // PSP firmware entries for V3000
            {
                source: { BlobFile: 'V3000_PspBootLoader.sbin' },
                target: {
                    type: 'PspBootLoader',
                    sub_program: 0,
                }
            },
            // ... more PSP entries ...
        ]
    },
    
    bhd: {
        entries: [
            {
                source: { BlobFile: 'V3000_AGESA_binary.csbin' },
                target: {
                    type: 'AgesaBootloader',
                    copy_image: true,
                    ram_destination_address: 0x1234_5678,
                }
            },
            // ... more BHD entries ...
        ]
    }
}
```

---

## Conclusion

Oxide Computer has designed **amd-host-image-builder** and **phbl** with clear, enumeration-based extension patterns:

### Strengths:
1. **Extensible Architecture**: ProcessorGeneration enum makes it easy to add new platforms
2. **Configuration-Driven**: JSON5/TOML configs separate platform details from code
3. **Runtime Detection**: CPUID-based platform detection allows single binary for multiple targets
4. **Clear Constants**: Platform-specific values isolated in `static_config.rs` and iomux module

### Weaknesses (V3000-Specific):
1. **EPYC-Specific Assumptions**: Memory addresses, pin mappings assume EPYC architecture
2. **Hardcoded Values**: UART base, GPIO base not parameterized
3. **No Feature Flags**: Cannot conditionally compile V3000-specific code
4. **NDA-Dependent**: Requires AMD documentation for constants

### Recommended Approach:
**Fork and Modify Pattern** - These projects expect to be forked for new platforms, with modifications tracked in separate branches or repositories. The modular design supports this well.

