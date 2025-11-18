# PHBL Quick Reference Guide for V3000 Porting

## Repository
- **URL**: https://github.com/oxidecomputer/phbl
- **Language**: Rust 92.6%, Assembly 6.2%, Linker Script 1.2%
- **License**: Mozilla Public License 2.0
- **Key Design Doc**: RFD 284 (https://rfd.shared.oxide.computer/rfd/0284)

## Critical File Paths (with line numbers for key sections)

### Assembly & Early Boot
- **File**: `src/start.S` (265 lines)
  - Reset vector: lines 67-80
  - 16-bit real mode: lines 89-174
  - 32-bit protected mode: lines 125-173
  - 64-bit long mode: lines 197-228
  - Early page tables: lines 239-265

### Rust Entry Points & Initialization
- **File**: `src/main.rs` (87 lines) - Main bootloader logic
  - `entry()`: lines 24-35 - Kernel loading sequence
  - `expand_ramdisk()`: lines 41-64 - ZLIB decompression
  - `find_kernel()`: lines 66-73 - Kernel path: "platform/oxide/kernel/amd64/unix"

- **File**: `src/phbl.rs` (236 lines) - System initialization
  - `init()`: lines 81-106 - Bootstrap initialization
  - `Config` struct: lines 56-60 - System configuration
  - Memory addresses:
    - CPIO region: lines 123-126 (128 MiB)
    - MMIO start: line 180 (0x8000_0000)
    - MMIO end: line 186 (0x1_0000_0000)

### Page Table Management (Type-Driven Design)
- **File**: `src/mmu.rs` (1275 lines) - Core paging logic
  - Frame types: lines 126-190 (PFN4K, PFN2M, PFN1G)
  - Page types: lines 192-221 (Page4K, Page2M, Page1G)
  - PTE structure: lines 357-388 (bit 11 = kernel nucleus)
  - PML4-PML1 table definitions: lines 562-804
  - Page mapping algorithm: lines 837-903
  - Page table allocator: lines 1207-1274 (RFD 215 compliance)

### Hardware Abstraction
- **File**: `src/uart.rs` (564 lines) - UART driver
  - Base address: line 252 (0xFEDC_9000)
  - Device enumeration: lines 415-420
  - System clock: line 277 (48 MHz)
  - Register structures: lines 257-410
  - Initialization: lines 434-443
  - Baud rate: line B3M (3 Mbps)

- **File**: `src/iomux.rs` (72 lines) - GPIO/IO mux setup
  - GPIO base: line 26 (0xFED8_0000)
  - IO mux offset: line 27 (+0x0D00)
  - Processor detection: lines 42-71
  - UART pin config: lines 51-56 (pins 135-138)

- **File**: `src/idt.rs` (328 lines) - Exception handling
  - Gate descriptor: lines 23-40
  - IDT table: lines 223-243
  - Exception stubs: lines 107-154
  - Trap handler: lines 261-276

### Memory & Addresses
- **File**: `src/mem.rs` (211 lines) - Address types
  - V4KA (virtual): lines 11-68
  - P4KA (physical): lines 70-93
  - Attrs structure: lines 95-180
  - Canonical range: lines 17-47

### ELF Kernel Loading
- **File**: `src/loader.rs` (172 lines)
  - Thunk type definition: lines 20-27 (calling convention)
  - ELF parsing: lines 29-56
  - Header validation: lines 61-91
  - Segment loading: lines 115-163
  - Permission mapping: lines 153-157

### Allocator
- **File**: `src/allocator.rs` (133 lines)
  - Bump allocator: lines 26-81
  - Global allocator: lines 124-132
  - Page table arena: arena module (128 pages)

### Build Configuration
- **File**: `Cargo.toml` - Project manifest
  - Dependencies: bit_field, bitstruct, cpio_reader, goblin, miniz_oxide, seq-macro, static_assertions, x86
  - Edition: 2024

- **File**: `x86_64-oxide-none-elf.json` - Custom target
  - Soft float disabled
  - Redzone disabled
  - All SIMD/FPU instructions disabled

- **File**: `src/phbl.ld` - Linker script (70 lines)
  - Bootblock: 0x000000007ffef000 (line 6)
  - Reset vector: 0x000000007ffefff0 (line 7)
  - Section layout: lines 9-61

- **File**: `xtask/src/main.rs` (215 lines) - Build system
  - Build command: lines 118-152
  - CPIO environment variable: line 138 (PHBL_PHASE1_COMPRESSED_CPIO_ARCHIVE_PATH)

## Key Hardware Constants

### UART Configuration
```
Base Address: 0xFEDC_9000
UART0: 0xFEDC_9000
UART1: 0xFEDC_A000
UART2: 0xFEDC_E000
UART3: 0xFEDC_F000
System Clock: 48 MHz
Baud Rate: 3 Mbps (3,000,000 baud)
```

### IO Mux
```
GPIO Base: 0xFED8_0000
IO Mux: GPIO_BASE + 0x0D00 = 0xFED8_0D00
UART0 Pins:
  - Pin 135: CTS (Clear-To-Send)
  - Pin 136: RXD (Receive Data)
  - Pin 137: RTS (Request-To-Send)
  - Pin 138: TXD (Transmit Data)
```

### Memory Layout
```
Bootblock: 0x000000007FFEF000 (virtually, in ROM physically)
Reset Vector: 0x000000007FFEFFF0
MMIO Start: 0x8000_0000 (2 GiB)
MMIO End: 0x1_0000_0000 (4 GiB)
CPIO Archive: 128 MiB (128 * 1024 * 1024 bytes)
Stack: 32 KiB (8 * 4096 bytes)
Page Table Arena: 512 KiB (128 * 4096 bytes)
```

## CPU Register State at Kernel Entry

| Register | Value | Notes |
|----------|-------|-------|
| RDI | CPIO physical address | Ramdisk start |
| RSI | CPIO size | Ramdisk length |
| RDX, RCX, R8, R9 | 0 | Reserved |
| RSP | Stack pointer | 4KiB-aligned stack |
| RBP | 0 | Terminated frame |
| CR0 | PG, WP, PE set | Paging + protection |
| CR3 | PML4 physical address | Page table root |
| CR4 | PAE set | 4-level paging |
| EFER | LME, NX set | Long mode + NX |
| RFLAGS | IF=0, DF=0 | Interrupts disabled |

## Processor Family Support

Currently supported (from iomux.rs):
- **0x17**: Naples, Rome
- **0x19**: Milan, Genoa, Bergamo, Sienna
- **0x1a**: Turin

**V3000 Action**: Add appropriate family ID and model range

## V3000 Porting Checklist

### Must Verify/Change (Platform-Specific)
- [ ] UART MMIO base address (currently 0xFEDC_9000)
- [ ] UART system clock frequency (currently 48 MHz)
- [ ] GPIO base address (currently 0xFED8_0000)
- [ ] IO mux offset (currently +0x0D00)
- [ ] UART pin numbers (currently 135-138)
- [ ] Processor family ID and model ranges
- [ ] Bootblock/reset vector addresses
- [ ] MMIO region boundaries
- [ ] CPIO archive size allocation (128 MiB default)

### Can Reuse As-Is (Generic)
- [x] 16→32→64-bit transition code (start.S)
- [x] GDT/IDT setup (universal x86-64)
- [x] Page table algorithm (4-level paging generic)
- [x] MTRR configuration (standard x86-64)
- [x] ELF parsing and loading (standard format)
- [x] ZLIB decompression (standard algorithm)
- [x] Exception handling framework (standard x86-64)
- [x] Type-driven memory safety patterns (Rust language)

## Build Process

```bash
# Prerequisites
cargo install cargo-xtask
export PHBL_PHASE1_COMPRESSED_CPIO_ARCHIVE_PATH=/path/to/phase1.cpio.z

# Build commands
cargo xtask build --cpioz=/path/to/phase1.cpio.z [--release]
cargo xtask test [--debug|--release]
cargo xtask disasm --cpioz=/path/to/phase1.cpio.z [--source]
cargo xtask clippy
cargo xtask expand

# Output
target/x86_64-oxide-none-elf/debug/phbl     (debug build)
target/x86_64-oxide-none-elf/release/phbl   (release build)
```

## Key Design Patterns for Porting

### 1. Type-Driven Memory Safety
```rust
struct V4KA(usize);     // Only 4KiB-aligned virtual addresses
struct P4KA(u64);       // Only 4KiB-aligned physical addresses
trait Page { type FrameType: Frame; }  // Ensures page/frame size matching
```

### 2. Hardware Abstraction
```rust
impl Device {
    fn init(...) -> bool { ... }  // Standardized init sequence
}
fn cpuinfo() -> Option<(u8,u8,u8,Option<u32>)>  // CPU detection
```

### 3. Allocator for Contiguity (RFD 215)
```rust
static PAGE_ALLOCATOR: BumpAlloc<512_KiB>  // Static arena
// Guarantees contiguous page table memory
```

### 4. Attributes-to-PTEs Mapping
```rust
struct Attrs { r, w, x, c, k }  // ELF permissions
// Directly maps to PTE bits (p, w, nx, nc, k)
```

## Testing & Validation

1. **Unit Tests**: `cargo xtask test`
   - Page table tests (mmu.rs)
   - Allocator tests (allocator.rs)
   - Address type tests (mem.rs)

2. **Disassembly Inspection**: `cargo xtask disasm --source`
   - Verify 16→32→64 transitions
   - Check MSR operations

3. **Integration Testing**:
   - Deploy to real hardware or QEMU
   - Verify bootloader output via UART
   - Confirm kernel entry with correct register state

## References

- **RFD 284**: Full PHBL design specification
- **RFD 215**: Page table memory contract details
- **x86-64 ABI**: System V AMD64 calling convention
- **AMD EPYC Manuals**: Processor-specific details
- **Linker Script Reference**: GNU ld documentation

## Contact & Documentation

- **Repository**: https://github.com/oxidecomputer/phbl
- **Build Tool**: cargo-xtask (https://github.com/matklad/cargo-xtask)
- **ELF Parser**: goblin (https://github.com/m4b/goblin)
- **Decompression**: miniz_oxide (ZLIB-compatible)

---

**Report Generated**: 2024-11-18
**Total Documentation**: 1538 lines in PHBL_RESEARCH_REPORT.md
**Quick Reference**: This file
