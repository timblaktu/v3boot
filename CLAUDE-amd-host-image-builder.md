# CLAUDE.md - amd-host-image-builder for V3000

## Repository Purpose
Fork of Oxide's AMD boot image construction tool, adapted to generate complete SPI flash images for AMD V3000 platforms with integrated bootloader, PSP firmware, and AGESA.

## Critical Context
- **Parent**: https://github.com/oxidecomputer/amd-host-image-builder
- **Function**: Assembles PSP directory, integrates AGESA, builds final flash image
- **Input**: Bootloader binary, PSP/AGESA blobs, APCB configuration
- **Output**: Complete SPI flash image ready for programming

## Priority Task List

### P0: Understanding (BLOCKING)
- [ ] Study Oxide's EFS (Embedded Firmware Structure) generation
- [ ] Document PSP directory format differences V3000 vs EPYC
- [ ] Map V3000 flash layout requirements (BHD structure)
- [ ] Identify required firmware blobs from AMD package

### P1: V3000 Platform Definition
- [ ] Create V3000 family identifier configuration
- [ ] Define V3000-specific directory entries
- [ ] Add V3C18I SKU-specific parameters
- [ ] Document flash size constraints (16MB typical)

### P2: Configuration System
- [ ] Port APCB (AMD PSP Customization Block) for V3000
- [ ] Add DDR5 training parameters for embedded platform
- [ ] Configure PCIe lanes (20x Gen4 for V3000)
- [ ] Set power management for industrial temperature

### P3: Image Construction
- [ ] Integrate phbl bootloader binary at correct offset
- [ ] Add PSP firmware blobs from AMD package
- [ ] Include AGESA v3000PI binary
- [ ] Generate proper checksums and signatures

### P4: Flash Layout
- [ ] Define partition scheme for A/B redundancy
- [ ] Reserve space for configuration storage
- [ ] Add recovery image support
- [ ] Implement boot counter region

### P5: Validation Tools
- [ ] Add image verification command
- [ ] Create flash layout visualizer
- [ ] Implement binary comparison tool
- [ ] Add APCB parameter validator

## Key Files to Modify

```
src/
├── main.rs           # Add V3000 command support
├── config/
│   ├── v3000.rs     # NEW: V3000-specific configuration
│   └── apcb_v3000.rs # NEW: APCB generation for V3000
├── psp/
│   ├── directory.rs  # Update for V3000 directory format
│   └── firmware.rs   # Add V3000 firmware blob handling
├── flash/
│   ├── layout.rs     # V3000 flash layout definition
│   └── builder.rs    # Image assembly logic
└── validate/
    └── v3000.rs      # NEW: V3000-specific validation
```

## Configuration Format

```toml
# v3000-image.toml
[platform]
family = "v3000"
sku = "v3c18i"
flash_size = "16MB"

[psp]
firmware = "path/to/PSP_V3000.bin"  # From AMD package
directory_offset = 0x020000

[agesa]
binary = "path/to/AGESA_V3000PI.bin"  # From AMD package
version = "1.0.0.0"

[apcb]
template = "configs/v3000_apcb_template.bin"
ddr5_speed = 4800
channels = 2

[bootloader]
binary = "../phbl/target/x86_64-v3000-none-elf/release/phbl"
load_address = 0xFFFF0000  # To be confirmed

[layout]
redundancy = "a_b"
recovery = true
```

## Build Commands

```bash
# Generate V3000 flash image
cargo run -- build \
    --config v3000-image.toml \
    --output v3000-flash.bin

# Validate image structure
cargo run -- validate v3000-flash.bin

# Extract components for analysis
cargo run -- extract \
    --input v3000-flash.bin \
    --output-dir extracted/
```

## AMD Firmware Requirements (NDA)
- **PSP Firmware**: V3000-specific binary from AMD
- **AGESA**: v3000PI package
- **APCB Template**: Board-specific from AMD FAE
- **Microcode**: Latest V3000 CPU microcode patches
- **SMU Firmware**: System Management Unit binary

## Flash Programming

```bash
# Using flashrom on Linux
flashrom -p linux_spi:dev=/dev/spidev0.0 \
    -w v3000-flash.bin

# Using DediProg SF100
dpcmd -u v3000-flash.bin -v
```

## Critical Unknowns (Need AMD)
- Exact PSP directory format for V3000
- Required firmware blob versions
- Signature/authentication requirements
- Boot-time security processor validation
- Recovery mode trigger mechanism

## Testing Strategy
1. **Binary validation**: Check structure before flashing
2. **QEMU testing**: If V3000 QEMU model available
3. **Incremental flashing**: Test components separately
4. **Recovery testing**: Verify fallback mechanisms
5. **Power cycle testing**: Ensure cold boot reliability

## Success Criteria
- Generates valid V3000 flash images
- PSP accepts and boots firmware
- Supports A/B redundancy
- Recovery mode functions correctly
- Image size <16MB

## References
- Oxide RFD-15: AMD Boot ROM Configuration
- PSPTool: https://github.com/PSPReverse/PSPTool
- AMD PSP Architecture: [REQUIRES NDA]
- AGESA Integration Guide: [REQUIRES NDA]
