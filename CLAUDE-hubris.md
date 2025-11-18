# CLAUDE.md - hubris Embedded Reference

## Repository Purpose
Fork of Oxide's Hubris embedded microkernel for studying minimal boot patterns and embedded Rust practices. Reference for understanding Service Processor interaction patterns.

## Study Focus Areas

### P0: Boot Architecture
- [ ] Study minimal boot sequence in kernel/src/arch/arm_m.rs
- [ ] Understand task initialization model
- [ ] Review memory protection setup
- [ ] Document IPC patterns for first-stage communication

### P1: Relevant Patterns
- [ ] Static memory allocation strategies
- [ ] No-heap Rust patterns
- [ ] Fault isolation architecture
- [ ] Debug infrastructure design

### P2: Applicable Concepts
- [ ] Task manifest system for configuration
- [ ] Build-time memory layout generation
- [ ] Panic handling in embedded context
- [ ] Serial console implementation

## Key Components

```
kernel/          # Core microkernel
├── src/
│   └── arch/    # Architecture-specific boot
task/           # System tasks
├── bsp/        # Board support packages
└── net/        # Network stack (reference)
build/          # Build system patterns
```

## Transferable Patterns
- Build-time configuration via TOML
- Memory safety without heap
- Minimal privileged code
- Clear task boundaries

## Not Directly Applicable
- ARM Cortex-M focus (we need x86_64)
- Task/IPC model (bootloader is simpler)
- Network stack (not needed in bootloader)
