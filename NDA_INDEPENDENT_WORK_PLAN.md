# V3000 Bootloader: NDA-Independent Work Plan

## Executive Summary

Deep research into Oxide Computer's repositories reveals that **60-70% of the bootloader project can proceed without AMD NDA access**. The phbl architecture is 80% generic/reusable, extensive public PSP documentation exists, and tooling is mature. This document outlines actionable work that can begin immediately.

---

## Key Research Findings

### 1. phbl Architecture (80% Reusable)

**Generic Components (Use As-Is):**
- CPU mode transitions (16→32→64-bit) - complete assembly code
- Page table construction algorithm (4-level paging with 1GiB/2MiB optimization)
- ELF loading and parsing (goblin crate)
- CPIO archive handling
- Exception handling framework
- Bit 11 ownership transfer protocol (RFD 215)
- Kernel handoff protocol (System V AMD64 ABI)

**Platform-Specific (~20% needs porting):**
- UART base address (currently 0xFEDC_9000)
- GPIO base address (currently 0xFED8_0000)
- UART pins (135-138)
- Processor family detection
- Memory map boundaries

### 2. Public PSP Knowledge (Extensive)

**Fully Documented:**
- PSP directory structure (magic numbers, entry types, 30+ types)
- ROMSIG search locations and format
- Boot sequence: BootROM → IPL → ABL0-7 → SecureOS → x86
- APCB/APOB purpose and relationship
- Platform Secure Boot (PSB) architecture
- Entry point addresses and ABL execution order

**Tools Available:**
- PSPTool - complete firmware analysis
- Coreboot amdfwtool - image building
- AMD-SP-Loader - Binary Ninja analysis

### 3. amd-host-image-builder (Highly Extensible)

The tool is designed for adding new platforms. Current support:
- Milan (EPYC 7003)
- Genoa (EPYC 9004)
- Turin (EPYC 9005)

Adding V3000 requires:
- New `ProcessorGeneration::V3000` enum
- Platform configuration TOML
- Firmware blob directory structure

### 4. Helios Boot Protocol (Fully Documented)

Critical discovery: **Bit 11 Protocol**
```
Every kernel page table entry MUST have bit 11 set
PTE[11] = 1 → Kernel nucleus page (bootloader ownership transferred)
PTE[11] = 0 → Non-kernel page
```

**Handoff Specification:**
```
RDI = Physical address of decompressed CPIO archive
RSI = Archive size in bytes
CR3 = Physical address of PML4
CPU: 64-bit long mode, paging enabled, interrupts disabled
Stack: 32 KiB available, aligned
```

---

## Immediate Work Items (No NDA Required)

### Phase 1: Infrastructure Setup (Week 1-2)

#### 1.1 Project Scaffolding
```bash
# Create v3000-bootloader repository structure
v3000-bootloader/
├── src/
│   ├── main.rs           # Entry point
│   ├── asm/boot.S        # 16/32-bit assembly (port from phbl)
│   ├── cpu/              # Mode transitions
│   ├── memory/           # Page tables
│   ├── drivers/          # UART (placeholder addresses)
│   └── platform/         # V3000 constants (TBD markers)
├── xtask/                # Build automation
└── tests/qemu/           # Integration tests
```

#### 1.2 Build System Configuration
- Custom target spec: `x86_64-v3000-none-elf.json`
- Cargo xtask pattern from phbl
- Linker script with V3000 memory layout placeholders
- CI/CD pipeline (GitHub Actions)

#### 1.3 QEMU Test Harness
- Generic x86_64 boot testing
- Mode transition validation
- Page table construction verification

### Phase 2: Core Boot Sequence (Week 3-4)

#### 2.1 Port phbl Assembly
Copy and adapt from phbl:
- Reset vector entry (16-bit)
- GDT setup
- Protected mode switch
- Long mode activation
- Paging enablement

**These are GENERIC x86 operations - no V3000-specific code needed.**

#### 2.2 Rust Entry Point
```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[no_mangle]
pub extern "C" fn rust_main(cpio_paddr: u64, cpio_size: u64) -> ! {
    // Initialize UART (address TBD)
    // Build page tables
    // Load kernel from CPIO
    // Handoff
    loop {}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

#### 2.3 Page Table Implementation
Port phbl's greedy allocation algorithm:
- 4-level paging (PML4/PML3/PML2/PML1)
- 1GiB pages where aligned
- 2MiB pages otherwise
- 4KiB for remainder
- Bit 11 marking for kernel pages

**Algorithm is GENERIC - works on any x86_64.**

### Phase 3: Memory Management (Week 5-6)

#### 3.1 Memory Map Abstraction
```rust
// Platform-agnostic interface
pub trait MemoryMap {
    fn bootloader_region(&self) -> (u64, u64);
    fn page_table_region(&self) -> (u64, u64);
    fn cpio_region(&self) -> (u64, u64);
    fn kernel_virtual_base(&self) -> u64;
}

// V3000 implementation (values TBD with NDA)
pub struct V3000MemoryMap {
    // Placeholders marked with TODO
}
```

#### 3.2 Identity Mapping
- Bootloader code/data < 4GB (identity mapped)
- Use large pages for efficiency
- Clear separation from kernel addresses

#### 3.3 Kernel High Address Mapping
- Map kernel to 0xFFFFFF8000000000+
- Transfer ownership via bit 11 protocol
- Physically contiguous page table allocation

### Phase 4: UART Driver (Week 7)

#### 4.1 Generic 16550 UART Driver
```rust
pub struct Uart16550 {
    base: u64,
    clock: u32,
}

impl Uart16550 {
    pub fn init(&self, baud: u32) {
        // Standard 16550 initialization
        // Divisor calculation
        // 8N1 configuration
    }

    pub fn putc(&self, c: u8) {
        // Wait for THRE, write to THR
    }
}
```

**The 16550 driver logic is standard - only base address is platform-specific.**

#### 4.2 Console Integration
- Panic handler output
- Debug logging
- Boot progress messages

### Phase 5: Kernel/U-Boot Loading (Week 8-9)

#### 5.1 CPIO Archive Handling
Port from phbl:
- SVR4/newc format parsing
- File extraction
- Path matching (e.g., finding U-Boot ELF)

#### 5.2 ELF Loader
Using `goblin` crate:
- Parse ELF64 headers
- Load segments to correct addresses
- Relocate if needed
- Determine entry point

#### 5.3 Handoff Protocol Design
For U-Boot (different from Helios):
- Research U-Boot entry expectations
- Design boot parameter structure
- Document register conventions

### Phase 6: Analysis and Preparation (Week 10+)

#### 6.1 Analyze Existing V3000 Firmware
Using PSPTool:
```bash
# Get firmware from SolidRun board or vendor
psptool -E v3000_firmware.bin > v3000_analysis.txt
psptool -X -u -o extracted/ v3000_firmware.bin
```

This reveals:
- Actual entry types used
- Directory structure
- Firmware versions
- Memory addresses (some)

#### 6.2 Study Coreboot Rembrandt
V3000 is based on Rembrandt (Zen 3, 6nm):
```bash
# In coreboot tree
cd src/soc/amd/rembrandt
# Study boot flow, APCB handling, memory init
```

#### 6.3 Monitor openSIL Progress
- https://github.com/openSIL/openSIL
- May provide AGESA replacement for V3000 by 2026
- Track for future integration

---

## Public Information Sources

### Tools to Use Now

| Tool | Purpose | Link |
|------|---------|------|
| PSPTool | Analyze AMD firmware | https://github.com/PSPReverse/PSPTool |
| Coreboot | Rembrandt code study | https://github.com/coreboot/coreboot |
| AMD firmware_binaries | Reference blobs | https://github.com/amd/firmware_binaries |
| openSIL | Future AGESA replacement | https://github.com/openSIL/openSIL |

### Documentation Available

- V3000 Product Brief (public PDF)
- SolidRun HoneyComb docs (serial at 115200 baud)
- PSP architecture (Black Hat 2020, DayZeroSec)
- Coreboot PSP integration guide
- Platform Secure Boot blogs

### Research Papers
- Black Hat 2020: "All You Ever Wanted to Know About AMD PSP"
- IOActive: "Exploring AMD Platform Secure Boot"
- TU Berlin: PSPTool development papers

---

## What Specifically Requires NDA

### Hard Blockers

1. **V3000 UART Addresses** - FCH UART register locations
2. **Memory Map** - PSP reserved regions, APOB location
3. **Processor Family ID** - CPUID family/model for V3000
4. **First-Stage Interface** - Exact state at handoff from PSP

### Soft Blockers (Can Work Around)

1. **APCB Structure** - Can use template from FAE
2. **PSP Firmware Blobs** - Provided by AMD
3. **AGESA Binary** - Provided by AMD

---

## Concrete Next Steps

### This Week

1. **Initialize Repository**
   ```bash
   cargo new --lib v3000-bootloader
   cd v3000-bootloader
   rustup target add x86_64-unknown-none
   ```

2. **Port phbl Assembly**
   - Copy `reset.S` equivalent
   - Adapt for build system

3. **Set Up QEMU Testing**
   ```bash
   qemu-system-x86_64 -machine q35 -nographic \
     -bios target/x86_64-v3000-none-elf/release/v3000-bootloader.bin
   ```

### This Month

4. **Complete Mode Transitions**
   - Verify 16→32→64-bit in QEMU
   - Ensure paging works correctly

5. **Implement Page Tables**
   - Port phbl algorithm
   - Test with various memory layouts

6. **Create UART Driver**
   - Generic 16550 code
   - Parameterized base address

### When NDA Arrives

7. **Fill In Platform Constants**
   - UART: Replace placeholder with V3000 address
   - GPIO: Add IO mux configuration
   - Memory: Define actual layout

8. **Integrate Firmware Blobs**
   - PSP firmware from AMD
   - AGESA binary
   - APCB template

---

## Estimated Progress Without NDA

| Component | Completable Without NDA |
|-----------|------------------------|
| Build system | 100% |
| CPU mode transitions | 100% |
| Page table algorithm | 100% |
| ELF loading | 100% |
| CPIO handling | 100% |
| UART driver logic | 90% (address TBD) |
| Memory map | 50% (layout TBD) |
| Platform constants | 10% (need NDA) |
| Flash image building | 30% (need blobs) |

**Overall: 65-70% of bootloader can be implemented and tested before NDA.**

---

## Risk Mitigation

### If NDA is Delayed

1. **Analyze vendor BIOS** with PSPTool to extract addresses
2. **Study Rembrandt in coreboot** for architectural hints
3. **Use generic x86 values** for QEMU testing
4. **Document all "TBD" markers** for quick fill-in later

### If Hardware is Delayed

1. **QEMU development** covers most functionality
2. **Unit tests** for pure functions
3. **Prepare flash programming scripts** in advance

---

## Conclusion

The research reveals that the V3000 bootloader project has a **clear path forward without NDA access**. By focusing on:

- Generic x86_64 boot code (from phbl)
- Platform-agnostic algorithms (page tables, ELF loading)
- Abstracted hardware interfaces (UART driver framework)
- Public PSP knowledge (directory structure, boot flow)

We can have a **functional bootloader that boots in QEMU** within 4-6 weeks. Once the NDA arrives, filling in V3000-specific constants and integrating firmware blobs is estimated at 2-3 weeks additional work.

**Recommendation**: Begin Phase 1 immediately. The infrastructure and generic code will be ready when platform-specific details become available.

---

## References

- Oxide phbl: https://github.com/oxidecomputer/phbl
- Oxide amd-host-image-builder: https://github.com/oxidecomputer/amd-host-image-builder
- Oxide helios: https://github.com/oxidecomputer/helios
- PSPTool: https://github.com/PSPReverse/PSPTool
- Coreboot AMD: https://doc.coreboot.org/soc/amd/
- openSIL: https://github.com/openSIL/openSIL
- SolidRun HoneyComb: https://solidrun.atlassian.net/wiki/spaces/developer/
