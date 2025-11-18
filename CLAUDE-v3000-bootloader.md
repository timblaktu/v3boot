# CLAUDE.md - v3000-bootloader Implementation Repository

## Repository Purpose
New repository for AMD V3000 minimal Rust bootloader implementation. This is the main development repository combining learnings from Oxide's phbl with V3000-specific requirements.

## Development Tasks

### Priority 1: Project Setup (Week 1)
- [ ] Initialize Rust project with no_std configuration
- [ ] Create x86_64-v3000-none target specification
- [ ] Set up cargo-xtask build system
- [ ] Configure CI/CD pipeline
- [ ] Add QEMU test harness

### Priority 2: Core Boot Sequence (Week 2-3)
- [ ] Implement 16-bit real mode entry point
- [ ] Code 32-bit protected mode transition
- [ ] Implement 64-bit long mode activation
- [ ] Create minimal GDT/IDT structures
- [ ] Test mode transitions in QEMU

### Priority 3: Memory Management (Week 4-5)
- [ ] Design page table structure
- [ ] Implement 4-level paging (PML4)
- [ ] Add large page optimization (2MB/1GB)
- [ ] Create identity mapping for bootloader
- [ ] Map high addresses for kernel

### Priority 4: UART Console (Week 6)
- [ ] Research V3000 UART configuration
- [ ] Implement 16550 UART driver
- [ ] Add V3000 IO mux setup
- [ ] Create panic handler with UART output
- [ ] Test on hardware

### Priority 5: Kernel Loading (Week 7-8)
- [ ] Design CPIO archive format
- [ ] Integrate miniz_oxide decompression
- [ ] Implement ELF parser using goblin
- [ ] Load U-Boot binary
- [ ] Execute handoff protocol

### Priority 6: Hardware Integration (Week 9-10)
- [ ] Integrate with amd-host-image-builder
- [ ] Create V3000 board configuration
- [ ] Flash and test on SolidRun HoneyComb
- [ ] Measure and optimize boot time
- [ ] Implement recovery mechanism

### Priority 7: Production Hardening (Week 11-12)
- [ ] Security audit all unsafe blocks
- [ ] Add integrity checking
- [ ] Implement A/B redundancy
- [ ] Complete documentation
- [ ] Performance optimization

## Project Structure
```
v3000-bootloader/
├── src/
│   ├── main.rs          # Entry point
│   ├── asm/
│   │   └── boot.S       # 16/32-bit assembly
│   ├── cpu/
│   │   ├── gdt.rs       # Global Descriptor Table
│   │   ├── idt.rs       # Interrupt Descriptor Table
│   │   └── modes.rs     # CPU mode transitions
│   ├── memory/
│   │   ├── paging.rs    # Page table management
│   │   └── layout.rs    # Memory map
│   ├── drivers/
│   │   └── uart.rs      # Serial console
│   ├── loader/
│   │   ├── cpio.rs      # Archive handling
│   │   └── elf.rs       # ELF loading
│   └── platform/
│       └── v3000.rs     # Platform-specific code
├── xtask/
│   └── src/main.rs      # Build automation
├── tests/
│   └── qemu/            # Integration tests
└── link.ld              # Linker script
```

## Configuration
```toml
[package]
name = "v3000-bootloader"
version = "0.1.0"
edition = "2021"

[dependencies]
x86_64 = { version = "0.15", default-features = false }
goblin = { version = "0.8", default-features = false, features = ["elf64"] }
miniz_oxide = { version = "0.7", default-features = false }
bit_field = "0.10"

[profile.release]
panic = "abort"
lto = true
opt-level = "z"
```

## Key Design Decisions

### Memory Map
- 0x00000000-0x00100000: Real mode area
- 0x00100000-0x00200000: Bootloader code/data
- 0x00200000-0x00400000: Page tables
- 0x00400000-0x01000000: Temporary decompression buffer
- 0xFFFFFF8000000000+: Kernel virtual address

### Boot Protocol
1. PSP loads bootloader at 0x100000
2. CPU starts in 16-bit real mode
3. Bootloader transitions to 64-bit
4. Loads U-Boot from embedded CPIO
5. Calls U-Boot entry with initrd pointer

### Security Model
- No dynamic memory allocation
- All data statically sized
- Panic = immediate halt
- Minimal unsafe blocks
- No external dependencies at runtime

## Testing Strategy
- Unit tests for pure functions
- QEMU for boot sequence validation
- Hardware tests on V3C18I board
- Fuzzing for parser code
- Boot time benchmarking

## Success Metrics
- Boot time: <500ms to U-Boot
- Code size: <5000 lines Rust
- Binary size: <2MB including kernel
- Test coverage: >80%
- Zero CVEs in production

## AMD Dependencies (BLOCKING)
- [ ] PSP firmware binaries
- [ ] AGESA integration package  
- [ ] APCB configuration tool
- [ ] V3000 BKDG documentation
- [ ] Platform Programming Reference

## Hardware Requirements
- SolidRun HoneyComb V3C18I board
- SPI flash programmer
- JTAG debugger
- USB-UART adapter
- Oscilloscope for timing
