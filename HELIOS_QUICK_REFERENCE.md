# Helios Boot Protocol - Quick Reference for V3000 Bootloader

## The Handoff Protocol (EXACT SPECIFICATION)

### Kernel Entry Point Signature
```c
// extern "C" calling convention
void kernel_entry(
    uint64_t rdi,   // Physical address of CPIO archive
    uint64_t rsi,   // Archive length in bytes
    uint64_t rdx,   // Reserved (must be 0)
    uint64_t rcx,   // Reserved (must be 0)
    uint64_t r8,    // Reserved (must be 0)
    uint64_t r9     // Reserved (must be 0)
);
```

### Required CPU State at Entry
- 64-bit long mode (EFER.LM=1)
- Paging enabled (CR0.PG=1)
- Write protection enabled (CR0.WP=1)
- Interrupts disabled (RFLAGS.IF=0)
- 32 KiB stack available
- GDT loaded with minimal descriptors
- CR3 loaded with PML4 physical address

### Critical: The "Bit 11" Protocol
**Every page table entry pointing to kernel code/data MUST have bit 11 set.**

```
PTE format:
[63:63] NX - No Execute bit
[62:51] PFN - Physical Frame Number
[11:11] K - KERNEL BIT (MUST BE SET FOR KERNEL PAGES!)
[10:5]  Reserved
[4:4]   NC - Cache Disable bit
[3:3]   WT - Write Through
[2:2]   U - User/Supervisor
[1:1]   W - Writable
[0:0]   P - Present
```

### Page Table Constraints
1. All page table structures must be physically contiguous
2. PML4 must be at the lowest physical address in that region
3. Minimum 16 pages (64 KiB), but may be larger
4. All non-page-table pages mapping kernel must have bit 11 set

---

## CPIO Archive Format

### Archive Location and Access
- **Path to kernel:** `platform/oxide/kernel/amd64/unix` (exact match, case-sensitive)
- **Format:** Standard newc (SVR4 portable) CPIO archive
- **Compression:** ZLIB with RFC 1950 header
- **Archive size:** 80-150 MiB typically (decompressed)
- **Reserved region:** 128 MiB minimum

### Decompression
```
1. Embedded as binary blob in bootloader
2. Must decompress before kernel entry
3. Use ZLIB decompressor with RFC 1950 header flag
4. Place decompressed archive in memory
5. Pass physical address in RDI, size in RSI
```

---

## Phase 1 vs Phase 2 Boot

### Phase 1 (Bootloader → Kernel)
- Bootloader passes compressed/decompressed CPIO in memory
- Kernel ELF extracted and loaded by bootloader
- Kernel receives ramdisk in RDI/RSI
- Path: `platform/oxide/kernel/amd64/unix`

### Phase 2 (Kernel → Root Filesystem)
- Kernel determines boot source from boot properties
- Sources: disk (NVMe), network (K.2), service processor
- Disk image format: 4 KiB header + ZLIB-compressed data
- Header magic: `0x1DEB0075`
- Header version: `2`

---

## Disk Image Header Format

```c
struct disk_header {
    uint32_t magic;              // 0x1DEB0075
    uint32_t version;            // 2
    uint64_t flags;              // Bit 0: COMPRESSED
    uint64_t data_size;          // Uncompressed size
    uint64_t image_size;         // Actual file size
    uint64_t target_size;        // Target allocation (4GB)
    uint8_t  sha256[32];         // SHA256 hash
    char     dataset_name[128];  // "rpool/ROOT/ramdisk"
    char     image_name[128];    // Image label
};
```

---

## Virtual Memory Layout

```
0xFFFF_FFFF_FFFF_FFFF ├─ Top of VA space
                       │
0xFFFF_8000_0000_0000  ├─ Kernel (higher-half)
                       │  └─ .text, .data, .bss
                       │
0x0000_7FFF_FFFF_FFFF  ├─ User space
                       │
0x0000_0000_0000_0000  └─ Start
```

**Bootloader typically uses:** Lower half (VA < 0x8000_0000_0000_0000)  
**Kernel uses:** Higher half (VA >= 0xFFFF_8000_0000_0000)

---

## MMIO Configuration

- UART console at **0x8000_0000** (unmapped initially by bootloader)
- Must be mapped as uncached (NC bit set in PTEs)
- 4 KiB page at minimum
- Used for console I/O before kernel boot

---

## Critical Implementation Checklist for V3000 Bootloader

### Must Implement
- [ ] CPIO archive reader (newc format)
- [ ] ZLIB decompressor (RFC 1950)
- [ ] ELF loader for x86-64 LOAD segments
- [ ] 64-bit page table setup
- [ ] Bit 11 marking in kernel PTEs
- [ ] Physically-contiguous page table allocation
- [ ] Long mode (EFER.LME) enable
- [ ] CR3 load with PML4 physical address

### Security Validations
- [ ] Kernel path exact match: `platform/oxide/kernel/amd64/unix`
- [ ] Bit 11 set on all kernel pages
- [ ] Bit 11 NOT set on non-kernel pages
- [ ] Page tables properly formed and accessible
- [ ] BIST validation in bootloader startup

### Common Mistakes to Avoid
1. **NOT setting bit 11** - Kernel will fail
2. **Non-contiguous page tables** - Kernel expects one contiguous region
3. **Wrong kernel path** - "unix" vs other filenames
4. **Size mismatch** - RSI must match decompressed CPIO size
5. **CPIO not decompressed** - Must decompress before passing to kernel

---

## Compatibility Notes: U-Boot Integration

### What U-Boot Would Need
If using U-Boot as bootloader instead of phbl:

1. Load CPIO archive from storage
2. Decompress if needed
3. Extract kernel ELF from CPIO
4. Load kernel sections at linked virtual addresses
5. Create physically-contiguous page tables
6. Mark kernel pages with bit 11
7. Jump with: RDI=ramdisk_paddr, RSI=ramdisk_len

### Not Compatible Without Shim
- U-Boot's default handoff is **NOT compatible**
- Would need custom x86 boot code modifications
- Alternative: Use phbl as secondary bootloader

---

## Boot Properties Used by Kernel

| Property | Default Value | Purpose |
|----------|---------------|---------|
| boot_source | "sp" | Where to load phase 2: "disk:0", "disk:1", "net", "sp" |
| fstype | "zfs" | Filesystem type |
| zfs-bootfs | (determined by boot code) | Root filesystem dataset |
| zfs-rootdisk-path | (discovered) | Physical path to boot disk |

---

## Memory Map Example

```
Physical Memory Layout:
0x0000_0000 ├─ DRAM Start
            │
0x0800_0000 ├─ CPIO Archive (128 MiB region)
            │  └─ Decompressed CPIO contents
            │
0x1000_0000 ├─ Kernel Code/Data
            │  └─ Loaded ELF segments
            │
0xNNNN_NNNN ├─ Page Tables (contiguous, ~64 KiB+)
            │  ├─ PML4 (at lowest PA in region)
            │  ├─ PML3 entries
            │  ├─ PML2 entries
            │  └─ PML1 entries
            │
0xFFFF_FFFF └─ Top of memory
```

---

## Key Source Files Reference

| File | Purpose |
|------|---------|
| phbl/src/loader.rs | Kernel ELF loading and handoff |
| phbl/src/mmu.rs | Page table setup and bit 11 protocol |
| phbl/src/main.rs | Boot orchestration |
| illumos-gate/usr/src/uts/i86pc/ml/locore.S | Kernel assembly entry |
| boot-image-tools/src/diskimage.rs | Phase 2 image header |

---

## Architecture Decision Points for V3000

### Option 1: Implement Full Helios Compatibility
**Pros:** Uses Helios kernel unmodified, shared maintenance
**Cons:** Must implement bit 11 protocol, CPIO handling, complex page tables

### Option 2: Simplified Kernel Entry
**Pros:** Simpler bootloader logic
**Cons:** Requires kernel modifications, diverges from Oxide

### Option 3: Use phbl as Secondary
**Pros:** Reuses existing, tested bootloader
**Cons:** Adds complexity layer, requires ROM space for phbl

---

**Last Updated:** 2024-11-18  
**Based on:** https://github.com/oxidecomputer/helios repositories  
**Reference:** See HELIOS_BOOT_PROTOCOL_RESEARCH.md for detailed analysis
