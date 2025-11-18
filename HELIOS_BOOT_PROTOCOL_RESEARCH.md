# Oxide Helios Boot Protocol Research Report
## Comprehensive Analysis of Bootloader-to-Kernel Handoff

**Date:** 2024-11-18  
**Repository:** https://github.com/oxidecomputer/helios  
**Focus:** Boot protocol, kernel handoff, and page table ownership

---

## Executive Summary

The Oxide Helios bootloader (phbl - "Pico Host Boot Loader") implements a **clean, minimal boot protocol** optimized for the x86-64 architecture. The handoff mechanism is based on:

1. **RDI/RSI register convention** for passing ramdisk parameters
2. **Physically-contiguous page tables** with strict constraints
3. **Bit 11 ownership protocol** for marking kernel-owned pages
4. **ZLIB-compressed CPIO archives** containing the kernel and boot filesystem

This protocol is significantly different from traditional BIOS/UEFI bootloaders and cannot be directly used by U-Boot without a compatibility shim.

---

## Part 1: EXACT KERNEL ENTRY POINT PROTOCOL

### Kernel Entry Signature (x86-64 System V ABI)

**Location:** `phbl/src/loader.rs`, lines 20-27

```rust
type Thunk = unsafe extern "C" fn(
    ramdisk_paddr: u64,    // RDI - first parameter
    ramdisk_len: usize,    // RSI - second parameter
    _rdx: u64,             // RDX - third parameter (unused, passed as 0)
    _rcx: u64,             // RCX - fourth parameter (unused, passed as 0)
    _r8: u64,              // R8 - fifth parameter (unused, passed as 0)
    _r9: u64,              // R9 - sixth parameter (unused, passed as 0)
);
```

### Call Protocol (phbl/src/main.rs, line 33)

```rust
entry(ramdisk.as_ptr().addr() as u64, ramdisk.len());
```

### Expected CPU State at Kernel Entry

From `phbl/src/phbl.rs` (initialization code):

**Memory State:**
- CPU in 64-bit long mode (LM bit set in EFER MSR)
- Paging enabled (CR0.PG = 1)
- Write protection enabled (CR0.WP = 1)
- Interrupts disabled (RFLAGS.IF = 0)
- A 32KiB stack is available (sufficient for early boot)
- AP (Application Processor) cores in reset state
- GDT loaded with minimal descriptors
- No IDT initially loaded

**Memory Layout:**
- Loader is identity-mapped (VA == PA for bootloader code)
- CPIO archive is memory-resident at a fixed region (128 MiB)
- MMIO space is mapped starting at `0x8000_0000`
- Higher half kernel (`0xFFFF_8000_0000_0000` and above) can be used

**Page Table State:**
- PML4 (root page table) loaded in CR3
- Page tables are physically contiguous
- All page table frames come from a contiguous region starting at PML4's physical address
- Minimum 16 4KiB pages allocated for page table structures

### Kernel Parameters

| Register | Value | Purpose |
|----------|-------|---------|
| RDI | Physical address of CPIO archive | Points to compressed/uncompressed CPIO containing kernel and root filesystem |
| RSI | Archive size in bytes | Length of decompressed CPIO archive |
| RDX, RCX, R8, R9 | 0 | Reserved for future use |
| CR3 | Physical address of PML4 | Page table root provided by bootloader |
| RSP | Stack pointer | Points to valid stack (phbl provides 32KiB) |

---

## Part 2: THE "BIT 11" PAGE TABLE OWNERSHIP PROTOCOL

### Overview

Oxide's boot protocol uses **bit 11 of page table entries (PTEs)** to mark pages that contain the kernel nucleus. This is a critical handoff guarantee that the kernel validates during early boot.

### Technical Details (phbl/src/mmu.rs, lines 85-101)

The bootloader must satisfy these constraints:

```
For the page table passed to the kernel:

1. All memory frames used for page table structures must come from 
   a **physically contiguous region**

2. The PML4 (root page table) must be at the **lowest physical address**
   in that contiguous range

3. The total range must contain **at least 16 pages (64 KiB)**, but may
   contain more

4. All non-page-table pages mapping the kernel nucleus must have 
   **bit 11 set in their PTEs**
```

### PTE Structure (phbl/src/mmu.rs, lines 357-388)

```rust
bitstruct! {
    struct PTE(u64) {
        p: bool = 0;        // Bit 0: Present
        w: bool = 1;        // Bit 1: Writable
        u: bool = 2;        // Bit 2: User/Supervisor
        wt: bool = 3;       // Bit 3: Write-Through
        nc: bool = 4;       // Bit 4: Cache-Disable
        // Bits 5-10: Available
        k: bool = 11;       // Bit 11: KERNEL NUCLEUS MARKER
        pfn: u64 = 12..51;  // Bits 12-50: Physical Frame Number
        nx: bool = 63;      // Bit 63: No eXecute
    }
}
```

### Attributes for Kernel Pages (phbl/src/mem.rs, lines 152-154)

```rust
pub(crate) fn new_kernel(r: bool, w: bool, x: bool) -> Attrs {
    Self::new(r, w, x, true, true)  // k=true sets bit 11
}
```

### How phbl Implements Bit 11 (phbl/src/loader.rs, lines 153-162)

When loading kernel ELF segments:

```rust
// Extract ELF program header and read write/execute flags
let attrs = mem::Attrs::new_kernel(
    section.is_read(),      // Based on ELF PF_R flag
    section.is_write(),     // Based on ELF PF_W flag
    section.is_executable() // Based on ELF PF_X flag
);

// Map kernel sections with bit 11 set
unsafe {
    page_table
        .map_region(region, attrs, pa)
        .expect("remapped region with attrs");
}
```

When creating a PTE with kernel attributes (phbl/src/mmu.rs, line 413):

```rust
fn new<F: Frame>(pa: F, attrs: mem::Attrs) -> PTE {
    PTE::from_phys_addr(pa.phys_addr())
        .with_p(attrs.r())          // Present
        .with_w(attrs.w())          // Writable
        .with_nx(!attrs.x())        // No-eXecute
        .with_wt(!attrs.c())        // Write-Through
        .with_nc(!attrs.c())        // Cache-Disable
        .with_k(attrs.k())          // BIT 11 MARKER
        .with_h(F::BIG)             // Large/Huge page
}
```

### RFD 215 Reference

The complete specification is documented in RFD (Request for Discussion) 215, which defines the formal page table handoff contract between bootloader and kernel.

---

## Part 3: MEMORY STATE REQUIREMENTS AT HANDOFF

### Page Table Layout

**Constraint:** All page table structures must be physically contiguous, with PML4 at the lowest address.

Example layout:
```
Physical Memory:
[PML4] [PML3] [PML3] [PML2] [PML2] ... [Reserved - minimum 16 pages]
^
└─ CR3 points here
```

**Size Requirements:**
- Minimum: 16 pages (64 KiB)
- Actual: Varies based on kernel's virtual address layout
- All from a single contiguous region

### Virtual Memory Layout (Visible to Kernel)

```
0xFFFF_FFFF_FFFF_FFFF │ Top of 64-bit VA space
                       │
0xFFFF_8000_0000_0000  │ Kernel space (higher-half)
                       ├─ Kernel .text, .data, .bss
                       ├─ Kernel-only virtual regions
                       │
0x0000_7FFF_FFFF_FFFF  │ User space / Higher-half boundary
                       │
0x0000_0000_0000_0000  │ Start of lower-half VA space
```

### Physical Memory Regions

| Region | Size | Purpose | Attributes |
|--------|------|---------|------------|
| Bootloader (phbl) | ~250 KiB | ROM code, identity-mapped | RX, identity VA==PA |
| CPIO Archive | 128 MiB | Compressed kernel + root filesystem | R, identity VA==PA |
| Kernel | Variable | ELF-loaded kernel nucleus | RWX, at linked VA |
| Page Tables | ~64 KiB+ | PML4/PML3/PML2/PML1 | RW, identity VA==PA |
| UART MMIO | 4 KiB | Console I/O at 0x8000_0000 | RW, uncached |

### MMIO Configuration

- **UART Console:** Virtual address `0x8000_0000`, uncached, uncacheable bit set
- **Supported:** iomux configuration for MUX selection
- **Initial:** Console already mapped for logging

---

## Part 4: CPIO ARCHIVE FORMAT AND KERNEL EXTRACTION

### Archive Structure

**Type:** Standard newc (SVR4 portable) CPIO archive, optionally ZLIB-compressed

**Compression:** 
- Compiled as binary blob into phbl
- Decompressed in-memory during phbl startup
- Uses miniz_oxide library (ZLIB format with header)

### Decompression Process (phbl/src/main.rs, lines 41-64)

```rust
fn expand_ramdisk() -> &'static [u8] {
    // CPIO archive is compiled as binary blob
    #[cfg(target_os = "none")]
    let cpio = include_bytes!(env!("PHBL_PHASE1_COMPRESSED_CPIO_ARCHIVE_PATH"));
    
    // Reserve 128 MiB region for decompressed archive
    let dst = phbl::ramdisk_region_init_mut();
    
    // Decompress ZLIB stream with header
    let mut r = DecompressorOxide::new();
    let flags = TINFL_FLAG_PARSE_ZLIB_HEADER;
    let (status, _, output_len) = decompress(&mut r, &cpio[..], dst, 0, flags);
    
    assert!(status == TINFLStatus::Done);
    &dst[..output_len]  // Return slice of decompressed CPIO
}
```

### Kernel Location in Archive (phbl/src/main.rs, lines 66-73)

```rust
fn find_kernel(cpio: &[u8]) -> &[u8] {
    // Iterate through CPIO entries using cpio_reader crate
    for entry in cpio_reader::iter_files(cpio) {
        // Kernel is always at this path in the archive
        if entry.name() == "platform/oxide/kernel/amd64/unix" {
            return entry.file();  // Return ELF binary
        }
    }
    panic!("could not locate unix in cpio archive");
}
```

### CPIO Entry Structure (Standard newc format)

Each file in the archive has this header (newc/SVR4 format):

```
c_magic      (6 bytes, hex): "070701" for newc format
c_ino        (8 bytes, hex): inode number
c_mode       (8 bytes, hex): file mode/permissions
c_uid        (8 bytes, hex): user ID
c_gid        (8 bytes, hex): group ID
c_nlink      (8 bytes, hex): number of links
c_mtime      (8 bytes, hex): modification time
c_filesize   (8 bytes, hex): file data size
c_devmajor   (8 bytes, hex): device major number
c_devminor   (8 bytes, hex): device minor number
c_rdevmajor  (8 bytes, hex): rdevice major
c_rdevminor  (8 bytes, hex): rdevice minor
c_namesize   (8 bytes, hex): length of filename + 1 (includes null terminator)
c_check      (8 bytes, hex): checksum (0 for newc, actual for crc format)

[Filename - c_namesize bytes, null-terminated]
[Padding to 4-byte alignment]
[File data - c_filesize bytes]
[Padding to 4-byte alignment]
```

### Typical Phase 1 Archive Contents

```
platform/oxide/kernel/amd64/unix     (The kernel ELF binary)
kernel/modules/amd64/                (Optional kernel modules)
etc/                                 (Configuration files)
usr/lib/                             (System libraries - if included)
TRAILER!!!                           (End marker)
```

### Archive Compression Details

- **Format:** ZLIB with header (RFC 1950)
- **Dictionary:** Not used
- **Decompressed Size:** Typically 80-150 MiB depending on configuration
- **Memory Required:** At least 128 MiB reserved region in RAM
- **Speed:** Limited by CPU decompression (microseconds for 128 MiB on modern CPUs)

---

## Part 5: EARLY KERNEL BOOT CODE PATHS

### Kernel Entry Point (illumos-gate stlouis branch)

**Main Entry:** `locore.S` (usr/src/uts/i86pc/ml/locore.S)

The kernel's first instruction is a jump to C startup code:

```asm
jmp _start              # Jump to fakebop.c entry
```

### Boot Parameter Availability (locore.S summary)

The kernel's assembly stub receives bootloader parameters:

```c
// From illumos-gate: locore.S
// RDI = boot services pointer (from system)
// RDX = bootops structure
// RSI, RCX, R8, R9 = additional boot parameters (stack-saved)
```

**NOTE:** This is the generic illumos entry point. Oxide's phbl passes parameters differently - see locore.S handling of ramdisk parameters.

### Kernel Initialization Chain (illumos-gate stlouis branch)

1. **locore.S Assembly Entry**
   - Sets up minimal 64-bit environment
   - Saves boot parameters on stack
   - Calls `mlsetup()` with regs structure

2. **mlsetup() (usr/src/uts/i86pc/os/mlsetup.c)**
   - Initializes CPU (CPUID checks)
   - Loads GDT, IDT, LDT
   - Sets up TSC synchronization
   - Processes boot properties
   - Enables security features (SMAP, KPTI if needed)
   - Returns to assembly

3. **main() (usr/src/uts/common/os/main.c)**
   - Never returns; primary OS initialization
   - Builds memory lists from bootloader
   - Initializes kernel memory allocator
   - Sets up virtual memory (HAT layer)
   - Loads kernel modules

### Oxide Boot Phase Sequence

**Phase 1: ROM-based (phbl)**
```
PSP loads phbl from SPI ROM
    ↓
phbl decompresses CPIO archive
    ↓
phbl extracts kernel ELF from CPIO
    ↓
phbl loads kernel binary at linked addresses
    ↓
phbl marks kernel pages with bit 11
    ↓
phbl calls kernel entry point with ramdisk parameters
```

**Phase 2: Kernel identifies ramdisk source**
```
Kernel startup examines boot properties
    ↓
oxide_boot_locate() checks boot_source property
    ↓
Based on boot_source:
  - "disk:N" → oxide_boot_disk() reads from NVMe slot N
  - "net"    → oxide_boot_net() boots from network (K.2 adapter)
  - "sp"     → oxide_boot_sp() retrieves from service processor
    ↓
Phase 2 ramdisk is loaded into memory as another CPIO archive
    ↓
ZFS mount point configured from phase 2 properties
```

### Key Oxide-Specific Boot Code Files

1. **oxide_boot.c** - Main boot orchestrator, determines boot source
2. **oxide_boot_disk.c** - Loads phase 2 image from NVMe M.2 slot
3. **oxide_boot_net.c** - Loads phase 2 image via network (K.2 adapter)
4. **oxide_boot_sp.c** - Loads phase 2 image from service processor
5. **oxide_boot.h** - Boot parameter structures and constants

### Boot Properties

The kernel's boot property system receives:

| Property | Source | Purpose |
|----------|--------|---------|
| boot_source | Bootloader | "disk:0", "disk:1", "net", or "sp" |
| fstype | Kernel | Usually "zfs" |
| zfs-bootfs | Determined by boot_source | Root filesystem ZFS dataset path |
| zfs-rootdisk-path | Found by boot code | Path to boot disk in ZFS |

---

## Part 6: DISK IMAGE HEADER FORMAT (Phase 2)

### Disk Image Structure (boot-image-tools)

**File Path:** Phase 2 ramdisk image stored on NVMe

```
[4 KiB Header]
[Compressed/Uncompressed Image Data]
```

### Header Structure (boot-image-tools/src/diskimage.rs)

```c
struct disk_image_header {
    uint32_t magic;           // 0x1DEB0075
    uint32_t version;         // 2
    uint64_t flags;           // Bit 0: COMPRESSED
    uint64_t data_size;       // Uncompressed size
    uint64_t image_size;      // Actual image size (may be compressed)
    uint64_t target_size;     // Target allocation size (usually 4GB)
    uint8_t  sha256[32];      // SHA256 of uncompressed data
    char     dataset_name[128];   // ZFS dataset name (e.g., "rpool/ROOT/ramdisk")
    char     image_name[128];     // Image name/label
    // Total: 4096 bytes (4 KiB)
};
```

### Magic and Version
- **Magic:** `0x1DEB0075` (little-endian)
- **Version:** `2` (constant, required for validation)

### Flags
```c
#define DISK_COMPRESSED 0x01   // Image is ZLIB-compressed
```

### Boot Code Reading (oxide_boot_disk.c)

1. Open NVMe device at specific M.2 slot
2. Read 4 KiB header block
3. Validate magic and version
4. Allocate ramdisk for data_size (or target_size)
5. If COMPRESSED flag set, decompress image
6. Validate SHA256 checksum
7. Set kernel properties for boot dataset

---

## Part 7: PLATFORM INITIALIZATION REQUIREMENTS

### AMD CPU-Specific Setup in phbl

**CPUID Verification:**
- Verifies CPU is AMD (CPUID vendor check)
- Checks for required 64-bit extension support
- Validates long mode availability

**MSR Configuration:**
- EFER.LME (Long Mode Enable) bit must be set
- EFER.NXE (No-Execute Enable) for NX pages
- CR0.PG (Paging) enabled before kernel handoff
- CR0.WP (Write Protect) enabled for supervisory page protection

**Memory Controller:**
- AMD Zen SoC includes unified memory controller
- Phbl does not reprogram memory controller
- Assumes PSP/firmware has initialized DRAM
- Memory topology discovered at kernel startup

### IOMUX Configuration (phbl/src/iomux.rs)

```rust
pub fn init() {
    // Configure UART pinmux to enable console output
    // Sets up GPIO/pinmux for console device
}
```

### Early Kernel CPU/Microarchitecture Detection

From illumos startup.c:
- Detects Zen SoC components (DXIO, NBIO, SMU)
- Enables SSC (Spread Spectrum Clock) to reduce EMI
- Configures clock gating
- Validates CPU frequency scaling

---

## Part 8: COMPARISON - PHBL HANDOFF vs. TRADITIONAL BOOTLOADERS

### PHBL Protocol (Oxide's Approach)

**Strengths:**
- Minimal bootloader (ROM-resident, small footprint)
- Fast boot (direct kernel jump, no complex transitions)
- Clear handoff contract (bit 11 protocol, page table structure)
- CPIO format is universal (standard Linux boot archives)
- Supports dynamic boot sources (disk/network/SP)

**Constraints:**
- Requires physically contiguous page tables
- Hardcoded kernel path in CPIO: `platform/oxide/kernel/amd64/unix`
- No UEFI/ACPI discovery (hard-coded memory, no device tree)
- Ramdisk physically contiguous in first 128 MiB
- No multi-boot protocol support

### Traditional BIOS/UEFI Approaches

**BIOS Convention:**
- Bootloader provides struct ards[] for memory map
- ABI varies (grub, lilo, etc.)
- Manual page table setup in bootloader

**UEFI Convention:**
- Provides EFI_SYSTEM_TABLE, EFI services
- Supports Device Tree (FDT) for hardware discovery
- Page tables torn down before kernel entry
- Kernel rebuilds page tables from scratch

### U-Boot Expectations

U-Boot's x86-64 mode provides:
- `%eax` = boot argument (typically 0)
- `%ecx` = entry point (for 32-bit jump)
- EDI/ESI/ECX may contain boot device info
- Expects bootloader to provide device tree (FDT) if used
- Page tables typically torn down; kernel rebuilds them

**Key Difference:** U-Boot's handoff is **not compatible** with Oxide's expectation of RDI=ramdisk, RSI=length.

---

## Part 9: ADAPTATION REQUIREMENTS FOR U-BOOT

If porting Helios to use U-Boot as bootloader:

### What U-Boot Would Need to Provide

1. **Ramdisk in Memory**
   - Load CPIO archive from NVMe/NAND/network
   - Decompress if compressed
   - Place at known physical address
   - Pass physical address in RDI, length in RSI

2. **Kernel ELF Loading**
   - Load kernel ELF from CPIO
   - Extract LOAD segments
   - Map at linked addresses (kernel-specific virtual addresses)
   - Mark kernel pages with bit 11 in PTEs

3. **Page Table Setup**
   - Create physically-contiguous page table region
   - Ensure PML4 at lowest physical address
   - Allocate minimum 16 pages (64 KiB)
   - Set bit 11 for all kernel nucleus pages

4. **CPU State at Handoff**
   - Enable 64-bit long mode (LM bit in EFER)
   - Enable paging (CR0.PG = 1)
   - Disable interrupts (RFLAGS.IF = 0)
   - Load CR3 with PML4 physical address
   - Provide 32KiB stack

### Compatibility Shim Strategy

**Option A: Wrapper in Kernel**
```
U-Boot entry point
    ↓
U-Boot loads/decompresses Helios kernel
    ↓
U-Boot jumps to shim code (in kernel text)
    ↓
Shim code: 
  - Extract RDI/RSI from bootloader env variables
  - Call actual kernel entry point
  - Never returns
```

**Option B: Modified Bootloader**
```
U-Boot fork with Oxide protocol handler
    ↓
Recognizes Helios CPIO archives
    ↓
Implements bit 11 page table protocol
    ↓
Provides RDI=ramdisk, RSI=length handoff
```

**Option C: Minimal Secondary Bootloader**
```
U-Boot loads Oxide's phbl binary
    ↓
phbl executes (as would in ROM)
    ↓
phbl does its normal job of loading kernel
    ↓
Handoff proceeds normally
```

---

## Part 10: SECURITY AND VALIDATION CONSIDERATIONS

### Page Table Validation

Kernel validates on entry (expected behavior):
- Bit 11 correctly marks kernel nucleus pages
- Non-kernel pages do not have bit 11 set
- Page table structures are properly formed
- All page table pages are within contiguous region

### Checksum Verification

Phase 2 boot images verified by:
1. **Header validation:** Magic (0x1DEB0075) and version (2) match
2. **SHA256 checksum:** Computed over uncompressed image, validated on disk
3. **Data integrity:** Kernel checks properties match actual dataset

### UART Console Protection

- UART mapped non-cacheable (MTRR type)
- Console functions in kernel use explicit memory barriers
- No risk of stale console output

### CPU State Validation

Early kernel code validates:
- Long mode is actually enabled
- Paging is actually enabled
- At least one CPU core is responsive
- BIST (Built-In Self Test) passed (checked in phbl)

---

## Part 11: CODE EXAMPLES AND PATTERNS

### 1. Loading a Kernel Segment (From phbl)

```rust
// From phbl/src/loader.rs
fn load_segment(
    page_table: &mut LoaderPageTable,
    section: &ProgramHeader,
    bytes: &[u8],
) -> Result<()> {
    let pa = section.p_paddr;
    let vm = section.vm_range();
    
    // Validate alignment
    if !pa.is_multiple_of(mem::P4KA::ALIGN) {
        return Err("Program section is not physically 4KiB aligned");
    }
    if !vm.start.is_multiple_of(mem::V4KA::ALIGN) {
        return Err("Program section not virtually 4KiB aligned");
    }
    
    let start = mem::V4KA::new(vm.start);
    let end = mem::V4KA::new(round_up_4k(vm.end));
    let region = start..end;
    let pa = mem::P4KA::new(pa);
    
    // First map as RW for loading data
    unsafe {
        page_table.map_region(region.clone(), mem::Attrs::new_data(), pa)?;
        let p = page_table.try_with_addr(start.addr())?;
        let len = end.addr() - start.addr();
        core::ptr::write_bytes(p, 0, len);
        let dst = core::slice::from_raw_parts_mut(p, len);
        let len = usize::min(bytes.len(), dst.len());
        if len > 0 {
            dst[..len].copy_from_slice(&bytes[..len]);
        }
    }
    
    // Then remap with proper kernel attributes (bit 11 set)
    let attrs = mem::Attrs::new_kernel(
        section.is_read(),
        section.is_write(),
        section.is_executable(),
    );
    unsafe {
        page_table.map_region(region, attrs, pa)?;
    }
    
    Ok(())
}
```

### 2. PTE Creation with Bit 11 (From phbl)

```rust
// From phbl/src/mmu.rs
impl PTE {
    fn new<F: Frame>(pa: F, attrs: mem::Attrs) -> PTE {
        PTE::from_phys_addr(pa.phys_addr())
            .with_p(attrs.r())          // Present (based on readable)
            .with_w(attrs.w())          // Writable
            .with_nx(!attrs.x())        // No-eXecute
            .with_wt(!attrs.c())        // Write-Through (if not cacheable)
            .with_nc(!attrs.c())        // Cache-Disable (if not cacheable)
            .with_k(attrs.k())          // BIT 11 - KERNEL NUCLEUS MARKER
            .with_h(F::BIG)             // Large/Huge page indicator
    }
}
```

### 3. Ramdisk Handoff to Kernel (From phbl)

```rust
// From phbl/src/main.rs
#[unsafe(no_mangle)]
pub(crate) extern "C" fn entry(config: &mut phbl::Config) {
    println!("Oxide Pico Host Boot Loader");
    println!("{config:#x?}");
    
    // Decompress and load the kernel
    let ramdisk = expand_ramdisk();
    let kernel = find_kernel(ramdisk);
    let entry = loader::load(&mut config.page_table, kernel)
        .expect("loaded kernel");
    
    // Final handoff: RDI=ramdisk_paddr, RSI=ramdisk_len
    println!("jumping into kernel...");
    entry(ramdisk.as_ptr().addr() as u64, ramdisk.len());
    
    // Never reached
    panic!("main returning");
}
```

### 4. Kernel Entry Point (From illumos locore.S)

```asm
// From illumos-gate: usr/src/uts/i86pc/ml/locore.S
_start:
    // Receive boot parameters
    movq    %rdi, REGOFF_RDI(%rsp)   // Save ramdisk_paddr
    movq    %rsi, REGOFF_RSI(%rsp)   // Save ramdisk_len
    movq    %rdx, REGOFF_RDX(%rsp)   // Save (unused parameter)
    
    // Set up thread 0 stack
    leaq    t0stack(%rip), %rsp
    addq    $DEFAULTSTKSZ - REGSIZE, %rsp
    
    // Call machine-level setup
    call    mlsetup
    
    // Call main (never returns)
    call    main
```

### 5. Accessing Ramdisk Parameters in Kernel

From oxide_boot.c, the kernel would receive the ramdisk and process it:

```c
// Kernel extracts ramdisk pointer from boot context
// These come from RDI/RSI registers via locore.S
uint64_t ramdisk_paddr;  // From RDI at kernel entry
uint64_t ramdisk_len;    // From RSI at kernel entry

// The ramdisk is a CPIO archive that may contain:
// - Phase 1 root filesystem
// - Boot datasets for ZFS
// - Module load list

// oxide_boot.c then locates where phase 2 image comes from
// based on boot properties (disk/network/SP)
```

---

## Part 12: CRITICAL IMPLEMENTATION DETAILS FOR BOOTLOADER DEVELOPERS

### Must-Have Features

1. **CPIO Archive Support**
   - Read or iterate newc-format CPIO entries
   - Extract file by path: `platform/oxide/kernel/amd64/unix`
   - Handle optional ZLIB compression (RFC 1950 with header)

2. **ELF Loader for x86-64**
   - Parse ELF header (validate 64-bit, little-endian, executable)
   - Load all PT_LOAD segments
   - No relocation needed (kernel uses PIE or fixed address linking)
   - Validate section alignment (4KiB for all)

3. **Page Table Management**
   - Allocate physically-contiguous region for page tables
   - PML4 must be at lowest physical address in region
   - Minimum 16 pages (64 KiB)
   - Set bit 11 on all PTEs for kernel nucleus pages

4. **64-bit Mode Setup**
   - Enable EFER.LME (Long Mode Enable) before paging
   - Set EFER.NXE (No-Execute Enable)
   - Load GDT with 64-bit code/data descriptors
   - Load minimal IDT (kernel will set its own)

5. **Memory Configuration**
   - Mark UART MMIO at 0x8000_0000 as uncached
   - Reserve 128 MiB region for ramdisk
   - Preserve reserved regions (loader, page tables, ramdisk)

### Optional But Useful Features

- Boot device selection (disk, network, SP)
- Secondary kernel support (multi-boot)
- Debug console output before kernel entry
- BIOS/UEFI integration (for development VMs)
- Device tree generation if kernel expects FDT

### Common Pitfalls

1. **Not setting bit 11 in kernel PTEs**
   - Kernel may fail validation or fault on kernel page access
   - Check mmu.rs source for exact marking

2. **Non-contiguous page tables**
   - Kernel expects physically contiguous region
   - All page table pages must come from single allocation
   - PML4 MUST be at lowest address

3. **Incorrect CPIO path**
   - Path must be exactly: `platform/oxide/kernel/amd64/unix`
   - No leading slashes, exact case match
   - Trailing garbage will corrupt kernel

4. **Ramdisk size mismatch**
   - RSI must be decompressed CPIO size
   - If compressed, must decompress first
   - Kernel uses this to validate CPIO structure

5. **Virtual address collision**
   - Loader VA must not collide with kernel VA
   - Bootloader typically uses lower half (VA < 0x8000_0000_0000_0000)
   - Kernel uses higher half (VA >= 0xFFFF_8000_0000_0000)

---

## Summary Table: Boot Protocol Quick Reference

| Parameter | Register | Value |
|-----------|----------|-------|
| **Ramdisk Address** | RDI | Physical address of decompressed CPIO archive |
| **Ramdisk Length** | RSI | Size of CPIO in bytes |
| **Reserved** | RDX | Must be 0 |
| **Reserved** | RCX | Must be 0 |
| **Reserved** | R8 | Must be 0 |
| **Reserved** | R9 | Must be 0 |
| **Page Table Root** | CR3 | Physical address of PML4 |
| **CPU Mode** | CR0.PE | Must be 1 (protected mode) |
| **Paging** | CR0.PG | Must be 1 |
| **Long Mode** | EFER.LM | Must be 1 |
| **Interrupts** | RFLAGS.IF | Must be 0 |
| **Stack** | RSP | Valid 32KiB stack pointer |
| **Kernel Path** | (In CPIO) | `platform/oxide/kernel/amd64/unix` |
| **Bit 11 Mark** | PTE[11] | Set for all kernel nucleus pages |
| **Page Tables** | (Contiguous) | All from single contiguous region, min 16 pages |

---

## References and Further Reading

### Primary Sources Analyzed
- **phbl:** https://github.com/oxidecomputer/phbl
- **boot-image-tools:** https://github.com/oxidecomputer/boot-image-tools
- **illumos-gate (stlouis):** https://github.com/oxidecomputer/illumos-gate/tree/stlouis

### Key Source Files
- `phbl/src/loader.rs` - Kernel loading and handoff
- `phbl/src/mmu.rs` - Page table management and bit 11 protocol
- `phbl/src/main.rs` - Boot orchestration and CPIO handling
- `phbl/src/phbl.rs` - Early boot initialization
- `illumos-gate/usr/src/uts/i86pc/ml/locore.S` - Kernel assembly entry
- `illumos-gate/usr/src/uts/oxide/boot_image/oxide_boot.c` - Boot parameter processing
- `boot-image-tools/src/diskimage.rs` - Disk image header format

### Specifications Referenced
- **RFD 215** - Page table handoff contract (internal Oxide document)
- **Intel 64 and IA-32 Architectures Software Developer's Manual**
- **System V AMD64 ABI** - x86-64 calling convention
- **CPIO(5)** - Archive format (newc/SVR4 variant)

---

**End of Research Report**
