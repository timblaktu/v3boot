# COMPREHENSIVE PHBL (Pico Host Boot Loader) RESEARCH REPORT
## Oxide Computer's AMD Platform Bootloader - Porting Guide for V3000 Series

**Repository**: https://github.com/oxidecomputer/phbl  
**Language**: Rust (92.6%), Assembly (6.2%), Linker Script (1.2%)  
**License**: Mozilla Public License 2.0  
**Key Reference**: RFD 284

---

## EXECUTIVE SUMMARY

PHBL is a "load bearing" bootloader executing from the x86-64 reset vector (0x000000007FFEF000) that:
1. Transitions from 16-bit real mode → 32-bit protected mode → 64-bit long mode
2. Decompresses ZLIB-compressed phase1 CPIO archive into physical RAM
3. Parses ELF kernel image and loads segments to linked addresses
4. Invokes kernel entry point with specific register state guarantees

The bootloader is optimized for Oxide's specific hardware (Milan, Genoa, Bergamo, Turin EPYC processors) and uses sophisticated type-driven page table construction to prevent common memory safety errors.

---

## 1. SOURCE CODE ARCHITECTURE

### 1.1 Entry Point & Control Flow

**Assembly Entry Point**: `/tmp/phbl/src/start.S` (lines 1-265)

**Reset Vector Handler** (lines 67-80):
```assembly
.section ".reset", "a", @progbits
.globl reset
reset:
    cli                    # Clear interrupts
    cld                    # Clear direction flag
    jmp start
    ud2
    .balign 16, 0xff
```

**Physical Location**: `bootblock = 0x000000007FFEF000` (from phbl.ld line 6)  
**Reset Vector Address**: `resetaddr = 0x000000007FFEFFF0` (from phbl.ld line 7)

### 1.2 16-bit Real Mode Initialization (start.S lines 89-174)

**Purpose**: Establish GDT and transition to 32-bit protected mode

**Key Operations**:
1. **Save BIST Data** (line 94):
   ```assembly
   movl %eax, %ebp    # Save BIST value for later validation
   ```

2. **Clear Cache Inhibit Bits** (lines 100-101):
   ```assembly
   movl $CR0_MB1, %eax        # Set only reserved bits (CR0_ET)
   movl %eax, %cr0            # Clear CD/NW bits for writeback
   ```

3. **Load GDT** (lines 113-115):
   - GDT descriptor located at offset within current segment
   - GDT contains:
     - Index 0: Null segment
     - Index 1 (GDT_CODE64 = 0x08): 64-bit code segment
     - Index 2 (GDT_CODE32 = 0x10): 32-bit code segment
     - Index 3 (GDT_DATA32 = 0x18): 32-bit data segment

4. **Enable Protected Mode** (lines 118-120):
   ```assembly
   movl %cr0, %eax
   orl $CR0_PE, %eax         # Set PE bit (bit 0)
   movl %eax, %cr0
   ```

5. **Jump to 32-bit Code** (line 123):
   ```assembly
   ljmpl $GDT_CODE32, $1f    # Long jump to 32-bit handler
   ```

### 1.3 32-bit Protected Mode (start.S lines 125-173)

**Objective**: Configure MMU prerequisites and enter long mode

**Segment Setup** (lines 132-135):
```assembly
movw $GDT_DATA32, %ax
movw %ax, %ds               # Data segment
movw %ax, %es               # Extra segment
movw %ax, %ss               # Stack segment
```

**Memory Type Range Register (MTRR) Configuration** (lines 143-146):
```assembly
movl $IA32_MTRR_DEF_TYPE_MSR, %ecx
movl $(MTRR_ENABLE | MTRR_WB), %eax    # Enable MTRRs, default to writeback
xorl %edx, %edx
wrmsr
```
- **IA32_MTRR_DEF_TYPE MSR**: 0x2FF
- **MTRR_ENABLE**: Bit 11 (enables MTRR usage)
- **MTRR_WB**: 0x06 (writeback cache type)

**Physical Address Extension (PAE)** (lines 149-151):
```assembly
movl %cr4, %eax
orl $CR4_PAE, %eax         # Set PAE bit (bit 5) for 4-level paging
movl %eax, %cr4
```

**Page Table Root Pointer** (lines 154-155):
```assembly
movl $pml4, %eax
movl %eax, %cr3            # Load PML4 base address
```

**Long Mode & NX Support** (lines 159-162):
```assembly
movl $IA32_EFER_MSR, %ecx  # 0xc0000080
movl $(EFER_LME | EFER_NX), %eax
xorl %edx, %edx
wrmsr
```
- **EFER_LME**: Bit 8 (Long Mode Enable)
- **EFER_NX**: Bit 11 (Enable NX bit in page tables)

**Enable Paging & Write Protection** (lines 168-170):
```assembly
movl %cr0, %eax
orl $(CR0_PG | CR0_WP), %eax
movl %eax, %cr0
```
- **CR0_PG**: Bit 31 (Paging Enable)
- **CR0_WP**: Bit 16 (Write Protect - enforces kernel write-protection)

**Jump to 64-bit Code** (line 173):
```assembly
ljmpl $GDT_CODE64, $start64
```

### 1.4 64-bit Long Mode Initialization (start.S lines 197-228)

**Segment Register Clearing** (lines 202-205):
```assembly
xorl %eax, %eax
movw %ax, %ds
movw %ax, %es
movw %ax, %ss
```

**BSS Zeroing** (lines 208-212):
```assembly
movq $ebss, %rcx
movq $sbss, %rdi
xorl %eax, %eax
subq %rdi, %rcx
rep; stosb                 # Bulk zero from sbss to ebss
```

**Stack Setup** (lines 215-216):
```assembly
movq $stack, %rsp
addq $STACK_SIZE, %rsp     # Stack grows downward, RSP points to top
```
Stack size: `8 * PAGE_SIZE = 32 KiB` (from start.S line 64)

**Rust Entry Point Call** (lines 223-227):
```assembly
movl %ebp, %edi            # Pass BIST value as first argument (rdi register)
xorl %ebp, %ebp            # Clear RBP for stack frame termination
call init                  # Call Rust init() function
movq %rax, %rdi            # Pass Config reference to entry()
call entry                 # Call main Rust entry point
```

### 1.5 Early Page Table Setup (start.S lines 239-265)

**Initial PML4** (lines 247-249):
```assembly
.rodata
pml4:
    .quad pml3 + (PG_R | PG_W | PG_X)  # Entry points to PML3
    .space PAGE_SIZE - 8
```

**Initial PML3** (lines 251-256):
```assembly
pml3:
    .quad 0                            # Entry 0: unmapped (kernel space)
    .quad (1 << 30) + (PG_HUGE | PG_R | PG_W | PG_X)     # Entry 1: 1GiB@0x40000000 mapped
    .quad 0                            # Entry 2: unmapped
    .quad (3 << 30) + (PG_HUGE | PG_R | PG_W | PG_NX | PG_NC | PG_WT) # Entry 3: MMIO@0xC0000000
    .space PAGE_SIZE - 4 * 8
```

**Purpose**: Provides temporary identity mapping for:
- Bootloader code (2GiB region)
- MMIO registers uncached (4GiB region starting at 0x8000_0000)

### 1.6 Rust Initialization Flow

**File**: `/tmp/phbl/src/phbl.rs`

**init() Function** (lines 81-106):
```rust
pub(crate) unsafe extern "C" fn init(bist: u32) -> &'static mut Config {
    static INITED: AtomicBool = AtomicBool::new(false);
    if INITED.swap(true, Ordering::AcqRel) {
        panic!("Init already called");
    }
    
    unsafe {
        iomux::init();      // Configure IO mux for UART pins
        uart::init();       // Initialize UART0
    }
    idt::init();            // Set up exception handlers
    
    if bist != 0 {
        panic!("bist failed: {bist:#x}");  // Validate built-in self-test
    }
    
    let cons = Uart::uart0();
    let page_table = remap(mem::V4KA::new(cons.addr()));
    
    // Identify reserved memory regions
    let cpio_region = cpio_addr()..saddr();
    let loader_region = saddr()..eaddr();
    let mmio_region = mmio_addr()..mmio_end();
    
    let config = Box::new(Config {
        cons,
        loader_region,
        page_table: mmu::LoaderPageTable::new(page_table, &reserved_regions),
    });
    Box::leak(config)  // Return 'static reference
}
```

**Key Config Structure** (lines 56-60):
```rust
pub(crate) struct Config {
    pub(crate) cons: Uart,              // Console UART handle
    pub(crate) loader_region: Range<mem::V4KA>,  // Bootloader bounds
    pub(crate) page_table: mmu::LoaderPageTable, // Active page table
}
```

**entry() Function** (main.rs lines 24-35):
```rust
pub(crate) extern "C" fn entry(config: &mut phbl::Config) {
    println!();
    println!("Oxide Pico Host Boot Loader");
    println!("{config:#x?}");
    
    let ramdisk = expand_ramdisk();     // Decompress CPIO archive
    let kernel = find_kernel(ramdisk);   // Locate kernel ELF in archive
    let entry = loader::load(&mut config.page_table, kernel)
        .expect("loaded kernel");
    
    println!("jumping into kernel...");
    entry(ramdisk.as_ptr().addr() as u64, ramdisk.len());  // Call kernel
    panic!("main returning");
}
```

### 1.7 Memory Addresses (from phbl.rs)

**CPIO Archive Region**:
```rust
const CPIO_LEN: usize = 128 * mem::MIB;
fn cpio_addr() -> mem::V4KA {
    mem::V4KA::new(saddr().addr() - CPIO_LEN)  // 128 MiB below loader
}
```

**MMIO Regions**:
```rust
fn mmio_addr() -> mem::V4KA {
    mem::V4KA::new(0x8000_0000)  // MMIO starts at 2GiB
}
fn mmio_end() -> mem::V4KA {
    mem::V4KA::new(0x1_0000_0000)  // MMIO ends at 4GiB
}
```

---

## 2. MEMORY MANAGEMENT

### 2.1 Address Space Types (mem.rs)

**Virtual Address Type - V4KA** (lines 11-68):
```rust
pub(crate) struct V4KA(usize);  // 4KiB-aligned virtual address

impl V4KA {
    pub(crate) const ALIGN: usize = 4096;
    pub(crate) const SIZE: usize = 4096;
    
    pub(crate) const fn new(va: usize) -> V4KA {
        assert!(is_canonical(va));
        assert!(va & Self::MASK == 0);
        V4KA(va)
    }
}
```

**Physical Address Type - P4KA** (lines 70-93):
```rust
pub(crate) struct P4KA(u64);  // 4KiB-aligned physical address

impl P4KA {
    pub(crate) const ALIGN: u64 = 4096;
    
    pub(crate) const fn new(pa: u64) -> P4KA {
        assert!(is_physical(pa));
        assert!(pa & Self::MASK == 0);
        P4KA(pa)
    }
}
```

**Canonical Address Space**:
```rust
pub const LOW_CANON_SUP: usize = 0x0000_7FFF_FFFF_FFFF + 1;
pub const HI_CANON_INF: usize = 0xFFFF_8000_0000_0000 - 1;

const fn is_canonical(va: usize) -> bool {
    va <= 0x0000_7FFF_FFFF_FFFF || 0xFFFF_8000_0000_0000 <= va
}
```

### 2.2 Page Attributes (mem.rs lines 95-180)

**Attributes Structure**:
```rust
pub(crate) struct Attrs {
    r: bool,  // Readable
    w: bool,  // Writable
    x: bool,  // Executable
    c: bool,  // Cacheable
    k: bool,  // Kernel nucleus (bit 11 - RFD 215 contract)
}
```

**Predefined Attribute Sets**:
- `new_text()`: r=true, w=false, x=true, c=true, k=false
- `new_rodata()`: r=true, w=false, x=false, c=true, k=false
- `new_data()`: r=true, w=true, x=false, c=true, k=false
- `new_bss()`: r=true, w=true, x=false, c=true, k=false
- `new_mmio()`: r=true, w=true, x=false, c=false, k=false (uncached)
- `new_kernel()`: r/w/x configurable, c=true, k=true (bit 11 set)

### 2.3 Page Table Entry (PTE) Structure (mmu.rs lines 357-445)

**Bitstruct Definition** (lines 357-388):
```rust
bitstruct! {
    struct PTE(u64) {
        p: bool = 0;           // Present
        w: bool = 1;           // Writable
        u: bool = 2;           // User/Supervisor
        wt: bool = 3;          // Write-Through
        nc: bool = 4;          // Cache-Disabled (No-Cache)
        // a: bool = 5;         // Accessed (unused)
        // d: bool = 6;         // Dirty (unused)
        h: bool = 7;           // Huge page (Large/Huge)
        // g: bool = 8;         // Global (unused)
        // ign: u8 = 9..12;     // Ignored bits
        k: bool = 11;          // Kernel nucleus ownership (RFD 215)
        pfn: u64 = 12..51;     // Physical Frame Number
        // ign: u8 = 51..63;    // Ignored bits
        nx: bool = 63;         // No-Execute
    }
}
```

**Bit 11 Ownership Protocol** (from RFD 215, referenced in mmu.rs lines 99-101):
- Set on leaf PTEs for pages containing kernel nucleus image
- Used by host OS to determine memory ownership
- Must be set on all mappings of kernel code/data segments

**PTE Construction** (lines 406-415):
```rust
fn new<F: Frame>(pa: F, attrs: mem::Attrs) -> PTE {
    PTE::from_phys_addr(pa.phys_addr())
        .with_p(attrs.r())           // Present = Readable
        .with_w(attrs.w())           // Writable
        .with_nx(!attrs.x())         // NX = Not Executable
        .with_wt(!attrs.c())         // Write-Through if not cached
        .with_nc(!attrs.c())         // No-Cache if not cached
        .with_k(attrs.k())           // Kernel nucleus bit 11
        .with_h(F::BIG)              // Large/Huge for 2M/1G pages
}
```

### 2.4 Page Size Hierarchy (mmu.rs)

**Frame Types**:
```rust
struct PFN4K(u64);   // 4KiB frame (SIZE = 1 << 12)
struct PFN2M(u64);   // 2MiB frame (SIZE = 1 << 21, BIG = true)
struct PFN1G(u64);   // 1GiB frame (SIZE = 1 << 30, BIG = true)
```

**Page Table Levels**:
- **PML4** (top): Points to PML3 entries
- **PML3** (level 3): Can map 1GiB huge pages or point to PML2
- **PML2** (level 2): Can map 2MiB large pages or point to PML1
- **PML1** (level 1): Maps only 4KiB pages

**Table Index Shifts**:
```rust
impl Table for PML4 {
    const INDEX_SHIFT: usize = 39;  // Bits 39-47 select entry
}
impl Table for PML3 {
    const INDEX_SHIFT: usize = 30;  // Bits 30-38 select entry
}
impl Table for PML2 {
    const INDEX_SHIFT: usize = 21;  // Bits 21-29 select entry
}
impl Table for PML1 {
    const INDEX_SHIFT: usize = 12;  // Bits 12-20 select entry
}
```

### 2.5 Page Table Construction (mmu.rs lines 837-903)

**Identity Mapping Algorithm**:
```rust
pub(crate) unsafe fn identity_map(&mut self, regions: &[mem::Region]) {
    for region in regions {
        let pa = mem::P4KA::new(region.start().addr() as u64);
        unsafe {
            self.map_region(region, pa);
        }
    }
}
```

**Region Mapping** (lines 850-890):
```rust
unsafe fn map_region(&mut self, region: &mem::Region, pa: mem::P4KA) {
    let mut start = region.start().addr();
    let end = region.end().addr();
    let mut pa = pa.phys_addr();
    
    while start != end {
        let attrs = region.attrs();
        let len = if end.wrapping_sub(start) >= PFN1G::SIZE
            && start.is_multiple_of(PFN1G::SIZE)
            && (pa as usize).is_multiple_of(PFN1G::SIZE)
        {
            // Use 1GiB huge page
            unsafe {
                self.map(Page1G::new(start), PFN1G::new(pa), attrs);
            }
            PFN1G::SIZE
        } else if end.wrapping_sub(start) >= PFN2M::SIZE
            && start.is_multiple_of(PFN2M::SIZE)
            && (pa as usize).is_multiple_of(PFN2M::SIZE)
        {
            // Use 2MiB large page
            unsafe {
                self.map(Page2M::new(start), PFN2M::new(pa), attrs);
            }
            PFN2M::SIZE
        } else if end.wrapping_sub(start) >= PFN4K::SIZE
            && start.is_multiple_of(PFN4K::SIZE)
            && (pa as usize).is_multiple_of(PFN4K::SIZE)
        {
            // Use 4KiB page
            unsafe {
                self.map(Page4K::new(start), PFN4K::new(pa), attrs);
            }
            PFN4K::SIZE
        } else {
            panic!("bad page size");
        };
        
        start = start.wrapping_add(len);
        pa = pa.checked_add(len as u64).unwrap();
    }
}
```

**Optimization**: The algorithm greedily uses the largest page size that satisfies:
1. Remaining region size >= page size
2. Virtual address aligned to page size
3. Physical address aligned to page size

### 2.6 Page Table Allocator (mmu.rs lines 1207-1274)

**Static Arena** (lines 1217-1227):
```rust
mod arena {
    const PAGE_SIZE: usize = 4096;
    const PAGE_ARENA_SIZE: usize = 128 * PAGE_SIZE;  // 512 KiB total
    
    // Minimum 16 4KiB pages per RFD 215
    const_assert!(PAGE_ARENA_SIZE > 16 * PAGE_SIZE);
    
    static PAGE_ALLOCATOR: SyncUnsafeCell<BumpAlloc<{ PAGE_ARENA_SIZE }>> =
        SyncUnsafeCell::new(BumpAlloc::new([0; PAGE_ARENA_SIZE]));
}
```

**RFD 215 Compliance Requirements**:
1. All page table memory must be from physically contiguous region
2. PML4 must be at lowest physical address in region
3. Minimum 16 4KiB pages (64 KiB) but typically 128 pages (512 KiB)
4. All kernel nucleus leaf PTEs must have bit 11 set

### 2.7 Reserved Region Tracking (LoaderPageTable)

**Structure** (mmu.rs lines 1100-1112):
```rust
pub(crate) struct LoaderPageTable {
    page_table: &'static mut PageTable,
    reserved: Vec<Range<mem::V4KA>>,
}

impl LoaderPageTable {
    pub(crate) fn new(
        page_table: &'static mut PageTable,
        reserved: &[Range<mem::V4KA>],
    ) -> LoaderPageTable {
        LoaderPageTable { page_table, reserved: reserved.into() }
    }
}
```

**Protected Regions** (phbl.rs lines 96-99):
```rust
let cpio_region = cpio_addr()..saddr();
let loader_region = saddr()..eaddr();
let mmio_region = mmio_addr()..mmio_end();
let reserved_regions = [loader_region.clone(), cpio_region, mmio_region];
```

---

## 3. HARDWARE ABSTRACTION

### 3.1 UART Driver (uart.rs)

**Device Enumeration** (lines 415-420):
```rust
#[repr(usize)]
pub enum Device {
    Uart0 = UART_MMIO_BASE_ADDR,                    // 0xFEDC_9000
    _Uart1 = UART_MMIO_BASE_ADDR + 0x1000,          // 0xFEDC_A000
    _Uart2 = UART_MMIO_BASE_ADDR + 0x5000,          // 0xFEDC_E000
    _Uart3 = UART_MMIO_BASE_ADDR + 0x6000,          // 0xFEDC_F000
}
```

**UART Base Address**: `const UART_MMIO_BASE_ADDR: usize = 0xFEDC_9000;` (line 252)

**Register Layout** - DesignWare APB UART (NS16550 compatible):

```rust
#[repr(C)]
struct ConfigMmio {  // During DLAB = 1
    dll: Dll,        // 0x00: Divisor Latch Low
    dlh: Dlh,        // 0x04: Divisor Latch High
    fcr: Fcr,        // 0x08: FIFO Control Register
    lcr: Lcr,        // 0x0C: Line Control Register
    mcr: Mcr,        // 0x10: Modem Control Register
    _res: [u32; 29],
    srr: Srr,        // 0x88: Software Reset Register
    _rest: [u32; 29],
}

#[repr(C)]
struct MmioRead {
    rbr: Rbr,        // 0x00: Receive Buffer Register
    _ier: u32,       // 0x04: Interrupt Enable Register
    _iir: u32,       // 0x08: Interrupt ID Register
    _lcr: u32,       // 0x0C: Line Control Register
    _mcr: u32,       // 0x10: Modem Control Register
    _lsr: Lsr,       // 0x14: Line Status Register
    _msr: u32,       // 0x18: Modem Status Register
    _scr: u32,       // 0x1C: Scratch Register
    // ... more registers through offset 0xFC
}

#[repr(C)]
struct MmioWrite {
    thr: Thr,        // 0x00: Transmit Hold Register
    // ... mirrors MmioRead layout
}
```

**Initialization** (lines 434-443):
```rust
fn init(self, rate: Rate, data: Datas, stop: Stops, par: Parity) -> bool {
    let uart = self.reset();
    uart.set_rate(rate);           // Set baud rate divisor
    uart.set_data_bits(data);      // 8 bits
    uart.set_stop_bits(stop);      // 1 stop bit
    uart.set_parity(par);          // No parity
    uart.config_flow_control();    // Enable auto flow control
    uart.config_fifos();           // Enable and configure FIFOs
    true
}
```

**Baud Rate Configuration** (lines 276-288):
```rust
fn set_rate(&mut self, rate: Rate) {
    const SCLK: u32 = 48_000_000;  // 48 MHz system clock
    let divisor = SCLK / (16 * rate as u32);
    
    let dll = Dll(divisor & 0xFF);
    let dlh = Dlh(divisor >> 8);
    
    unsafe {
        // Set DLAB = 1 to access divisor latch
        let lcr = self.lcr().with_dlab(true);
        ptr::write_volatile(&mut self.lcr, lcr);
        ptr::write_volatile(&mut self.dll, dll);
        ptr::write_volatile(&mut self.dlh, dlh);
        // Clear DLAB
        let lcr = self.lcr().with_dlab(false);
        ptr::write_volatile(&mut self.lcr, lcr);
    }
}
```

**Rate Configuration** (lines 131-133):
```rust
#[repr(u32)]
enum Rate {
    B3M = 3_000_000u32,  // 3 Mbps
}

// Example: UART0 initialization
Device::Uart0.init(Rate::B3M, Datas::Bits8, Stops::Stop1, Parity::No);
```

**Data Transmission** (lines 500-512):
```rust
fn putb(&mut self, b: u8) {
    // Wait for TX FIFO not full
    while {
        let usr = unsafe { ptr::read_volatile(&self.write_mmio_mut().usr) };
        !usr.tx_fifo_not_full()
    } {
        core::hint::spin_loop();
    }
    
    let data = Thr(0).with_data(b);
    unsafe {
        ptr::write_volatile(&mut self.write_mmio_mut().thr, data);
    }
}
```

**Console Printf Support** (lines 537-547):
```rust
impl fmt::Write for Uart {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for b in s.bytes() {
            if b == b'\n' {
                self.putb(b'\r');  // CR+LF conversion
            }
            self.putb(b);
        }
        Ok(())
    }
}

#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}
```

### 3.2 IO Mux Configuration (iomux.rs)

**GPIO Base Addresses**:
```rust
const GPIO_BASE_ADDR: usize = 0xFED8_0000;    // GPIO memory map base
const IOMUX_BASE_ADDR: usize = GPIO_BASE_ADDR + 0x0D00;  // 0xFED8_0D00
```

**Processor Family Detection** (lines 64-71):
```rust
fn cpuinfo() -> Option<(u8, u8, u8, Option<u32>)> {
    let cpuid = x86::cpuid::CpuId::new();
    let features = cpuid.get_feature_info()?;
    let family = features.family_id();        // CPUID EAX[11:8] + base
    let ext = cpuid.get_extended_processor_and_feature_identifiers()?;
    let pkg_type = (family > 0x10).then_some(ext.pkg_type());
    Some((family, features.model_id(), features.stepping_id(), pkg_type))
}
```

**Supported Processors** (lines 42-57):
```rust
match cpuinfo()? {
    // Naples: Family 0x17, Model 0x00-0x0F
    (0x17, 0x00..=0x0f, 0x0..=0xf, _) |
    // Rome: Family 0x17, Model 0x30-0x3F
    (0x17, 0x30..=0x3f, 0x0..=0xf, _) |
    // Milan: Family 0x19, Model 0x00-0x0F
    (0x19, 0x00..=0x0f, 0x0..=0xf, _) |
    // Genoa: Family 0x19, Model 0x10-0x1F, SP5
    (0x19, 0x10..=0x1f, 0x0..=0xf, Some(SP5)) |
    // Bergamo/Sienna: Family 0x19, Model 0xA0-0xAF, SP5
    (0x19, 0xa0..=0xaf, 0x0..=0xf, Some(SP5)) |
    // Turin: Family 0x1a, Model 0x00-0x1F, SP5
    (0x1a, 0x00..=0x1f, 0x0..=0xf, Some(SP5)) => {
        Some(&[
            (135, GpioX::F0),  // UART0 CTS
            (136, GpioX::F0),  // UART0 RXD
            (137, GpioX::F0),  // UART0 RTS
            (138, GpioX::F0),  // UART0 TXD
        ])
    },
    _ => None,
}
```

**Pin Function Values**:
```rust
enum GpioX {
    F0 = 0b00,  // Function 0 (primary)
    _F1 = 0b01, // Function 1
    _F2 = 0b10, // Function 2
    _F3 = 0b11, // Function 3
}
```

**UART0 Pin Configuration** (lines 51-56):
- Pin 135: UART0 CTS (Clear-To-Send)
- Pin 136: UART0 RXD (Receive Data)
- Pin 137: UART0 RTS (Request-To-Send)
- Pin 138: UART0 TXD (Transmit Data)

### 3.3 Interrupt Descriptor Table (IDT) (idt.rs)

**Gate Descriptor Structure** (lines 23-40):
```rust
bitstruct! {
    pub struct GateDesc(u128) {
        pub offset0: u16 = 0..16;              // Lower 16 bits of handler
        pub segment_selector: u16 = 16..32;    // Code segment selector
        pub stack_table_index: u8 = 32..35;    // IST index (0 = RSP0)
        mbz0: bool = 35;                       // Must be zero
        mbz1: bool = 36;                       // Must be zero
        mbz2: u8 = 37..40;                     // Must be zero
        pub fixed_type: u8 = 40..44;           // Type (0xE = interrupt gate)
        mbz3: bool = 44;                       // Must be zero
        pub privilege_level: u8 = 45..47;      // DPL (kernel = 0b00)
        pub present: bool = 47;                // Present bit
        pub offset16: u16 = 48..64;            // Middle 16 bits
        pub offset32: u32 = 64..96;            // Upper 32 bits
        reserved: u32 = 96..128;               // Reserved
    }
}
```

**IDT Table** (lines 223-243):
```rust
#[repr(C, align(4096))]
struct Idt {
    entries: [GateDesc; 256],  // All 256 exception/interrupt vectors
}

fn init(&mut self) {
    // Generate 256 exception handler stubs using seq! macro
    self.entries = seq!(N in 0..=255 {
        [#(
            GateDesc::new(vector~N),
        )*]
    });
}
```

**Exception Handling Flow**:
1. CPU pushes RIP, CS, RFLAGS, RSP, SS
2. Vector stub pushes vector number and error code (or zero)
3. All stubs jump to `alltraps()` function
4. `alltraps()` saves all general-purpose and segment registers
5. Calls Rust `trap()` handler
6. Restores registers and issues IRETQ

**Vector Stub Macros** (lines 107-154):
```rust
macro_rules! gen_stub {
    ($name:ident, $vecnum:expr) => {
        #[unsafe(naked)]
        unsafe extern "C" fn $name() -> ! {
            naked_asm!("pushq $0; pushq ${}; jmp {}",
                const $vecnum, sym alltraps,
                options(att_syntax))
        }
    };
    ($name:ident, $vecnum:expr, err) => {
        #[unsafe(naked)]
        unsafe extern "C" fn $name() -> ! {
            naked_asm!("pushq ${}; jmp {}",
                const $vecnum, sym alltraps,
                options(att_syntax))
        }
    };
}
```

**Vectors with Hardware Error Codes** (lines 129-149):
- Vector 8: Double Fault (generates error code)
- Vector 10: Invalid Task State Segment
- Vector 11: Segment Not Present
- Vector 12: Stack Segment Fault
- Vector 13: General Protection Fault
- Vector 14: Page Fault
- Vector 17: Alignment Check

**Trap Handler** (lines 261-276):
```rust
extern "C" fn trap(frame: &mut TrapFrame) {
    println!("Exception:");
    println!("{frame:#x?}");
    println!("cr0: {:#x}", unsafe { x86::controlregs::cr0() });
    println!("cr2: {:#x}", unsafe { x86::controlregs::cr2() });  // Faulting address
    println!("cr3: {:#x}", unsafe { x86::controlregs::cr3() });  // Page table root
    println!("cr4: {:#x}", unsafe { x86::controlregs::cr4() });
    println!("efer: {:#x}", unsafe { x86::msr::rdmsr(x86::msr::IA32_EFER) });
    unsafe {
        backtrace(frame.rbp);
    }
    frame.rip = crate::phbl::dnr as usize as u64;  // Halt on exception
}
```

---

## 4. BUILD SYSTEM

### 4.1 Cargo Configuration (Cargo.toml)

**Workspace Structure** (line 1):
```toml
workspace = { members = ["xtask"] }
```

**Package Metadata**:
- Edition: 2024
- License: MPL-2.0

**Dependencies**:
```toml
bit_field = "0.10"                                 # Bitfield manipulation
bitstruct = "0.1"                                  # Struct bitstruct derivation
cpio_reader = "0.1"                                # CPIO archive reading
goblin = { version = "0.10", default-features = false, features = [
    "endian_fd",                                   # Endianness detection
    "elf64",                                       # 64-bit ELF support
    "elf32",                                       # 32-bit ELF support (fallback)
    "alloc",                                       # Allocator support
] }
miniz_oxide = "0.8"                                # ZLIB decompression
seq-macro = "0.3"                                  # Sequence generation macros
static_assertions = "1.1"                          # Compile-time assertions
x86 = "0.52"                                       # x86/x86_64 CPU instruction bindings
```

**Build Profile**:
```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

### 4.2 Custom Target Specification (x86_64-oxide-none-elf.json)

**Target Definition**:
```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128",
    "target-endian": "little",
    "target-pointer-width": 64,
    "target-c-int-width": 32,
    "panic-strategy": "abort",
    "arch": "x86_64",
    "os": "none",
    "vendor": "oxide",
    "executables": true,
    "relocation-model": "static",
    "code-model": "small",
    "frame-pointer": "always",
    "disable-redzone": true,
    "features": "-avx,-avx2,-avx512bf16,-f16c,-fxsr,-mmx,-sse,-sse2,-sse3,-sse4.1,-sse4.2,-sse4a,-ssse3,-x87,+soft-float",
    "rustc-abi": "x86-softfloat",
    "linker-flavor": "ld",
    "linker": "gld",
    "no-default-libraries": true,
    "pre-link-args": {
        "ld": [
            "-nostdlib",
            "-Tsrc/phbl.ld",
            "-zmax-page-size=4096"
        ],
        "ld.lld": [
            "-nostdlib",
            "-Tsrc/phbl.ld"
        ]
    }
}
```

**Key Settings**:
- **Soft Float**: Disables SSE/AVX/FPU instructions for maximum compatibility
- **Frame Pointer**: Always kept for stack unwinding
- **Redzone Disabled**: x86-64 redzone not used (interrupt safety)
- **Linker**: GNU ld or LLVM lld with custom linker script
- **Page Size**: 4096 bytes (-zmax-page-size=4096)

### 4.3 Linker Script (phbl.ld)

**Fixed Memory Locations**:
```ld
ENTRY(reset);

HIDDEN(bootblock = 0x000000007ffef000);    // Bootblock virtual address
HIDDEN(resetaddr = 0x000000007ffefff0);    // Reset vector address
```

**Section Placement**:
```ld
SECTIONS {
    .start bootblock : {
        FILL(0xffffffff);
        *(.start)
    } :srodata
    
    .start.rodata ALIGN(64) : {
        *(.start.rodata)
    } :srodata
    
    .reset resetaddr : {
        FILL(0xffffffff);
        *(.reset)
        __eloader = ALIGN(65536);
    } :srodata
}
```

**Memory Layout**:
1. **.start**: 16/32-bit startup code
2. **.start.rodata**: GDT and early page tables
3. **.reset**: Reset vector and handler
4. **.text**: Executable code (with size calculation)
5. **.rodata**: Read-only data (with size calculation)
6. **.data**: Initialized read-write data (with size calculation)
7. **.bss**: Uninitialized data (with size calculation)

**Size Calculations**:
```ld
textsize = SIZEOF(.text);
rodatasize = SIZEOF(.rodata);
datasize = SIZEOF(.data);
bsssize = SIZEOF(.bss);

_BL_SPACE = __eloader - __sloader;  // Total bootloader size
```

**Program Headers**:
```ld
PHDRS {
    text    PT_LOAD;
    rodata  PT_LOAD;
    data    PT_LOAD;
    srodata PT_LOAD;  // Startup rodata
    empty   PT_NULL;
}
```

### 4.4 Build Process (xtask)

**xtask Commands**:

```rust
cargo xtask build --cpioz=<path> [--release] [--locked]
```
- Builds PHBL binary with embedded CPIO archive
- Accepts compressed CPIO via `--cpioz` environment variable
- Uses `-Z build-std` to compile core/alloc for custom target
- Output: `target/x86_64-oxide-none-elf/[debug|release]/phbl`

```rust
cargo xtask test [--debug|--release] [--locked]
```
- Runs unit tests on the host

```rust
cargo xtask disasm [--debug|--release] [--source] --cpioz=<path>
```
- Generates disassembly listing with optional source interleaving

```rust
cargo xtask clippy [--locked]
```
- Runs Rust linter

```rust
cargo xtask expand
```
- Expands all macros for inspection

```rust
cargo xtask clean
```
- Removes build artifacts

**Build Flags**:
```bash
PHBL_PHASE1_COMPRESSED_CPIO_ARCHIVE_PATH=<path>  # CPIO archive path
CARGO_TARGET_X86_64_OXIDE_NONE_ELF_LINKER=gld    # Custom linker path
TARGET=x86_64-oxide-none-elf                      # Target triple
OBJDUMP=llvm-objdump                              # Objdump binary
```

---

## 5. HANDOFF PROTOCOL

### 5.1 Kernel Entry Point

**Function Signature** (loader.rs line 20-27):
```rust
type Thunk = unsafe extern "C" fn(
    ramdisk_paddr: u64,    // rdi: physical address of CPIO archive
    ramdisk_len: usize,    // rsi: size of CPIO archive
    _rdx: u64,             // rdx: reserved (0)
    _rcx: u64,             // rcx: reserved (0)
    _r8: u64,              // r8:  reserved (0)
    _r9: u64,              // r9:  reserved (0)
);
```

**Calling Convention**: System V AMD64 ABI (x86-64 calling convention)

### 5.2 Kernel Invocation (main.rs lines 28-34)

```rust
let ramdisk = expand_ramdisk();     // &[u8] to decompressed CPIO
let kernel = find_kernel(ramdisk);  // Locate kernel ELF in archive
let entry = loader::load(&mut config.page_table, kernel)
    .expect("loaded kernel");
println!("jumping into kernel...");
entry(ramdisk.as_ptr().addr() as u64, ramdisk.len());
panic!("main returning");
```

**Kernel Image Location in Archive**:
```rust
fn find_kernel(cpio: &[u8]) -> &[u8] {
    for entry in cpio_reader::iter_files(cpio) {
        if entry.name() == "platform/oxide/kernel/amd64/unix" {
            return entry.file();
        }
    }
    panic!("could not locate unix in cpio archive");
}
```

### 5.3 ELF Kernel Loading (loader.rs)

**Parsing** (lines 29-56):
```rust
pub(crate) fn load(
    page_table: &mut LoaderPageTable,
    bytes: &[u8],
) -> Result<impl FnOnce(u64, usize)> {
    let elf = parse_elf(bytes)?;
    
    // Load all PT_LOAD segments
    for section in elf.program_headers.iter().filter(|&h| h.p_type == PT_LOAD) {
        let file_range = section.file_range();
        if bytes.len() < file_range.end {
            return Err("load: truncated executable");
        }
        load_segment(page_table, section, &bytes[file_range])?;
    }
    
    // Transmute ELF entry point to callable function
    let entry = unsafe { core::mem::transmute::<u64, Thunk>(elf.entry) };
    Ok(move |ramdisk_paddr: u64, ramdisk_len: usize| unsafe {
        entry(ramdisk_paddr, ramdisk_len, 0, 0, 0, 0)
    })
}
```

**ELF Header Validation** (lines 61-91):
- Machine type: EM_X86_64 (0x3E)
- Class: 64-bit (ELFCLASS64)
- Endianness: Little-endian (ELFDATA2LSB)
- Type: ET_EXEC (executable file)
- Entry point: Must not be zero
- Version: EV_CURRENT

**Segment Loading** (lines 115-163):
```rust
fn load_segment(
    page_table: &mut LoaderPageTable,
    section: &ProgramHeader,
    bytes: &[u8],
) -> Result<()> {
    let pa = section.p_paddr;
    
    // Validate alignment
    if !pa.is_multiple_of(mem::P4KA::ALIGN) {
        return Err("Program section is not physically 4KiB aligned");
    }
    
    let vm = section.vm_range();
    
    // Validate canonical addressing
    if vm.contains(&mem::LOW_CANON_SUP) || vm.contains(&mem::HI_CANON_INF) {
        return Err("Program section is not canonical");
    }
    
    // Validate virtual alignment
    if !vm.start.is_multiple_of(mem::V4KA::ALIGN) {
        return Err("Program section not virtually 4KiB aligned");
    }
    
    if vm.end <= vm.start {
        return Err("Program section ends before start or is empty");
    }
    
    let start = mem::V4KA::new(vm.start);
    let end = mem::V4KA::new(round_up_4k(vm.end));
    let region = start..end;
    let pa = mem::P4KA::new(pa);
    
    // Step 1: Map region as RW data for copying
    {
        unsafe {
            page_table.map_region(region.clone(), mem::Attrs::new_data(), pa)?;
            let p = page_table.try_with_addr(start.addr()).unwrap();
            let len = end.addr() - start.addr();
            core::ptr::write_bytes(p, 0, len);  // Zero BSS
            
            let dst = core::slice::from_raw_parts_mut(p, len);
            let len = usize::min(bytes.len(), dst.len());
            if len > 0 {
                dst[..len].copy_from_slice(&bytes[..len]);
            }
        }
    }
    
    // Step 2: Remap with correct attributes (honors ELF flags)
    let attrs = mem::Attrs::new_kernel(
        section.is_read(),      // Based on PF_R flag
        section.is_write(),     // Based on PF_W flag
        section.is_executable(), // Based on PF_X flag
    );
    unsafe {
        page_table.map_region(region, attrs, pa)?;
    }
    
    Ok(())
}
```

**Permission Mapping**:
- Readable (PF_R) → Present bit
- Writable (PF_W) → Write bit
- Executable (PF_X) → NX bit clear
- All kernel pages: Bit 11 (k flag) set per RFD 215

### 5.4 CPU State Guarantees

**Provided to Kernel**:

| Component | State |
|-----------|-------|
| **Mode** | 64-bit long mode with paging enabled |
| **%rax** | Undefined |
| **%rbx** | Undefined |
| **%rcx** | 0 |
| **%rdx** | 0 |
| **%rsi** | CPIO archive size |
| **%rdi** | CPIO archive physical address |
| **%r8-r9** | 0 |
| **%rsp** | Valid 4KiB-aligned stack, (%rsp + 8) % 16 == 0 |
| **%rbp** | Undefined |
| **%fs** | Reset state (0) |
| **%gs** | Reset state (0) |
| **%ds** | 0 (cleared) |
| **%es** | 0 (cleared) |
| **%ss** | 0 (cleared) |
| **%cs** | GDT code selector (kernel mode) |
| **%rflags** | IF=0, DF=0 |
| **%cr0** | PG=1, WP=1, PE=1 |
| **%cr3** | Physical address of PML4 |
| **%cr4** | PAE=1 |
| **EFER** | LME=1, NX=1 |

**Memory State**:
- Kernel mapped at linked addresses with correct permissions
- Kernel BSS zeroed
- Stack: 32 KiB identity-mapped, writable
- UART: Uncached, writable for debugging
- Page tables: Identity-mapped, contiguous per RFD 215
- PML4 at lowest physical address in page table region

### 5.5 Kernel Archive Contents

**CPIO Path**: `platform/oxide/kernel/amd64/unix`

**Archive Format**: ZLIB-compressed CPIO (newc format)

**Expected Contents**:
- Kernel ELF binary at `platform/oxide/kernel/amd64/unix`
- Additional modules/drivers (implementation-specific)

---

## 6. DEPENDENCIES & CRATES

### 6.1 Core Dependencies

| Crate | Version | Purpose | Usage |
|-------|---------|---------|-------|
| `bit_field` | 0.10 | Bitfield manipulation | PTE bit extraction, interrupt gate descriptors |
| `bitstruct` | 0.1 | Struct bitfield derivation | UART registers, control registers, PTE bits |
| `cpio_reader` | 0.1 | CPIO archive parsing | Iterate CPIO entries, extract kernel image |
| `goblin` | 0.10 | ELF binary parsing | Parse kernel ELF header, program headers, entry point |
| `miniz_oxide` | 0.8 | ZLIB decompression | Decompress CPIO archive in-place |
| `seq-macro` | 0.3 | Sequence generation | Generate 256 exception handlers (vector0..vector255) |
| `static_assertions` | 1.1 | Compile-time assertions | Validate struct sizes (e.g., page table = 4096 bytes) |
| `x86` | 0.52 | x86/x86_64 primitives | CPUID detection, MSR operations, control registers |

### 6.2 Cargo Flags (in target spec)

```json
"features": "-avx,-avx2,-avx512bf16,-f16c,-fxsr,-mmx,-sse,-sse2,-sse3,-sse4.1,-sse4.2,-sse4a,-ssse3,-x87,+soft-float"
```

**Disabled** (no SIMD/FPU):
- AVX, AVX2, AVX-512
- SSE variants
- MMX
- x87 FPU
- FSX, F16C

**Rationale**: Ensure maximum processor compatibility and avoid FPU context switches

---

## 7. PLATFORM-SPECIFIC VS. GENERIC CODE ANALYSIS

### 7.1 PLATFORM-SPECIFIC CODE (Requires V3000 Porting)

| Component | File | Lines | Details | V3000 Action |
|-----------|------|-------|---------|--------------|
| **UART Base Address** | uart.rs | 252 | `0xFEDC_9000` | Verify V3000 UART0 MMIO address |
| **UART Device Offsets** | uart.rs | 417-419 | +0x1000, +0x5000, +0x6000 | Verify offset spacing on V3000 |
| **System Clock Rate** | uart.rs | 277 | 48 MHz reference clock | Verify V3000 UART clock frequency |
| **GPIO Base Address** | iomux.rs | 26 | `0xFED8_0000` | Verify V3000 GPIO MMIO address |
| **IO Mux Base Address** | iomux.rs | 27 | `0x0D00` offset | Verify V3000 IO mux offset |
| **UART Pin Numbers** | iomux.rs | 51-56 | Pins 135-138 for RXD/TXD | Verify V3000 pin configuration |
| **Processor Family ID** | iomux.rs | 45-50 | 0x17, 0x19, 0x1a families | Add V3000 family ID and model ranges |
| **Memory Map** | phbl.rs | 180, 186 | 0x8000_0000 MMIO start, 0x1_0000_0000 end | Verify V3000 MMIO space boundaries |
| **Bootblock Address** | phbl.ld | 6 | `0x000000007ffef000` | Verify V3000 ROM address space |
| **Reset Vector** | phbl.ld | 7 | `0x000000007ffefff0` | Verify V3000 reset vector address |
| **CPIO Region Size** | phbl.rs | 124 | 128 MiB allocation | Adjust based on V3000 memory available |

### 7.2 GENERIC CODE (Reusable As-Is)

| Component | File | Why Generic |
|-----------|------|-------------|
| **16→32→64-bit Mode Transition** | start.S | CPU architecture standard, no platform specifics |
| **GDT Setup** | start.S | Standard x86-64 architecture |
| **MTRR Configuration** | start.S | Generic x86-64 memory type configuration |
| **PAE & Long Mode Enable** | start.S | Standard x86-64 paging setup |
| **Page Table Algorithm** | mmu.rs | Generic 4-level paging, works on all AMD64 |
| **ELF Parsing** | loader.rs | Uses standard ELF format, architecture-agnostic |
| **CPIO Decompression** | main.rs | Standard ZLIB and CPIO formats |
| **Exception Handling** | idt.rs | Standard x86-64 exception vector structure |
| **Type-Driven Design** | mmu.rs | Rust memory safety patterns, reusable |
| **Memory Attributes** | mem.rs | Generic permission model |
| **Allocator** | allocator.rs | Generic bump allocator, works on any arch |

---

## 8. KEY CONSTANTS FOR V3000 PORTING

### 8.1 UART Configuration

```rust
// Current (Milan/Genoa/Turin)
const UART_MMIO_BASE_ADDR: usize = 0xFEDC_9000;
const UART_SCLK: u32 = 48_000_000;  // 48 MHz
const UART_BAUD_RATE: Rate = Rate::B3M;  // 3 Mbps

// V3000 values (to be verified from datasheets)
// const UART_MMIO_BASE_ADDR: usize = 0x????_????;
// const UART_SCLK: u32 = ????;
// const UART_BAUD_RATE: Rate = Rate::B???;
```

### 8.2 IO Mux Configuration

```rust
// Current processor families
(0x17, 0x00..=0x0f, _, _)  // Naples
(0x17, 0x30..=0x3f, _, _)  // Rome
(0x19, 0x00..=0x0f, _, _)  // Milan
(0x19, 0x10..=0x1f, _, Some(SP5))  // Genoa
(0x19, 0xa0..=0xaf, _, Some(SP5))  // Bergamo/Sienna
(0x1a, 0x00..=0x1f, _, Some(SP5))  // Turin

// Add V3000
(0x??, 0x??..=0x??, _, ?)  // V3000 family ID and model range
```

### 8.3 Memory Map

```rust
// Current layout
const MMIO_START: usize = 0x8000_0000;    // 2 GiB
const MMIO_END: usize = 0x1_0000_0000;    // 4 GiB
const BOOTBLOCK: usize = 0x7ffef000;      // 2 GiB - 64 KiB - 4 KiB
const RESETADDR: usize = 0x7ffefff0;      // Reset vector offset
const CPIO_SIZE: usize = 128 * MIB;       // 128 MiB ramdisk

// Verify for V3000
// const MMIO_START: usize = 0x????_????;
// const BOOTBLOCK: usize = 0x????_????;
```

---

## 9. CRITICAL DESIGN PATTERNS FOR PORTING

### 9.1 Type-Driven Memory Safety

```rust
// Pattern 1: Alignment enforcement via types
#[derive(Clone, Copy)]
pub(crate) struct V4KA(usize);  // Only 4KiB-aligned virtual addresses
impl V4KA {
    pub(crate) const fn new(va: usize) -> V4KA {
        assert!(va & Self::MASK == 0);  // Compile-time or runtime check
        V4KA(va)
    }
}

// Pattern 2: Page/Frame type matching
trait Page {
    type FrameType: Frame;
}
// This ensures Page4K only maps to PFN4K, never PFN2M
impl Page for Page4K {
    type FrameType = PFN4K;
}

// Pattern 3: Table level type safety
trait Table {
    type EntryType;
    type MappingType: Mapping;
}
// PML3 can only hold PML3E (either PML2 pointer or 1GiB page)
enum PML3E {
    Next(&'static mut PML2),
    Page(PFN1G, mem::Attrs),
}
```

**Benefit**: Many page table errors caught at compile-time, not runtime

### 9.2 Hardware Abstraction Pattern

```rust
// UART abstraction
impl Device {
    fn init(self, rate: Rate, data: Datas, stop: Stops, par: Parity) -> bool {
        // Initialize sequence: reset → set_rate → config_fifos → config_flow
    }
}

// CPUID detection
fn cpuinfo() -> Option<(u8, u8, u8, Option<u32>)> {
    let cpuid = x86::cpuid::CpuId::new();
    // Extract family, model, stepping, package type
}

// To port: Create similar abstractions for V3000-specific hardware
// pub struct V3000Config { ... }
// impl V3000Config { fn init() -> ... }
```

### 9.3 Allocator Pattern for Page Tables

```rust
// Specialized allocator ensures page table memory properties:
mod arena {
    static PAGE_ALLOCATOR: SyncUnsafeCell<BumpAlloc<{ PAGE_ARENA_SIZE }>> = ...;
    
    unsafe impl Allocator for TableAlloc {
        fn allocate(&self, layout: Layout) -> Result<...> {
            // Always align to PAGE_SIZE, allocate from static arena
            assert_eq!(align, PAGE_SIZE);
            assert_eq!(size, PAGE_SIZE);
            // This guarantees contiguity per RFD 215
        }
    }
}

// To port: May need to adjust PAGE_ARENA_SIZE if V3000 has different
// page table requirements
```

### 9.4 Attribute-Based Permission Mapping

```rust
// Single source of truth for memory attributes
pub(crate) struct Attrs {
    r: bool,  // Readable
    w: bool,  // Writable
    x: bool,  // Executable
    c: bool,  // Cacheable
    k: bool,  // Kernel nucleus bit 11
}

// Maps directly to PTE bits
fn new<F: Frame>(pa: F, attrs: mem::Attrs) -> PTE {
    PTE::from_phys_addr(pa.phys_addr())
        .with_p(attrs.r())
        .with_w(attrs.w())
        .with_nx(!attrs.x())
        .with_nc(!attrs.c())
        .with_k(attrs.k())
}
```

**Advantage**: ELF permissions → Attrs → PTE bits is traceable and correct

---

## 10. BUILD PROCESS PORTING CHECKLIST

### 10.1 Prerequisite Tools
- [ ] Rust toolchain with nightly (for `-Z build-std`)
- [ ] GNU ld or LLVM lld linker
- [ ] LLVM objdump (for disassembly)
- [ ] QEMU or real hardware for testing

### 10.2 Target Configuration
- [ ] Verify x86_64 custom target JSON still valid
- [ ] Confirm data layout is correct for V3000
- [ ] Validate soft-float flag compatibility

### 10.3 CPIO Archive Preparation
- [ ] Build phase1 OS package for V3000
- [ ] Compress with pinprick utility (must be ZLIB)
- [ ] Verify kernel binary path: `platform/oxide/kernel/amd64/unix`
- [ ] Test archive decompression

### 10.4 Build Verification
```bash
cargo xtask build --cpioz=/path/to/phase1.cpio.z
cargo xtask disasm --cpioz=/path/to/phase1.cpio.z --source
```

---

## 11. RFD REFERENCES & DESIGN DECISIONS

### 11.1 Referenced RFDs

- **RFD 284**: "Pico Host Boot Loader" - Complete PHBL design specification
- **RFD 215**: "Page Table Memory Contract" - Bit 11 ownership protocol, contiguity requirements
- **RFD 75, 216**: Security model references (TOCTOU exposure)

### 11.2 Key Design Decisions Explained

**Decision**: Compiled CPIO archive instead of external storage
- Simplifies flash layout
- Eliminates need for separate archive reading logic
- Any update requires rewriting entire image anyway
- Feasible for < 1 GiB kernels

**Decision**: Hard-coded kernel path in CPIO
- `platform/oxide/kernel/amd64/unix`
- No parameterization mechanism
- Could be extended in future if needed

**Decision**: No cryptographic validation in PHBL
- PSP/Root of Trust validates before loading PHBL
- PHBL defers all security checks to earlier stages
- Acceptable architectural trade-off

**Decision**: Type-driven page table construction
- Prevents page size mismatches at compile-time
- "Parse, don't validate" philosophy
- Makes porting safer (type system guides implementation)

---

## APPENDIX: FILE REFERENCE INDEX

### Source Files
- `/tmp/phbl/src/start.S` - Assembly entry point (265 lines)
- `/tmp/phbl/src/main.rs` - Rust entry point and kernel loading (87 lines)
- `/tmp/phbl/src/phbl.rs` - Initialization and memory setup (236 lines)
- `/tmp/phbl/src/mmu.rs` - Page table implementation (1275 lines)
- `/tmp/phbl/src/mem.rs` - Memory types and address management (211 lines)
- `/tmp/phbl/src/loader.rs` - ELF kernel loading (172 lines)
- `/tmp/phbl/src/uart.rs` - UART driver (564 lines)
- `/tmp/phbl/src/iomux.rs` - IO mux configuration (72 lines)
- `/tmp/phbl/src/idt.rs` - Exception handling (328 lines)
- `/tmp/phbl/src/allocator.rs` - Bump allocator (133 lines)

### Build Files
- `/tmp/phbl/Cargo.toml` - Package manifest
- `/tmp/phbl/Cargo.lock` - Dependency lock file
- `/tmp/phbl/build.rs` - Build script
- `/tmp/phbl/x86_64-oxide-none-elf.json` - Custom target spec
- `/tmp/phbl/src/phbl.ld` - Linker script (70 lines)
- `/tmp/phbl/xtask/Cargo.toml` - Build task manifest
- `/tmp/phbl/xtask/src/main.rs` - Build task implementation (215 lines)

### Documentation
- `/tmp/phbl/README.md` - Build and usage instructions
- `/tmp/phbl/LICENSE.txt` - MPL-2.0 license
- `https://rfd.shared.oxide.computer/rfd/0284` - RFD 284 design doc

