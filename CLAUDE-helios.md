# CLAUDE.md - helios Reference Repository

## Repository Purpose
Fork of Oxide's Helios (illumos-based OS) for understanding boot handoff protocol and kernel expectations. NOT for direct use - reference only for bootloader interface.

## Study Focus Areas

### P0: Boot Protocol Understanding
- [ ] Study kernel entry expectations in kernel/os/bootops.c
- [ ] Document memory state requirements at handoff
- [ ] Understand CPIO archive format expectations
- [ ] Map page table handoff (bit 11 ownership protocol)

### P1: Relevant Code Paths
- [ ] `usr/src/uts/oxide/os/oxide_boot.c` - Boot parameter processing
- [ ] `usr/src/uts/oxide/os/mlsetup.c` - Early kernel initialization
- [ ] `usr/src/stand/lib/sa/cpio.c` - CPIO extraction
- [ ] `usr/src/uts/i86pc/os/startup.c` - x86_64 startup sequence

### P2: Adaptation for U-Boot
- [ ] Map Helios boot params to U-Boot expectations
- [ ] Document differences in kernel entry protocol
- [ ] Create translation layer if needed
- [ ] Define minimal handoff state

## Key Files to Study

```
usr/src/
├── uts/oxide/os/
│   ├── oxide_boot.c    # Boot parameter handling
│   └── fakebop.c       # Boot operations interface
├── uts/i86pc/os/
│   ├── startup.c       # x86 startup
│   └── mlsetup.c       # Machine-level setup
└── stand/lib/sa/
    └── cpio.c          # Archive format
```

## Handoff Protocol (from phbl)

```c
// Entry state from bootloader:
// - RDI: Physical address of CPIO archive
// - RSI: Archive length in bytes
// - CPU in 64-bit long mode
// - Paging enabled
// - Interrupts disabled
// - 32KiB stack available
// - APs in reset state
```

## Adaptation Notes
- U-Boot expects different entry protocol
- May need shim layer between phbl and U-Boot
- Consider direct Linux boot instead?
- Document any protocol changes needed

## Reference Value
- Shows clean handoff interface
- Demonstrates minimal boot requirements
- Provides page table transfer pattern
- Illustrates CPIO usage pattern
