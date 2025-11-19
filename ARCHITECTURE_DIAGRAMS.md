# V3000 Bootloader Architecture Diagrams

This document provides comprehensive visual documentation of the V3000 bootloader architecture, boot sequence, and component relationships.

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Hardware/Software Stack](#2-hardwaresoftware-stack)
3. [Boot Sequence Timeline](#3-boot-sequence-timeline)
4. [Component Relationships](#4-component-relationships)
5. [Memory Layout](#5-memory-layout)
6. [Build and Flash Pipeline](#6-build-and-flash-pipeline)
7. [Handoff Protocol Details](#7-handoff-protocol-details)
8. [Community Tools and Analysis Workflow](#8-community-tools-and-analysis-workflow)
9. [Reference Implementation Sources](#9-reference-implementation-sources)
10. [Future Architecture: AMD openSIL](#10-future-architecture-amd-opensil)
11. [Complete Development Workflow](#11-complete-development-workflow)

---

## 1. System Architecture Overview

### 1.1 Complete System Context

```mermaid
graph TB
    subgraph "Development Environment"
        SRC[Source Code<br/>Rust + Assembly]
        XTASK[cargo-xtask<br/>Build System]
        AHIB[amd-host-image-builder<br/>Flash Image Tool]
    end

    subgraph "Build Artifacts"
        PHBL[phbl.elf<br/>Bootloader Binary]
        BLOBS[AMD Firmware Blobs<br/>PSP, AGESA, SMU]
        APCB[APCB<br/>Platform Config]
        FLASH[Complete Flash Image<br/>16 MiB SPI]
    end

    subgraph "Target Hardware"
        SPI[SPI Flash<br/>On-board]
        PSP[Platform Security Processor<br/>ARM Cortex-A5]
        X86[x86_64 CPU<br/>Zen 3 Cores]
        RAM[DDR5 Memory<br/>Dual Channel]
        UART[UART Console<br/>16550A]
    end

    subgraph "Runtime Software"
        IPL[PSP IPL<br/>Initial Program Loader]
        ABL[ABL0-7<br/>AGESA Boot Loader]
        BOOT[V3000 Bootloader<br/>Rust phbl Port]
        UBOOT[U-Boot<br/>System Bootloader]
        LINUX[Linux Kernel<br/>Operating System]
    end

    SRC --> XTASK
    XTASK --> PHBL
    PHBL --> AHIB
    BLOBS --> AHIB
    APCB --> AHIB
    AHIB --> FLASH
    FLASH --> SPI

    SPI --> PSP
    PSP --> IPL
    IPL --> ABL
    ABL --> RAM
    ABL --> X86
    X86 --> BOOT
    BOOT --> UART
    BOOT --> UBOOT
    UBOOT --> LINUX

    classDef dev fill:#e1f5fe,stroke:#01579b
    classDef artifact fill:#fff3e0,stroke:#e65100
    classDef hw fill:#f3e5f5,stroke:#4a148c
    classDef runtime fill:#e8f5e9,stroke:#1b5e20

    class SRC,XTASK,AHIB dev
    class PHBL,BLOBS,APCB,FLASH artifact
    class SPI,PSP,X86,RAM,UART hw
    class IPL,ABL,BOOT,UBOOT,LINUX runtime
```

### 1.2 Oxide Project Relationships

```mermaid
graph LR
    subgraph "Oxide Repositories"
        EFS[amd-efs<br/>Library]
        AHIB[amd-host-image-builder<br/>Tool]
        PHBL[phbl<br/>Bootloader]
        HELIOS[helios<br/>OS Reference]
    end

    subgraph "V3000 Project"
        V3EFS[amd-efs fork<br/>+V3000 enum]
        V3AHIB[amd-host-image-builder fork<br/>+V3000 config]
        V3BOOT[v3000-bootloader<br/>phbl port]
    end

    subgraph "AMD Proprietary"
        PSP_FW[PSP Firmware<br/>EmbeddedPi]
        AGESA[AGESA Binary<br/>FP7r2]
        APCB_TPL[APCB Template<br/>Board Config]
    end

    EFS --> AHIB
    AHIB --> PHBL
    HELIOS -.->|protocol ref| PHBL

    EFS --> V3EFS
    AHIB --> V3AHIB
    PHBL --> V3BOOT
    HELIOS -.->|protocol ref| V3BOOT

    V3EFS --> V3AHIB
    PSP_FW --> V3AHIB
    AGESA --> V3AHIB
    APCB_TPL --> V3AHIB

    classDef oxide fill:#bbdefb,stroke:#1565c0
    classDef v3000 fill:#c8e6c9,stroke:#2e7d32
    classDef amd fill:#ffccbc,stroke:#bf360c

    class EFS,AHIB,PHBL,HELIOS oxide
    class V3EFS,V3AHIB,V3BOOT v3000
    class PSP_FW,AGESA,APCB_TPL amd
```

---

## 2. Hardware/Software Stack

### 2.1 Layered Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        APP[Linux User Applications]
    end

    subgraph "Operating System Layer"
        KERNEL[Linux Kernel]
        DRIVERS[Device Drivers]
    end

    subgraph "System Bootloader Layer"
        UBOOT[U-Boot]
        ENV[Environment Variables]
        CMD[Boot Commands]
    end

    subgraph "Platform Bootloader Layer"
        PHBL_BOOT[V3000 Bootloader<br/>phbl port]
        PAGE[Page Tables<br/>4-Level Paging]
        UART_DRV[UART Driver<br/>16550A]
        ELF[ELF Loader<br/>goblin]
        CPIO[CPIO Handler<br/>Archive]
    end

    subgraph "Firmware Layer"
        ABL[AGESA Boot Loader<br/>ABL0-7]
        AGESA[AGESA Binary<br/>Memory Init]
        SMU[SMU Firmware<br/>Power Management]
    end

    subgraph "Security Processor Layer"
        IPL[PSP Initial Program Loader]
        SECUR[Security Services<br/>fTPM, Keys]
        BOOT_ROM[PSP Boot ROM<br/>Immutable]
    end

    subgraph "Hardware Layer"
        CPU[Zen 3 CPU Cores<br/>x86_64]
        PSP_HW[PSP Hardware<br/>ARM Cortex-A5]
        MEM[DDR5 Controller<br/>Dual Channel]
        FCH[FCH<br/>UART, GPIO, SPI]
        FLASH[SPI Flash<br/>16 MiB]
    end

    APP --> KERNEL
    KERNEL --> DRIVERS
    DRIVERS --> UBOOT
    UBOOT --> PHBL_BOOT
    PHBL_BOOT --> PAGE
    PHBL_BOOT --> UART_DRV
    PHBL_BOOT --> ELF
    PHBL_BOOT --> CPIO
    PHBL_BOOT --> ABL
    ABL --> AGESA
    ABL --> SMU
    ABL --> IPL
    IPL --> SECUR
    IPL --> BOOT_ROM
    BOOT_ROM --> PSP_HW
    ABL --> CPU
    AGESA --> MEM
    UART_DRV --> FCH
    BOOT_ROM --> FLASH

    classDef app fill:#e3f2fd,stroke:#1565c0
    classDef os fill:#e8eaf6,stroke:#283593
    classDef sysboot fill:#f3e5f5,stroke:#6a1b9a
    classDef platboot fill:#e8f5e9,stroke:#2e7d32
    classDef fw fill:#fff8e1,stroke:#f57f17
    classDef psp fill:#fce4ec,stroke:#c2185b
    classDef hw fill:#efebe9,stroke:#4e342e

    class APP app
    class KERNEL,DRIVERS os
    class UBOOT,ENV,CMD sysboot
    class PHBL_BOOT,PAGE,UART_DRV,ELF,CPIO platboot
    class ABL,AGESA,SMU fw
    class IPL,SECUR,BOOT_ROM psp
    class CPU,PSP_HW,MEM,FCH,FLASH hw
```

### 2.2 Code Reusability by Layer

```mermaid
pie title Code Reusability in V3000 Port
    "Generic x86_64 (100% reuse)" : 60
    "Generic Rust (100% reuse)" : 20
    "Verify Values (likely same)" : 10
    "Platform-Specific (needs NDA)" : 10
```

---

## 3. Boot Sequence Timeline

### 3.1 Complete Boot Sequence with Timing

```mermaid
sequenceDiagram
    participant FLASH as SPI Flash
    participant PSP as PSP<br/>(ARM Cortex-A5)
    participant MEM as DDR5 Memory
    participant CPU as x86_64 CPU<br/>(Zen 3)
    participant BOOT as V3000 Bootloader
    participant UART as UART Console
    participant UBOOT as U-Boot

    Note over FLASH,UBOOT: Power-On Reset (t=0)

    rect rgb(255, 230, 230)
        Note over FLASH,PSP: Phase 1: PSP Boot (~50-100ms)
        FLASH->>PSP: Load PSP Boot ROM
        PSP->>PSP: Execute Boot ROM
        PSP->>FLASH: Search for ROMSIG (0x55AA55AA)
        FLASH->>PSP: Return EFS location
        PSP->>FLASH: Load IPL (Initial Program Loader)
        PSP->>PSP: Verify IPL signature
        PSP->>PSP: Execute IPL
    end

    rect rgb(255, 243, 224)
        Note over PSP,MEM: Phase 2: AGESA Boot (~200-500ms)
        PSP->>FLASH: Load ABL0
        PSP->>PSP: Execute ABL0
        loop ABL1-7 Stages
            PSP->>FLASH: Load ABLx
            PSP->>PSP: Execute ABLx
        end
        PSP->>FLASH: Load AGESA, APCB
        PSP->>MEM: Initialize DDR5 (2 channels)
        MEM-->>PSP: Memory ready
        PSP->>MEM: Write APOB (training results)
    end

    rect rgb(232, 245, 233)
        Note over PSP,CPU: Phase 3: x86 Release (~1ms)
        PSP->>FLASH: Load Reset Image (bootloader)
        PSP->>MEM: Copy bootloader to DRAM
        PSP->>CPU: Release from reset
        CPU->>CPU: Begin at reset vector<br/>(0x7FFEFFF0)
    end

    rect rgb(227, 242, 253)
        Note over CPU,UART: Phase 4: V3000 Bootloader (~50-100ms)
        CPU->>CPU: 16-bit Real Mode entry
        CPU->>CPU: Setup GDT
        CPU->>CPU: Switch to 32-bit Protected Mode
        CPU->>CPU: Enable PAE, set EFER.LME
        CPU->>CPU: Enable paging (CR0.PG)
        CPU->>CPU: Switch to 64-bit Long Mode
        CPU->>BOOT: Jump to Rust entry point
        BOOT->>UART: Initialize UART (0xFEDC9000)
        BOOT->>UART: Print banner
        BOOT->>BOOT: Build page tables<br/>(4-level, 1G/2M pages)
        BOOT->>MEM: Read CPIO archive
        BOOT->>BOOT: Decompress (zlib)
        BOOT->>BOOT: Parse ELF (U-Boot)
        BOOT->>MEM: Load U-Boot segments
        BOOT->>BOOT: Set bit 11 on kernel PTEs
    end

    rect rgb(243, 229, 245)
        Note over BOOT,UBOOT: Phase 5: Handoff (~1ms)
        BOOT->>BOOT: Prepare handoff state
        Note right of BOOT: RDI = CPIO paddr<br/>RSI = CPIO size<br/>CR3 = PML4 paddr
        BOOT->>UBOOT: Jump to entry point
        UBOOT->>UART: Print U-Boot banner
        UBOOT->>UBOOT: Initialize devices
    end

    Note over FLASH,UBOOT: Total boot time target: <1 second
```

### 3.2 Critical Handoff Points

```mermaid
graph LR
    subgraph "Handoff 1: PSP → ABL"
        PSP1[PSP IPL]
        ABL1[ABL0]
        PSP1 -->|"• SRAM initialized<br/>• Crypto ready<br/>• Flash access"| ABL1
    end

    subgraph "Handoff 2: ABL → x86"
        ABL2[ABL7]
        X86[x86 Reset]
        ABL2 -->|"• DRAM trained<br/>• Bootloader in RAM<br/>• APOB written"| X86
    end

    subgraph "Handoff 3: Bootloader → U-Boot"
        BOOT[V3000 Bootloader]
        UBOOT[U-Boot Entry]
        BOOT -->|"• 64-bit long mode<br/>• Paging enabled<br/>• Page tables ready<br/>• RDI/RSI set<br/>• Bit 11 marked"| UBOOT
    end

    classDef handoff fill:#fff9c4,stroke:#f9a825,stroke-width:2px

    class PSP1,ABL1,ABL2,X86,BOOT,UBOOT handoff
```

### 3.3 CPU Mode Transitions Detail

```mermaid
stateDiagram-v2
    [*] --> RealMode: Reset Vector<br/>0x7FFEFFF0

    RealMode: 16-bit Real Mode
    RealMode: • CS:IP addressing
    RealMode: • 1MB address space
    RealMode: • No protection

    ProtectedMode: 32-bit Protected Mode
    ProtectedMode: • GDT active
    ProtectedMode: • 4GB address space
    ProtectedMode: • Segmentation

    LongMode: 64-bit Long Mode
    LongMode: • 4-level paging
    LongMode: • 256TB virtual
    LongMode: • Flat segments

    RustEntry: Rust Entry Point
    RustEntry: • rust_main()
    RustEntry: • Full language features
    RustEntry: • Type safety

    RealMode --> ProtectedMode: Set PE bit (CR0.PE=1)
    ProtectedMode --> LongMode: Set LME+PG<br/>(EFER.LME=1, CR0.PG=1)
    LongMode --> RustEntry: Far jump to<br/>64-bit code
```

---

## 4. Component Relationships

### 4.1 phbl Internal Architecture

```mermaid
graph TB
    subgraph "Entry Points"
        RESET[reset.S<br/>16-bit entry]
        START[start.S<br/>Mode transitions]
    end

    subgraph "Core Modules"
        MAIN[main.rs<br/>Control flow]
        MMU[mmu.rs<br/>Page tables]
        MEM[mem.rs<br/>Memory types]
        LOADER[loader.rs<br/>ELF loading]
    end

    subgraph "Hardware Drivers"
        UART[uart.rs<br/>Serial console]
        IOMUX[iomux.rs<br/>GPIO config]
    end

    subgraph "Utilities"
        IDT[idt.rs<br/>Exceptions]
        ALLOC[allocator.rs<br/>Bump alloc]
    end

    subgraph "Platform Config"
        LD[phbl.ld<br/>Linker script]
        CONFIG[phbl.rs<br/>Constants]
    end

    RESET --> START
    START --> MAIN
    MAIN --> MMU
    MAIN --> LOADER
    MAIN --> UART
    MMU --> MEM
    MMU --> ALLOC
    LOADER --> MEM
    UART --> IOMUX
    MAIN --> IDT
    LD --> RESET
    CONFIG --> UART
    CONFIG --> MMU

    classDef entry fill:#ffcdd2,stroke:#c62828
    classDef core fill:#c8e6c9,stroke:#2e7d32
    classDef driver fill:#bbdefb,stroke:#1565c0
    classDef util fill:#fff9c4,stroke:#f9a825
    classDef config fill:#e1bee7,stroke:#7b1fa2

    class RESET,START entry
    class MAIN,MMU,MEM,LOADER core
    class UART,IOMUX driver
    class IDT,ALLOC util
    class LD,CONFIG config
```

### 4.2 Data Flow During Boot

```mermaid
flowchart LR
    subgraph "Flash Storage"
        EFS[EFS Structure]
        PSP_DIR[PSP Directory]
        BHD[BHD Directory]
        BOOT_IMG[Bootloader Image]
        CPIO_ARC[CPIO Archive<br/>Compressed]
    end

    subgraph "DRAM"
        BOOT_CODE[Bootloader Code<br/>0x100000]
        PAGE_TBL[Page Tables<br/>0x200000]
        CPIO_DEC[CPIO Decompressed<br/>0x400000]
        UBOOT_IMG[U-Boot Loaded<br/>High addresses]
        APOB[APOB<br/>Training results]
    end

    subgraph "CPU State"
        CR3[CR3 Register<br/>Page table root]
        RDI[RDI Register<br/>CPIO address]
        RSI[RSI Register<br/>CPIO size]
        RIP[RIP Register<br/>Entry point]
    end

    EFS --> PSP_DIR
    EFS --> BHD
    BHD --> BOOT_IMG
    BHD --> CPIO_ARC

    BOOT_IMG -->|PSP copies| BOOT_CODE
    CPIO_ARC -->|Bootloader decompresses| CPIO_DEC
    BOOT_CODE -->|Builds| PAGE_TBL
    CPIO_DEC -->|ELF load| UBOOT_IMG

    PAGE_TBL --> CR3
    CPIO_DEC --> RDI
    CPIO_DEC --> RSI
    UBOOT_IMG --> RIP
```

### 4.3 Dependency Graph

```mermaid
graph BT
    subgraph "Rust Crates"
        X86[x86 crate<br/>CPU instructions]
        GOBLIN[goblin<br/>ELF parsing]
        MINIZ[miniz_oxide<br/>Decompression]
        BITSTRUCT[bitstruct<br/>Bit fields]
        STATIC[static_assertions<br/>Compile checks]
    end

    subgraph "Core Libraries"
        CORE[core<br/>no_std basics]
        COMPILER_BUILTINS[compiler_builtins<br/>Intrinsics]
    end

    subgraph "External"
        AMD_EFS[amd-efs<br/>Flash structures]
        AMD_APCB[amd-apcb<br/>Config blocks]
    end

    subgraph "V3000 Bootloader"
        BOOT[v3000-bootloader]
    end

    BOOT --> X86
    BOOT --> GOBLIN
    BOOT --> MINIZ
    BOOT --> BITSTRUCT
    BOOT --> STATIC
    BOOT --> CORE
    BOOT --> COMPILER_BUILTINS

    X86 --> CORE
    GOBLIN --> CORE
    MINIZ --> CORE
    BITSTRUCT --> CORE

    AMD_EFS -.->|Build tool| BOOT
    AMD_APCB -.->|Build tool| BOOT
```

---

## 5. Memory Layout

### 5.1 Physical Memory Map

```mermaid
graph TB
    subgraph "Physical Address Space"
        direction TB

        LOW[0x0000_0000<br/>Low Memory]
        LEGACY[0x0010_0000<br/>Bootloader Load<br/>1 MiB]
        PAGE[0x0020_0000<br/>Page Tables<br/>512 KiB arena]
        CPIO[0x0040_0000<br/>CPIO Region<br/>128 MiB]
        FREE[0x0840_0000<br/>Free Memory]

        MMIO_START[0x8000_0000<br/>MMIO Start<br/>2 GiB]

        GPIO[0xFED8_0000<br/>GPIO/IO Mux]
        UART[0xFEDC_9000<br/>UART0]

        FLASH[0xFF00_0000<br/>Flash Window]
        RESET[0x7FFE_FFF0<br/>Reset Vector]
        TOP[0xFFFF_FFFF<br/>4 GiB]
    end

    LOW --- LEGACY
    LEGACY --- PAGE
    PAGE --- CPIO
    CPIO --- FREE
    FREE --- MMIO_START
    MMIO_START --- GPIO
    GPIO --- UART
    UART --- FLASH
    FLASH --- RESET
    RESET --- TOP

    classDef code fill:#c8e6c9,stroke:#2e7d32
    classDef data fill:#bbdefb,stroke:#1565c0
    classDef mmio fill:#ffccbc,stroke:#bf360c
    classDef flash fill:#fff9c4,stroke:#f9a825

    class LEGACY,PAGE code
    class CPIO,FREE data
    class MMIO_START,GPIO,UART mmio
    class FLASH,RESET flash
```

### 5.2 Virtual Address Mapping

```mermaid
graph LR
    subgraph "Virtual Addresses"
        V_LOW[0x0000_0000_0000_0000<br/>Identity Mapped<br/>Bootloader region]
        V_HOLE[Canonical Hole<br/>Not mapped]
        V_HIGH[0xFFFF_FF80_0000_0000<br/>Kernel Space<br/>Direct map]
    end

    subgraph "Physical Addresses"
        P_LOW[0x0000_0000<br/>Low 4 GiB]
        P_HIGH[0x0000_0000<br/>Same physical]
    end

    V_LOW -->|Identity<br/>VA = PA| P_LOW
    V_HIGH -->|Offset<br/>VA - base = PA| P_HIGH

    Note1[Bootloader uses<br/>identity mapping]
    Note2[Kernel uses<br/>high virtual addresses]
```

### 5.3 Page Table Structure

```mermaid
graph TD
    subgraph "4-Level Paging"
        PML4[PML4<br/>512 entries<br/>@ CR3]
        PML3[PML3/PDPT<br/>512 entries<br/>1 GiB per entry]
        PML2[PML2/PD<br/>512 entries<br/>2 MiB per entry]
        PML1[PML1/PT<br/>512 entries<br/>4 KiB per entry]
        PAGE[Physical Page<br/>4 KiB]
    end

    PML4 --> PML3
    PML3 --> PML2
    PML2 --> PML1
    PML1 --> PAGE

    subgraph "Optimization Strategy"
        OPT1[1 GiB pages<br/>When aligned]
        OPT2[2 MiB pages<br/>When possible]
        OPT3[4 KiB pages<br/>Remainder]
    end

    PML3 -.->|Direct| OPT1
    PML2 -.->|Direct| OPT2
    PML1 -.->|Via PT| OPT3

    subgraph "Bit 11 Protocol"
        BIT11[PTE bit 11 = 1<br/>Kernel owns page<br/>Bootloader transfers<br/>ownership]
    end

    PML1 --> BIT11
```

---

## 6. Build and Flash Pipeline

### 6.1 Build Process

```mermaid
flowchart TD
    subgraph "Source Code"
        RUST[Rust Source<br/>*.rs files]
        ASM[Assembly<br/>*.S files]
        LD[Linker Script<br/>phbl.ld]
        TOML[Config<br/>Cargo.toml]
    end

    subgraph "Build Tools"
        CARGO[cargo build<br/>--release]
        XTASK[cargo xtask<br/>Custom commands]
        RUSTC[rustc<br/>x86_64-unknown-none]
        LINK[GNU ld<br/>Linker]
    end

    subgraph "Intermediate"
        OBJ[Object Files<br/>*.o]
        RLIB[Rust Libraries<br/>*.rlib]
    end

    subgraph "Output"
        ELF[phbl.elf<br/>Executable]
        BIN[phbl.bin<br/>Raw binary]
    end

    RUST --> CARGO
    TOML --> CARGO
    CARGO --> RUSTC
    ASM --> RUSTC
    RUSTC --> OBJ
    RUSTC --> RLIB
    OBJ --> LINK
    RLIB --> LINK
    LD --> LINK
    LINK --> ELF
    ELF --> |objcopy| BIN

    XTASK --> CARGO
```

### 6.2 Flash Image Assembly

```mermaid
flowchart LR
    subgraph "Inputs"
        BOOT[phbl.bin<br/>Bootloader]
        PSP[PSP Firmware<br/>*.sbin]
        AGESA[AGESA Binary<br/>*.bin]
        SMU[SMU Firmware<br/>*.sbin]
        APCB[APCB<br/>Config block]
        CPIO[CPIO Archive<br/>Kernel + initrd]
    end

    subgraph "amd-host-image-builder"
        PARSE[Parse Config<br/>TOML]
        BUILD_EFS[Build EFS<br/>Structure]
        BUILD_PSP[Build PSP<br/>Directory]
        BUILD_BHD[Build BHD<br/>Directory]
        ASSEMBLE[Assemble<br/>Flash Image]
        CHECKSUM[Calculate<br/>Checksums]
    end

    subgraph "Output"
        FLASH[v3000-flash.bin<br/>16 MiB image]
    end

    BOOT --> PARSE
    PSP --> PARSE
    AGESA --> PARSE
    SMU --> PARSE
    APCB --> PARSE
    CPIO --> PARSE

    PARSE --> BUILD_EFS
    BUILD_EFS --> BUILD_PSP
    BUILD_EFS --> BUILD_BHD
    BUILD_PSP --> ASSEMBLE
    BUILD_BHD --> ASSEMBLE
    ASSEMBLE --> CHECKSUM
    CHECKSUM --> FLASH
```

### 6.3 Flash Layout Structure

```mermaid
graph TB
    subgraph "16 MiB SPI Flash"
        direction TB

        EFS_HDR[0x00_0000<br/>EFS Header]
        PSP_L1[0x02_0000<br/>PSP Directory L1]
        PSP_L2[0x04_0000<br/>PSP Directory L2]
        BHD_L1[0x06_0000<br/>BHD Directory L1]
        BHD_L2[0x08_0000<br/>BHD Directory L2]

        PSP_FW[0x10_0000<br/>PSP Firmware Blobs]
        AGESA_BL[0x40_0000<br/>AGESA/ABL Blobs]

        APCB_A[0x80_0000<br/>APCB Primary]
        APCB_B[0x84_0000<br/>APCB Backup]

        APOB[0x90_0000<br/>APOB<br/>(Written by PSP)]

        BOOT_A[0xA0_0000<br/>Bootloader A]
        BOOT_B[0xC0_0000<br/>Bootloader B<br/>(Redundancy)]

        CPIO_A[0xE0_0000<br/>CPIO Archive]

        TOP[0xFF_FFFF<br/>End of Flash]
    end

    EFS_HDR --- PSP_L1
    PSP_L1 --- PSP_L2
    PSP_L2 --- BHD_L1
    BHD_L1 --- BHD_L2
    BHD_L2 --- PSP_FW
    PSP_FW --- AGESA_BL
    AGESA_BL --- APCB_A
    APCB_A --- APCB_B
    APCB_B --- APOB
    APOB --- BOOT_A
    BOOT_A --- BOOT_B
    BOOT_B --- CPIO_A
    CPIO_A --- TOP

    classDef dir fill:#e1bee7,stroke:#7b1fa2
    classDef fw fill:#ffccbc,stroke:#bf360c
    classDef config fill:#fff9c4,stroke:#f9a825
    classDef boot fill:#c8e6c9,stroke:#2e7d32
    classDef data fill:#bbdefb,stroke:#1565c0

    class EFS_HDR,PSP_L1,PSP_L2,BHD_L1,BHD_L2 dir
    class PSP_FW,AGESA_BL fw
    class APCB_A,APCB_B,APOB config
    class BOOT_A,BOOT_B boot
    class CPIO_A data
```

---

## 7. Handoff Protocol Details

### 7.1 Bootloader to Kernel/U-Boot Handoff

```mermaid
sequenceDiagram
    participant BOOT as V3000 Bootloader
    participant REG as CPU Registers
    participant MEM as Memory
    participant UBOOT as U-Boot

    Note over BOOT,UBOOT: Handoff Preparation

    BOOT->>MEM: Finalize page tables
    BOOT->>MEM: Set bit 11 on kernel PTEs

    BOOT->>REG: Load CR3 with PML4 paddr
    BOOT->>REG: RDI = CPIO physical address
    BOOT->>REG: RSI = CPIO size in bytes
    BOOT->>REG: RDX = 0 (reserved)
    BOOT->>REG: RCX = 0 (reserved)
    BOOT->>REG: R8 = 0 (reserved)
    BOOT->>REG: R9 = 0 (reserved)

    Note over REG: CPU State Guarantees
    Note right of REG: • 64-bit long mode (EFER.LME=1)<br/>• Paging enabled (CR0.PG=1)<br/>• Write protect (CR0.WP=1)<br/>• Interrupts disabled (RFLAGS.IF=0)<br/>• Stack: 32 KiB, 16-byte aligned<br/>• All exceptions have handlers

    BOOT->>UBOOT: JMP to entry point

    UBOOT->>REG: Read RDI, RSI
    UBOOT->>MEM: Access CPIO archive
    UBOOT->>UBOOT: Continue boot
```

### 7.2 Register State at Handoff

```mermaid
graph LR
    subgraph "General Purpose Registers"
        RAX[RAX: Undefined]
        RBX[RBX: Undefined]
        RCX[RCX: 0]
        RDX[RDX: 0]
        RSI[RSI: CPIO size]
        RDI[RDI: CPIO paddr]
        RBP[RBP: Undefined]
        RSP[RSP: Stack top]
        R8[R8-R15: 0]
    end

    subgraph "Control Registers"
        CR0[CR0: PG+WP+PE]
        CR3[CR3: PML4 paddr]
        CR4[CR4: PAE+...]
    end

    subgraph "Model Specific Registers"
        EFER[EFER: LME+LMA]
    end

    subgraph "Flags"
        RFLAGS[RFLAGS: IF=0]
    end

    classDef param fill:#c8e6c9,stroke:#2e7d32
    classDef zero fill:#e0e0e0,stroke:#616161
    classDef ctrl fill:#bbdefb,stroke:#1565c0

    class RDI,RSI param
    class RCX,RDX,R8 zero
    class CR0,CR3,CR4,EFER ctrl
```

### 7.3 Page Table Entry with Bit 11

```mermaid
graph LR
    subgraph "64-bit Page Table Entry"
        B0[0: Present]
        B1[1: R/W]
        B2[2: U/S]
        B3[3: PWT]
        B4[4: PCD]
        B5[5: Accessed]
        B6[6: Dirty]
        B7[7: PS/PAT]
        B8[8: Global]
        B9_10[9-10: Available]
        B11[11: Kernel Owned<br/>CRITICAL]
        B12_51[12-51: Physical Address]
        B52_62[52-62: Available]
        B63[63: NX]
    end

    B11 --> NOTE[When bit 11 = 1:<br/>• Page belongs to kernel<br/>• Bootloader transfers ownership<br/>• Kernel won't reallocate]

    classDef critical fill:#ffcdd2,stroke:#c62828,stroke-width:3px
    class B11 critical
```

---

## Summary

These diagrams illustrate the complete V3000 bootloader architecture:

1. **System Architecture**: Shows all components and their relationships
2. **Hardware/Software Stack**: Layered view from hardware to applications
3. **Boot Sequence**: Timeline with timing estimates and handoff points
4. **Component Relationships**: Internal phbl structure and dependencies
5. **Memory Layout**: Physical and virtual address space organization
6. **Build Pipeline**: From source to flash image
7. **Handoff Protocol**: Detailed register and memory state requirements

Key takeaways:
- **80% of code is generic x86_64** - reusable without modification
- **Critical handoff points** are well-defined with clear state requirements
- **Bit 11 protocol** is essential for kernel page ownership transfer
- **Total boot time target** is under 1 second
- **Extension points** are clearly defined for adding V3000 support

---

## 8. Community Tools and Analysis Workflow

### 8.1 Firmware Analysis Pipeline

```mermaid
flowchart LR
    subgraph "Input Sources"
        VENDOR[Vendor BIOS<br/>SolidRun]
        EXISTING[Existing Flash<br/>Dump from board]
    end

    subgraph "Analysis Tools"
        PSPTOOL[PSPTool<br/>Directory analysis]
        PSPEMU[PSPEmu<br/>PSP emulation]
        SPLOADER[AMD-SP-Loader<br/>Binary Ninja]
    end

    subgraph "Extracted Data"
        PSP_DIR[PSP Directory<br/>Entry types]
        APCB_DATA[APCB Structure<br/>Tokens, values]
        ADDR[Addresses<br/>UART, GPIO, Memory]
        GEN_ID[Generation ID<br/>Detection bits]
    end

    subgraph "V3000 Config"
        CONFIG[v3000-config.toml<br/>Platform definition]
        CPUID[CPUID Pattern<br/>0x19/0x40-0x4F]
        PINS[Pin Configuration<br/>UART mux values]
    end

    VENDOR --> PSPTOOL
    EXISTING --> PSPTOOL

    PSPTOOL --> PSP_DIR
    PSPTOOL --> APCB_DATA
    PSPTOOL --> ADDR
    PSPTOOL --> GEN_ID

    PSPTOOL --> PSPEMU
    PSPEMU --> SPLOADER

    PSP_DIR --> CONFIG
    APCB_DATA --> CONFIG
    ADDR --> PINS
    GEN_ID --> CPUID

    classDef tool fill:#bbdefb,stroke:#1565c0
    classDef data fill:#fff9c4,stroke:#f9a825
    classDef output fill:#c8e6c9,stroke:#2e7d32

    class PSPTOOL,PSPEMU,SPLOADER tool
    class PSP_DIR,APCB_DATA,ADDR,GEN_ID data
    class CONFIG,CPUID,PINS output
```

### 8.2 Processor Generation Detection Flow

```mermaid
flowchart TD
    START[CPUID Instruction] --> READ[Read Family/Model]

    READ --> CHECK{Match Pattern}

    CHECK -->|0x17, 0x00-0x0F| NAPLES[Naples<br/>EPYC 7001]
    CHECK -->|0x17, 0x30-0x3F| ROME[Rome<br/>EPYC 7002]
    CHECK -->|0x19, 0x00-0x0F| MILAN[Milan<br/>EPYC 7003<br/>bits: 0xfc]
    CHECK -->|0x19, 0x10-0x1F| GENOA[Genoa<br/>EPYC 9004<br/>bits: 0xfe]
    CHECK -->|0x1A, 0x00-0x1F| TURIN[Turin<br/>EPYC 9005<br/>bits: 0xe3]
    CHECK -->|0x19, 0x40-0x4F| V3000[V3000<br/>Rembrandt<br/>bits: TBD]
    CHECK -->|Other| UNKNOWN[Unknown<br/>Processor]

    NAPLES --> GPIO_CFG[Load GPIO Config]
    ROME --> GPIO_CFG
    MILAN --> GPIO_CFG
    GENOA --> GPIO_CFG
    TURIN --> GPIO_CFG
    V3000 --> GPIO_CFG

    GPIO_CFG --> UART_INIT[Initialize UART<br/>Set Pin Mux]

    classDef known fill:#c8e6c9,stroke:#2e7d32
    classDef new fill:#fff9c4,stroke:#f9a825
    classDef unknown fill:#ffcdd2,stroke:#c62828

    class NAPLES,ROME,MILAN,GENOA,TURIN known
    class V3000 new
    class UNKNOWN unknown
```

### 8.3 Pin Mux Configuration (Critical)

```mermaid
sequenceDiagram
    participant BOOT as Bootloader
    participant CPUID as CPUID Check
    participant GPIO as GPIO Registers
    participant MUX as IO Mux
    participant UART as UART

    Note over BOOT,UART: Pin Mux Must Be Explicit<br/>(Cannot Trust Reset Defaults)

    BOOT->>CPUID: Get processor family/model
    CPUID-->>BOOT: (0x19, 0x40-0x4F)

    BOOT->>BOOT: Select V3000 pin config

    loop For each UART pin
        BOOT->>GPIO: Read GPIO base (0xFED80000)
        BOOT->>MUX: Calculate mux offset (+0x0D00)
        BOOT->>MUX: Set function (F0 = UART)
    end

    Note right of BOOT: Pins 135-138 (verify)<br/>CTS, RXD, RTS, TXD

    BOOT->>UART: Initialize 16550
    UART-->>BOOT: Ready

    Note over BOOT,UART: Issue phbl#48: Genoa pins<br/>defaulted to GPIO, not UART
```

---

## 9. Reference Implementation Sources

### 9.1 Code Reference Hierarchy

```mermaid
graph TB
    subgraph "Primary Reference"
        PHBL[Oxide phbl<br/>Direct port source]
    end

    subgraph "Architecture Reference"
        COREBOOT[Coreboot Rembrandt<br/>src/soc/amd/rembrandt]
        MENDOCINO[Coreboot Mendocino<br/>Derived from Rembrandt]
    end

    subgraph "Integration Reference"
        THREMDEB[3mdeb V1000/R1000<br/>AGESA+EDK2 work]
        AHIB[amd-host-image-builder<br/>Image creation]
    end

    subgraph "Analysis Reference"
        PSPREVERSE[PSPReverse<br/>Firmware analysis]
        AMDBLOBS[amd/firmware_binaries<br/>Reference blobs]
    end

    subgraph "V3000 Bootloader"
        V3BOOT[v3000-bootloader]
    end

    PHBL -->|Core code| V3BOOT
    COREBOOT -->|FCH init patterns| V3BOOT
    THREMDEB -->|AGESA integration| V3BOOT
    AHIB -->|Image building| V3BOOT
    PSPREVERSE -->|Firmware analysis| V3BOOT

    classDef primary fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
    classDef reference fill:#bbdefb,stroke:#1565c0
    classDef analysis fill:#fff9c4,stroke:#f9a825

    class PHBL primary
    class COREBOOT,MENDOCINO,THREMDEB,AHIB reference
    class PSPREVERSE,AMDBLOBS analysis
```

### 9.2 APCB Structure Variations

```mermaid
graph TB
    subgraph "APCB Type ID"
        TYPE[Same Type ID<br/>e.g., DimmInfoSmbusElement]
    end

    subgraph "Structure Variations"
        MILAN_S[Milan Structure<br/>Old fields]
        GENOA_S[Genoa/Turin Structure<br/>New fields:<br/>- Socket/Channel/DIMM<br/>- BusID ranges<br/>- I2C/SMBUS/I3C]
        V3000_S[V3000 Structure<br/>TBD - may differ]
    end

    TYPE --> MILAN_S
    TYPE --> GENOA_S
    TYPE --> V3000_S

    Note1[Issue amd-apcb#157:<br/>AMD reuses Type IDs<br/>with different layouts]

    classDef warning fill:#fff9c4,stroke:#f9a825,stroke-width:2px
    class TYPE,MILAN_S,GENOA_S,V3000_S warning
```

---

## 10. Future Architecture: AMD openSIL

### 10.1 Timeline and Roadmap

```mermaid
gantt
    title AMD openSIL Roadmap
    dateFormat YYYY
    axisFormat %Y

    section Proof of Concept
    Genoa POC           :done, 2022, 2023
    Turin/Phoenix POC   :done, 2023, 2024

    section Code Release
    Phoenix Code        :active, 2024, 2025
    Turin Code          :2025, 2026

    section Production
    Venice (Zen 6)      :2026, 2027
    Medusa (Zen 6)      :2027, 2028

    section V3000 Project
    V3000 (Zen 3)       :milestone, 2024, 2025
```

### 10.2 V3000 Strategy vs openSIL

```mermaid
graph LR
    subgraph "Current: V3000 (Zen 3)"
        V3_AGESA[Binary AGESA<br/>EmbeddedPi FP7r2]
        V3_PHBL[phbl-style<br/>Integration]
    end

    subgraph "Future: Zen 6+"
        OPENSIL[AMD openSIL<br/>Open Source]
        FUTURE[Future Projects<br/>Direct integration]
    end

    V3_AGESA --> V3_PHBL
    OPENSIL --> FUTURE

    Note1[V3000 uses binary AGESA<br/>openSIL supports Zen 6+]

    classDef current fill:#c8e6c9,stroke:#2e7d32
    classDef future fill:#e1bee7,stroke:#7b1fa2

    class V3_AGESA,V3_PHBL current
    class OPENSIL,FUTURE future
```

---

## 11. Complete Development Workflow

### 11.1 From Analysis to Hardware Boot

```mermaid
flowchart TB
    subgraph "Phase 1: Analysis"
        DUMP[Dump V3000 Firmware]
        ANALYZE[Analyze with PSPTool]
        EXTRACT[Extract Addresses<br/>& Configuration]
    end

    subgraph "Phase 2: Development"
        FORK[Fork Oxide Repos]
        CONFIG[Create V3000 Config]
        BUILD[Build Bootloader]
        QEMU[Test in QEMU]
    end

    subgraph "Phase 3: Integration"
        BLOBS[Obtain AMD Blobs<br/>PSP, AGESA, APCB]
        IMAGE[Build Flash Image]
        FLASH[Flash to SolidRun]
    end

    subgraph "Phase 4: Validation"
        SERIAL[Serial Console Debug]
        ITERATE[Iterate & Fix]
        BOOT[Boot to U-Boot]
    end

    DUMP --> ANALYZE
    ANALYZE --> EXTRACT
    EXTRACT --> CONFIG

    FORK --> CONFIG
    CONFIG --> BUILD
    BUILD --> QEMU

    QEMU --> IMAGE
    BLOBS --> IMAGE
    IMAGE --> FLASH

    FLASH --> SERIAL
    SERIAL --> ITERATE
    ITERATE --> BOOT

    classDef analysis fill:#e3f2fd,stroke:#1565c0
    classDef dev fill:#e8f5e9,stroke:#2e7d32
    classDef integrate fill:#fff3e0,stroke:#e65100
    classDef validate fill:#f3e5f5,stroke:#6a1b9a

    class DUMP,ANALYZE,EXTRACT analysis
    class FORK,CONFIG,BUILD,QEMU dev
    class BLOBS,IMAGE,FLASH integrate
    class SERIAL,ITERATE,BOOT validate
```

---

## Summary

These diagrams now include:

1. **Original diagrams** (sections 1-7) - System architecture, boot sequence, memory layout
2. **Analysis workflow** (section 8) - PSPTool integration, CPUID detection, pin mux
3. **Reference sources** (section 9) - Coreboot Rembrandt, 3mdeb, PSPReverse
4. **Future architecture** (section 10) - openSIL timeline and strategy
5. **Complete workflow** (section 11) - End-to-end development process

Key insights integrated from community research:
- **Explicit pin mux required** (issue phbl#48)
- **Processor generation detection** with specific bits
- **APCB structure variations** between generations
- **PSPTool as primary analysis tool**
- **Coreboot Rembrandt as architecture reference**
- **openSIL future path** (V3000 uses binary AGESA)
