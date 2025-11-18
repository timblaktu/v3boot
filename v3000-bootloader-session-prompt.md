# AMD V3000 Minimal Rust Bootloader Development Session

## Project Overview
Develop a minimal, memory-safe Rust bootloader for AMD Ryzen Embedded V3000 (specifically V3C18I) following Oxide Computer's proven phbl architecture. Target: <1 second boot, <5000 lines of code, 99% attack surface reduction vs UEFI.

## Core Architecture
- **Platform**: AMD V3C18I (Zen 3, 6nm, -40°C to 105°C industrial variant)
- **Boot Chain**: PSP → First-stage loader → **Our Rust bootloader** → U-Boot + Linux
- **Scope**: CPU mode transitions (16→32→64-bit), page tables, UART console, kernel/U-Boot loading
- **Non-Scope**: Hardware init (deferred to U-Boot/OS), device drivers, ACPI/device tree generation

## Key Repositories & Forks

### Primary Implementation
- **phbl** - github.com/timblaktu/phbl
  - Oxide's minimal bootloader (reference implementation)
  - Port core architecture to V3000 platform
  
- **amd-host-image-builder** - github.com/timblaktu/amd-host-image-builder  
  - Creates AMD SPI flash images with PSP firmware
  - Adapt for V3000 EFS configuration

### Supporting Infrastructure  
- **helios** - github.com/timblaktu/helios
  - Oxide's OS for boot protocol reference
  - Study handoff interface patterns

- **hubris** - github.com/timblaktu/hubris
  - Embedded microkernel patterns
  - Reference for minimal initialization

- **v3000-bootloader** - github.com/timblaktu/v3000-bootloader (NEW)
  - Our implementation repository
  - Will contain V3000-specific adaptations

## Critical Requirements

### AMD Dependencies (BLOCKING)
1. **Secure NDA access** for V3000 BKDG, PPR, PSP specs
2. **Obtain firmware binaries**: PSP firmware, AGESA, APCB templates
3. **Clarify first-stage boundary**: What state guaranteed at handoff?
4. **V3000 memory map**: Reserved regions, APOB structure

### Technical Constraints
- **No heap allocation** - static only
- **No SIMD/FP** - integer ops only  
- **Single-threaded** - BSC only, APs in reset
- **Panic = abort** - no unwinding
- **Embedded kernel** - CPIO archive compiled in

## Development Phases

### Phase 1: Foundation (Months 1-3)
1. AMD engagement & NDA execution
2. Hardware acquisition (SolidRun HoneyComb)
3. QEMU prototyping of mode transitions
4. Study Oxide RFDs #24, #26, #57

### Phase 2: Core Implementation (Months 4-6)  
1. Port phbl to V3000 addresses/registers
2. Integrate with amd-host-image-builder
3. Implement page table optimization
4. UART console on real hardware

### Phase 3: Integration (Months 7-9)
1. APCB generation with AMD FAE
2. Complete boot chain PSP→loader→U-Boot
3. Performance optimization (<1 sec)
4. A/B redundancy implementation

### Phase 4: Production (Months 10-12)
1. Security audit & fuzzing
2. Documentation & maintainability
3. CI/CD pipeline
4. Field deployment procedures

## Key Technical Decisions

### Rust Configuration
```toml
target = "x86_64-unknown-none"
panic = "abort"  
relocation-model = "static"
features = "-mmx,-sse,-sse2,-avx,-avx2"
```

### Memory Layout
- Bootloader: Identity-mapped <4GB
- Page tables: Physical contiguous, identity-mapped
- Kernel/U-Boot: High virtual addresses (>0xFFFFFF8000000000)
- Stack: 32KB, identity-mapped

### Build System
- cargo xtask pattern
- Reproducible builds
- Embedded CPIO via include_bytes!()
- GNU ld with custom linker script

## Risk Mitigations

| Risk | Mitigation |
|------|------------|
| AMD cooperation delay | Start reverse-engineering vendor BIOS |
| First-stage boundary unclear | Implement flexible handoff interface |
| Boot time >1 sec | Profile early, consider LZ4 compression |
| Security vulnerabilities | Rust safety, minimal scope, external audit |

## Success Metrics
- Boot time: <1 second to U-Boot entry
- Code size: <5000 lines Rust (excluding dependencies)
- Security: Zero unsafe blocks in critical paths
- Coverage: 100% boot success on V3C18I hardware
- Maintenance: <1 day to add new board variant

## Reference Materials
- Oxide RFDs: 24 (boot), 26 (server init), 57 (host boot CPIO)
- AMD V3000 Product Brief (public)
- phbl source: ~3000 lines production code
- SolidRun HoneyComb documentation

## Session Commands
```bash
# Clone all repositories
gh repo fork oxidecomputer/phbl --clone --remote
gh repo fork oxidecomputer/amd-host-image-builder --clone --remote
gh repo fork oxidecomputer/helios --clone --remote
gh repo fork oxidecomputer/hubris --clone --remote

# Create new implementation repo
gh repo create v3000-bootloader --public --clone

# Set up development environment
rustup target add x86_64-unknown-none
cargo install cargo-xtask
```

## First Actions
1. Review phbl source (~2000 lines) for architecture understanding
2. Map V3000 hardware differences from EPYC Milan
3. Create minimal "Hello World" bootloader for QEMU
4. Contact AMD for Developer Program enrollment
5. Order V3000 development hardware

Remember: **Extreme minimalism is the strategy**. If it's not absolutely required for loading U-Boot, it doesn't belong in the bootloader.
