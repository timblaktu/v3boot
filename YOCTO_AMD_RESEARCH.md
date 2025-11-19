# Yocto Project and OpenEmbedded Support for AMD Platforms
## Research Report for V3000 Bootloader Development

---

## Executive Summary

This report analyzes Yocto/OpenEmbedded support for AMD platforms to evaluate its suitability for V3000 bootloader development. While Yocto provides robust Linux distribution building capabilities and some AMD BSP support, **it does not address low-level firmware/bootloader construction** in the way Oxide's tools do.

**Key Finding**: Yocto and Oxide's amd-host-image-builder/phbl serve fundamentally different purposes:
- **Yocto**: Builds Linux distributions with kernel, rootfs, and UEFI bootloaders (GRUB/systemd-boot)
- **Oxide tools**: Construct SPI flash images with PSP firmware, AGESA blobs, and custom bootloaders

For V3000 bootloader development, Oxide's approach is more directly applicable, but Yocto may complement it for building the Linux payload.

---

## 1. Meta-AMD Layer Analysis

### 1.1 Official Repository

**Location**: https://git.yoctoproject.org/meta-amd

**Structure**:
```
meta-amd/
├── meta-amd-bsp/           # Board Support Packages
│   └── conf/machine/       # Machine configurations
├── meta-amd-distro/        # AMD-specific distribution layer
├── common/                 # Shared components
└── FEATURES.md             # Supported features per platform
```

### 1.2 Supported Platforms (Kirkstone Branch)

| Machine | Platform | Status |
|---------|----------|--------|
| `v1000` | Ryzen Embedded V1000 (Zen+) | Supported |
| `v2000` | Ryzen Embedded V2000 (Zen 2) | Supported |
| `e3000` | EPYC Embedded 3000 | Supported |
| `milan` | EPYC Milan (Server) | Supported |
| `rome` | EPYC Rome (Server) | Supported |
| `siena` | EPYC Siena | Supported |

**V3000 Status**: **NOT CURRENTLY SUPPORTED** in the official meta-amd layer. The V3000 series (Zen 3 based) launched in late 2022 but has not yet been added to the Yocto layers.

### 1.3 What Meta-AMD Provides

1. **Machine Configurations**: CPU tuning, architecture flags
2. **Graphics Support**: AMDGPU drivers, Vulkan (AMDVLK), ROCm
3. **Boot Images**: WIC images for UEFI/GRUB boot
4. **Kernel Configuration**: AMD-specific kernel options
5. **Firmware**: Linux-firmware packages (GPU microcode, CPU microcode)

### 1.4 What Meta-AMD Does NOT Provide

- **PSP firmware integration**
- **AGESA binary blob handling**
- **SPI flash image construction**
- **Custom bootloader building (like PHBL)**
- **APCB configuration tools**

---

## 2. Meta-AMD-BSP Details

### 2.1 Machine Configuration Example (v2000.conf)

```bitbake
# Typical AMD machine configuration
DEFAULTTUNE = "corei7-64"
require conf/machine/include/tune-corei7.inc

MACHINE_FEATURES = "alsa bluetooth usbgadget usbhost vfat wifi pci acpi"

# EFI boot configuration
EFI_PROVIDER = "grub-efi"
PREFERRED_VERSION_grub-efi = "2.06"

# Kernel selection
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
KERNEL_IMAGETYPE = "bzImage"

# WIC image configuration
WKS_FILE = "directdisk-gpt.wks"
IMAGE_FSTYPES = "wic wic.bmap"
```

### 2.2 Boot Configuration

Meta-AMD uses standard x86 UEFI boot flow:
1. **UEFI Firmware** (platform-provided, NOT built by Yocto)
2. **GRUB or systemd-boot** (built by Yocto)
3. **Linux Kernel** (built by Yocto)
4. **Root Filesystem** (built by Yocto)

**Critical Gap**: The UEFI firmware itself (containing PSP, AGESA, etc.) is assumed to exist on the platform. Yocto does not build this.

### 2.3 Layer Dependencies

```
meta-amd-bsp
├── openembedded-core (meta)
├── meta-openembedded (meta-oe, meta-python, meta-networking)
└── meta-dpdk (optional)

meta-amd-distro
├── meta-amd-bsp
└── openembedded-core
```

---

## 3. Boot-Related Recipes

### 3.1 Available in Meta-AMD

| Recipe | Purpose | Location |
|--------|---------|----------|
| `grub-efi` | UEFI bootloader | OE-Core (used by meta-amd) |
| `systemd-boot` | Alternative UEFI bootloader | OE-Core |
| `linux-yocto` | Kernel | OE-Core (configured by meta-amd) |
| `linux-firmware` | CPU/GPU microcode | OE-Core |

### 3.2 WIC Image Creation

Meta-AMD uses WKS (WIC Kickstart) files to define disk layout:

```wks
# Example: directdisk-gpt.wks for UEFI boot
part /boot --source bootimg-efi --ondisk sda --fstype=vfat --label efi --size 100M --active
part / --source rootfs --ondisk sda --fstype=ext4 --label root --align 1024
bootloader --ptable gpt --timeout 5
```

### 3.3 Missing Low-Level Firmware Recipes

Meta-AMD does **not** include:
- PSP firmware directory construction
- AGESA bootloader integration
- APCB (AMD PSP Customization Block) generation
- SPI flash layout tools
- amdfwtool equivalents

---

## 4. AGESA and PSP Firmware Handling

### 4.1 Current State in Yocto

**Yocto does not handle AGESA/PSP firmware integration**. The meta-amd layer assumes:
- Platform has pre-existing UEFI/BIOS firmware
- PSP and AGESA are already integrated by board vendor
- Yocto only provides operating system components

### 4.2 Binary Blob Handling in Yocto

Yocto has mechanisms for handling proprietary binaries:

```bitbake
# Example: Incorporating closed-source firmware
LICENSE = "CLOSED"
# or
LICENSE = "Proprietary"
LICENSE_FLAGS = "commercial"

# In local.conf to allow:
LICENSE_FLAGS_ACCEPTED = "commercial"

# File distribution
SRC_URI = "file://proprietary-firmware.bin"
do_install() {
    install -m 0644 ${WORKDIR}/proprietary-firmware.bin ${D}/lib/firmware/
}
```

### 4.3 How Other Platforms Handle Firmware

**Intel FSP (Firmware Support Package)**:
- Intel provides FSP binaries that can be integrated with coreboot
- FSP handles memory training, chipset init (similar to AGESA)
- meta-intel does NOT build FSP; assumes pre-built UEFI

**AMD Xilinx (Versal/Zynq)**:
- Different architecture (ARM-based)
- Uses First Stage Boot Loader (FSBL)
- Has recipes in meta-xilinx for boot image generation
- More complete firmware build support than x86 AMD

---

## 5. SolidRun Yocto Support

### 5.1 Available SolidRun Layers

| Layer | Platform | Status |
|-------|----------|--------|
| `meta-solidrun-bsp` | General iMX6 | Active |
| `meta-solidrun-arm-imx6` | i.MX6 SoC | Active |
| `meta-solidrun-arm-imx8` | i.MX8 SoC | Active |
| `meta-solidrun-arm-lx2xxx` | LX2160A (ARM) | Active |
| `meta-clearfog` | Armada 388 | Active |

### 5.2 HoneyComb V3000 Support

**No Yocto layer currently exists** for the SolidRun HoneyComb V3000.

**Current software support**:
- Ubuntu Server (pre-installed images available)
- UEFI BIOS (built-in, not user-buildable)
- No public BSP for custom bootloader development

**Developer Resources**:
- Quick Start Guide: https://solidrun.atlassian.net/wiki/spaces/developer/pages/592904196/
- BIOS Settings: https://solidrun.atlassian.net/wiki/spaces/developer/pages/464027649/

### 5.3 Gap Analysis for V3000

SolidRun provides:
- Pre-built UEFI firmware
- Linux distribution images
- Hardware documentation

SolidRun does NOT provide:
- Source code for BIOS/UEFI
- PSP firmware packages
- AGESA blobs
- Tools to build custom boot ROM

---

## 6. Related Layers

### 6.1 Meta-Intel (For Comparison)

**Repository**: https://git.yoctoproject.org/meta-intel

**Approach**:
- Provides machine configurations for Intel platforms
- Uses OE-Core's grub-efi/systemd-boot for UEFI boot
- Does NOT build Intel FSP or ME firmware
- Assumes pre-existing UEFI/coreboot on platform

**Boot Configuration**:
```bitbake
MACHINE = "intel-corei7-64"
EFI_PROVIDER = "grub-efi"
# or
EFI_PROVIDER = "systemd-boot"
```

**Key Insight**: Meta-intel, like meta-amd, operates at the OS level, not firmware level.

### 6.2 Meta-Coreboot

**Repository**: https://github.com/zarhus/meta-coreboot

**Purpose**: Build coreboot utilities in Yocto environment

**Contents**:
```
meta-coreboot/
├── conf/           # Layer configuration
└── recipes-bsp/    # BSP recipes for coreboot tools
```

**Provides**:
- cbfstool (coreboot filesystem tool)
- ifdtool (Intel Flash Descriptor tool)
- Other coreboot utilities

**Does NOT Provide**:
- Complete coreboot builds
- AMD-specific firmware tools
- amdfwtool or PSP tools

### 6.3 Meta-X86 Usage

Standard x86 support in OE-Core can be used with AMD platforms:
- Generic x86-64 tuning
- UEFI boot support
- Standard bootloader recipes

However, this is even more generic than meta-amd and provides no AMD-specific optimizations.

---

## 7. Build System Approach for Binary Blobs

### 7.1 Yocto's Binary Blob Philosophy

Yocto provides infrastructure but **defers firmware blob responsibility to vendors**:

1. **Runtime firmware** (GPU, WiFi): Handled via `linux-firmware` recipe
2. **Boot firmware** (BIOS, UEFI): Not Yocto's responsibility
3. **Licensing**: Strict tracking via `LICENSE`, `LIC_FILES_CHKSUM`

### 7.2 Incorporating Proprietary Firmware in Yocto

```bitbake
# Recipe for proprietary firmware
DESCRIPTION = "Vendor Proprietary Firmware"
LICENSE = "CLOSED"
LIC_FILES_CHKSUM = ""

SRC_URI = "file://vendor-firmware.bin"

do_install() {
    install -d ${D}${nonarch_base_libdir}/firmware
    install -m 0644 ${WORKDIR}/vendor-firmware.bin \
        ${D}${nonarch_base_libdir}/firmware/
}

FILES:${PN} = "${nonarch_base_libdir}/firmware/*"
```

### 7.3 License Flag Acceptance

```conf
# In conf/local.conf
LICENSE_FLAGS_ACCEPTED = "commercial_vendor-firmware"
```

### 7.4 Limitations for Firmware Development

Yocto's blob handling is designed for **runtime firmware loading**, not for:
- Constructing boot ROMs
- Integrating PSP directories
- Building flash images with AGESA
- Generating APCB configurations

---

## 8. Comparison: Yocto vs Oxide Tools

### 8.1 Architectural Comparison

| Aspect | Yocto/OpenEmbedded | Oxide PHBL/amd-host-image-builder |
|--------|--------------------|------------------------------------|
| **Primary Purpose** | Linux distribution building | SPI flash ROM construction |
| **Boot Stage** | OS bootloader (GRUB) onward | Reset vector to kernel handoff |
| **Firmware Handling** | Assumes pre-existing | Actively constructs |
| **Output** | Disk images (.wic, .iso) | ROM images (.img for SPI flash) |
| **PSP Integration** | None | Core functionality |
| **AGESA Handling** | None | Required input |
| **Target Users** | Embedded Linux developers | Firmware engineers |

### 8.2 Boot Flow Comparison

**Yocto Boot Flow**:
```
[Pre-existing UEFI] -> GRUB -> Linux Kernel -> Rootfs
                       ^^^^
                    Yocto builds this and beyond
```

**Oxide Boot Flow**:
```
[PSP] -> [AGESA] -> PHBL -> Helios Kernel -> Ramdisk
^^^^^^^^^^^^^^^^^^^^^^^
amd-host-image-builder constructs entire SPI ROM
```

### 8.3 Use Case Mapping

| Use Case | Tool |
|----------|------|
| Build custom Linux distro for V3000 | Yocto |
| Create SPI flash image with custom bootloader | Oxide tools |
| Integrate AGESA and PSP firmware | Oxide tools |
| Configure kernel and rootfs | Yocto |
| Generate APCB for DDR5 training | Oxide tools (or AMD FAE) |
| Build GRUB with custom config | Yocto |

### 8.4 Complementary Workflow

**Recommended approach for V3000**:

1. Use **Oxide tools** (adapted):
   - Build PHBL for V3000
   - Integrate PSP firmware and AGESA
   - Generate SPI flash ROM

2. Use **Yocto** (optionally):
   - Build Linux kernel with V3000 support
   - Create rootfs with required applications
   - Package as CPIO archive for PHBL

3. Combine:
   - PHBL loads Yocto-built kernel/rootfs
   - Or: Use Yocto-built image with vendor UEFI

---

## 9. AMD openSIL Future Direction

### 9.1 Overview

AMD is developing **openSIL** (Open Silicon Initialization Library) to replace AGESA:
- Open-source under MIT license
- Supports coreboot, UEFI, LinuxBoot
- Eliminates proprietary initialization code

### 9.2 Timeline

| Timeframe | Milestone |
|-----------|-----------|
| June 2023 | Initial code open-sourced |
| 2023-2024 | Proof-of-concept for Turin (server) and Phoenix (client) |
| 2025-2026 | Production-ready for Zen 6 (Venice/Medusa) |
| Post-2026 | Full AGESA phase-out |

### 9.3 Implications for V3000

- **V3000 will NOT receive openSIL support** (older platform)
- V3000 development must continue using AGESA binary blobs
- openSIL is relevant for **future** AMD embedded platforms

### 9.4 Industry Partners

- Oxide Computer Company
- 9elements
- AMI
- MiTAC (Tyan)
- Supermicro

---

## 10. Recommendations

### 10.1 For V3000 Bootloader Development

**Primary Tooling: Oxide amd-host-image-builder/PHBL approach**

Rationale:
- Directly addresses PSP/AGESA integration
- Provides complete SPI ROM construction
- Proven on similar AMD platforms (Milan, Genoa)
- Open-source, adaptable

**Yocto Role**: Optional, for Linux payload construction

### 10.2 Implementation Strategy

#### Phase 1: Firmware Foundation (Use Oxide Approach)
1. Adapt amd-host-image-builder for V3000 family ID
2. Obtain V3000 PSP firmware blobs (requires AMD NDA)
3. Port PHBL for V3000 UART, GPIO addresses
4. Generate APCB for V3000 DDR5 configuration

#### Phase 2: Payload Development (Optional Yocto)
1. Create `v3000.conf` machine configuration
2. Build Linux kernel with V3000 support
3. Create minimal rootfs
4. Package as CPIO for PHBL consumption

#### Phase 3: Integration
1. Combine adapted ROM tools with Linux payload
2. Test on SolidRun HoneyComb V3000
3. Iterate on APCB parameters for memory training

### 10.3 Specific Recommendations

| Task | Recommendation |
|------|----------------|
| V3000 machine config for Yocto | Create `meta-v3000` layer based on v2000 |
| PSP directory format | Study Oxide code, adapt for V3000 |
| AGESA integration | Use amdfwtool approach from coreboot |
| Boot image construction | Adapt amd-host-image-builder |
| APCB generation | Requires AMD FAE support or reverse engineering |
| Testing | SolidRun HoneyComb V3000 as reference platform |

### 10.4 When to Use Yocto

**Use Yocto when**:
- Building production Linux distribution
- Need reproducible builds
- Want package management (RPM/DEB)
- Require license compliance tracking
- Building for multiple platforms

**Do NOT use Yocto when**:
- Constructing SPI flash firmware
- Integrating PSP/AGESA
- Building custom bootloaders like PHBL
- Working below UEFI level

### 10.5 Hybrid Approach Example

```bash
# 1. Build Linux payload with Yocto
cd yocto-build
source oe-init-build-env
MACHINE=v3000 bitbake core-image-minimal
# Output: core-image-minimal-v3000.cpio.gz

# 2. Compress for PHBL
pinprick compress core-image-minimal-v3000.cpio.gz phase1.cpio.z

# 3. Build PHBL with Yocto payload
cd ../phbl
cargo xtask build --cpioz=../yocto-build/phase1.cpio.z

# 4. Build complete ROM image
cd ../amd-host-image-builder
cargo run -- build \
    --config v3000-image.toml \
    --phbl ../phbl/target/release/phbl \
    --output v3000-flash.bin

# 5. Flash to hardware
flashrom -p linux_spi:dev=/dev/spidev0.0 -w v3000-flash.bin
```

---

## 11. V3000-Specific Gaps and Actions

### 11.1 Identified Gaps

| Gap | Source | Action Required |
|-----|--------|-----------------|
| No V3000 machine config in meta-amd | Yocto | Create new config based on v2000 |
| No SolidRun Yocto layer for V3000 | SolidRun | Not critical if using Oxide approach |
| V3000 PSP directory format unknown | AMD | Requires NDA documentation |
| V3000 AGESA blobs unavailable | AMD | Requires NDA access |
| V3000 UART/GPIO addresses unverified | AMD/SolidRun | Extract from datasheet or UEFI |
| V3000 APCB parameters unknown | AMD | Requires FAE support |

### 11.2 Information Needed from AMD

1. V3000 PSP firmware package
2. AGESA v3000PI binary
3. APCB template for HoneyComb board
4. PSP directory format documentation
5. Memory training parameters for DDR5

### 11.3 Potential Sources Without NDA

1. **Datasheet extraction**: UART/GPIO addresses from public docs
2. **UEFI reverse engineering**: Extract info from SolidRun BIOS
3. **PSPTool analysis**: Analyze existing V3000 firmware images
4. **Community resources**: coreboot mailing lists, Oxide documentation

---

## Appendix A: Key Resources

### Official Repositories
- **meta-amd**: https://git.yoctoproject.org/meta-amd
- **meta-intel**: https://git.yoctoproject.org/meta-intel
- **meta-coreboot**: https://github.com/zarhus/meta-coreboot
- **Oxide PHBL**: https://github.com/oxidecomputer/phbl
- **Oxide Helios**: https://github.com/oxidecomputer/helios

### Documentation
- **Yocto BSP Guide**: https://docs.yoctoproject.org/bsp-guide/
- **meta-amd FEATURES**: https://git.yoctoproject.org/meta-amd/tree/FEATURES.md
- **coreboot PSP Integration**: https://doc.coreboot.org/soc/amd/psp_integration.html
- **SolidRun Developer Center**: https://solidrun.atlassian.net/wiki/spaces/developer/

### Tools
- **PSPTool**: https://github.com/PSPReverse/PSPTool
- **amdfwtool**: Part of coreboot project

### Community
- **meta-amd Mailing List**: meta-amd@lists.yoctoproject.org
- **Yocto Project Bugzilla**: http://bugzilla.yoctoproject.org/
- **coreboot Mailing List**: https://mail.coreboot.org/

---

## Appendix B: Creating V3000 Yocto Machine Configuration

If building Linux payload with Yocto, create a V3000 machine configuration:

```bitbake
# conf/machine/v3000.conf

#@TYPE: Machine
#@NAME: AMD Ryzen Embedded V3000
#@DESCRIPTION: Machine configuration for AMD Ryzen Embedded V3000 series

require conf/machine/include/x86/tune-corei7.inc
require conf/machine/include/x86/x86-base.inc

MACHINEOVERRIDES =. "v3000:"

# CPU tuning for Zen 3
DEFAULTTUNE = "corei7-64"

# Machine features
MACHINE_FEATURES = "alsa bluetooth pci wifi usbhost efi"

# EFI boot provider
EFI_PROVIDER = "grub-efi"

# Kernel configuration
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
KERNEL_IMAGETYPE = "bzImage"

# Serial console for debugging
SERIAL_CONSOLES = "115200;ttyS0"

# WIC image settings
WKS_FILE = "directdisk-gpt.wks"
IMAGE_FSTYPES = "wic wic.bmap"

# AMD GPU support
MACHINE_EXTRA_RRECOMMENDS += "linux-firmware-amdgpu"
```

---

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| **AGESA** | AMD Generic Encapsulated Software Architecture - silicon initialization code |
| **APCB** | AMD PSP Customization Block - configuration for PSP |
| **BSP** | Board Support Package |
| **EFS** | Embedded Firmware Structure |
| **FSP** | (Intel) Firmware Support Package |
| **PHBL** | Pico Host Boot Loader (Oxide) |
| **PSP** | Platform Security Processor |
| **openSIL** | Open Silicon Initialization Library (AGESA replacement) |
| **WIC** | OpenEmbedded Image Creator |
| **WKS** | WIC Kickstart file |

---

*Report generated: 2025-11-19*
*For V3000 bootloader research project*
