# CLAUDE.md - phbl (Pico Host Boot Loader) for AMD V3000

## Repository Purpose
Fork of Oxide's minimal Rust bootloader, adapted for AMD Ryzen Embedded V3000 (V3C18I) platform. Target: ~3,000 lines replacing traditional UEFI firmware.

## Critical Context
- **Parent**: https://github.com/oxidecomputer/phbl
- **Architecture**: x86_64 bare-metal Rust, no_std
- **Scope**: CPU mode transitions, UART console, kernel loading ONLY
- **Non-scope**: No hardware init, no drivers, no filesystems

## Priority Task List

### P0: Foundation (BLOCKING)
- [ ] Study original phbl architecture thoroughly (src/main.rs, src/bootloader.rs)
- [ ] Document differences between AMD EPYC Milan and V3000 platforms
- [ ] Create V3000 target specification (x86_64-v3000-none-elf.json)
- [ ] Setup xtask build system for V3000

### P1: CPU Mode Transitions
- [ ] Port 16-bit real mode entry (src/reset.S or equivalent)
- [ ] Verify 32-bit protected mode transition works on V3000
- [ ] Implement 64-bit long mode with paging
- [ ] Test transitions in QEMU before hardware

### P2: V3000-Specific Adaptations
- [ ] Add V3000 CPUID detection (Family 0x19, Model 0x40-0x4F)
- [ ] Verify UART pins for FP7r2 package (likely same: 135-138)
- [ ] Verify UART clock (30 MHz or 48 MHz)
- [ ] UART/GPIO addresses likely same as EPYC (0xFEDC_9000, 0xFED8_0000)

**Note**: Platform comparison shows V3000 uses same AMD FCH as EPYC. Most addresses are standardized across AMD platforms. Changes are minimal (~35 lines).

### P3: Memory Management
- [ ] Port page table construction algorithm (keep 1GiB/2MiB optimization)
- [ ] Update identity mapping for V3000 memory layout
- [ ] Implement kernel virtual address mapping
- [ ] Add bit 11 ownership transfer protocol

### P4: Boot Integration
- [ ] Define handoff from first-stage loader (state guarantees)
- [ ] Implement U-Boot ELF loading (replace Helios kernel loading)
- [ ] Add compressed payload support (miniz_oxide)
- [ ] Create boot parameter passing protocol

### P5: Hardware Validation
- [ ] Get "Hello World" on SolidRun HoneyComb serial console
- [ ] Validate memory initialization from PSP
- [ ] Test full boot chain to U-Boot
- [ ] Measure and optimize boot time (<1 second target)

### P6: Production Features
- [ ] Add boot counting for A/B redundancy
- [ ] Implement integrity checking
- [ ] Create recovery mode support
- [ ] Add manufacturing/debug modes

## Key Files to Modify

```
src/
├── main.rs          # Entry point - update for V3000
├── bootloader.rs    # Core logic - mostly reusable
├── uart.rs          # NEEDS REWRITE for V3000 addresses
├── memory.rs        # Update memory map
├── page_tables.rs   # Keep algorithm, update constants
├── elf.rs          # Reuse for U-Boot loading
└── v3000/          # NEW: Platform-specific code
    ├── registers.rs # V3000 MSRs and IO addresses
    ├── memory_map.rs # Platform memory layout
    └── io_mux.rs    # Pin routing configuration
```

## Build Configuration

```toml
# .cargo/config.toml updates needed
[target.x86_64-v3000-none-elf]
rustflags = [
    "-C", "link-arg=-Tsrc/v3000.ld",
    "-C", "link-arg=-nostdlib",
    "-C", "link-arg=-zmax-page-size=4096",
]

[build]
target = "x86_64-v3000-none-elf"
```

## Dependencies to Review
- `x86` (v0.52): CPU instructions - keep
- `bit_field`: Register manipulation - keep
- `goblin`: ELF parsing - keep for U-Boot
- `miniz_oxide`: Decompression - keep
- `static_assertions`: Compile-time checks - keep
- Consider adding: `cortex-a` for first-stage PSP interaction?

## Testing Strategy
1. **QEMU first**: Basic CPU transitions
2. **Hardware serial**: Get UART working early
3. **Memory validation**: Verify PSP initialization
4. **Boot chain**: Test each handoff point
5. **Performance**: Profile with timestamps

## Critical Unknowns (Need AMD NDA)

**Hard Blockers (Cannot Boot Without):**
- PSP firmware blobs (EmbeddedPi branch)
- AGESA binary (FP7r2)
- APCB template from AMD FAE

**Verification Needed:**
- UART pin assignments for FP7r2 package
- UART clock frequency (30 or 48 MHz)
- Any silicon errata workarounds

**Likely Known (Based on Platform Comparison):**
- CPUID: Family 0x19, Model 0x40-0x4F
- UART address: 0xFEDC_9000 (AMD FCH standard)
- GPIO address: 0xFED8_0000 (AMD FCH standard)
- Boot flow: Same PSP→ABL→x86 as EPYC

## Success Criteria
- Boots in <1 second from handoff
- Zero unsafe blocks in critical path
- <5,000 lines total Rust code
- Works on -40°C to 105°C range (V3C18I)
- Passes security audit

## References
- Oxide RFD-1: Boot Flowchart
- Oxide RFD-26: Authenticated Boot
- AMD V3000 BKDG: [REQUIRES NDA]
- SolidRun V3000 Schematics: https://solidrun.atlassian.net/
