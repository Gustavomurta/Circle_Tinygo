# Initial Analysis — TinyGo Bare Metal on Raspberry Pi 3B

Created by ChatGPT AI.

## Executive summary

It is technically feasible to run a TinyGo program without Linux on a Raspberry Pi 3B (BCM2837, four Cortex-A53 cores), but **TinyGo does not currently provide a ready-made target for this board**. The installation verified on this machine is tinygo 0.41.1 windows/amd64, with Go 1.26.5 and LLVM 20.1.1; neither the target enumeration nor the targets/ directory contains a raspberry, rpi, or pi3 target.

The first implementation must create an AArch64-specific target, a linker script, entry/exception-vector code, and a small TinyGo runtime extension. A target JSON file alone is insufficient: the generic bare-metal runtime requires linker symbols and platform-specific implementations for time, output, termination, and initialization.

The recommended choice is **AArch64**, using a flat kernel8.img image loaded at 0x80000, initially running on one core with no MMU, interrupts, or drivers beyond a future polling UART. AArch32 remains a contingency alternative, not the starting point.

Circle was not modified, no drivers were converted, and this stage contains no implementation.

## Scope, evidence, and limits

Circle was read only from D:\Gustavo\Projetos\.Circle_Master\circle-master. The local copy is an extracted ZIP without a .git directory; the CIRCLEVER constant in Rules.mk is 510101. The conclusions below come from the source and documentation in that copy, plus the official sources linked in [References](#references).

The following were also verified directly:

- tinygo version: 0.41.1 windows/amd64, Go 1.26.5, LLVM 20.1.1;
- tinygo targets: no Raspberry Pi 3/AArch64 bare-metal target;
- tinygo build -h: -target accepts either a target name **or a path to a JSON target specification**;
- the installed distribution includes targets/nintendoswitch.json, which is ARM64 but is neither a Raspberry Pi target nor a generic bare-metal target.

LLVM accepting aarch64 must not be treated as board compatibility: boot, memory layout, exceptions, timers, and MMIO remain the target's responsibility.

## 1. Raspberry Pi 3B initialization in Circle

### AArch64 path used as the reference

| Circle file | Observed responsibility | Relevance to the port |
| --- | --- | --- |
| boot/config64.txt | Selects arm_64bit=1, kernel_address=0x80000 and, in [pi3], kernel=kernel8.img. | Defines the boot contract our image must follow. |
| Rules.mk | For AARCH=64 and RASPPI=3, selects -mcpu=cortex-a53, TARGET=kernel8, and LOADADDR=0x80000. | Confirms CPU, image name, and load address. |
| circle.ld | Declares ENTRY(_start), orders sections, and exports BSS boundaries. | A linker model; the TinyGo script will need its own symbols. |
| lib/startup64.S | Implements _start; handles EL, sets sp_el1 and VBAR_EL2, drops from EL2 to EL1, sets sp_el0 and VBAR_EL1, then branches to sysinit. | Primary reference for exception-level transition and stacks. |
| lib/exceptionstub64.S | Builds a 2 KiB-aligned vector table with exception slots and IRQ/FIQ context preservation. | Reference for VBAR_EL1 requirements; not code to copy. |
| lib/sysinit.cpp | Clears .bss and .tbss, checks the image limit, initializes memory/interrupts, calls constructors, then calls the application. | Establishes the ordering: static memory must be valid before the runtime/application. |
| include/circle/memorymap64.h and doc/memorymap.txt | For Pi 3 AArch64: image at 0x80000, stacks after it, a coherent region, and a later heap; four cores. | Memory-map analysis basis, not a map to copy verbatim. |
| lib/multicore.cpp and include/circle/bcm2836.h | Starts secondary cores through the local spin table using SEV and cache/barriers; local peripherals start at 0x40000000. | Justifies parking cores 1–3 and deferring multicore support. |
| include/circle/bcm2835.h | For RASPPI <= 3, sets main MMIO at 0x3f000000; defines GPIO, PL011, mailbox, and interrupt controller. | Reference for BCM2837 addresses to validate against hardware documentation. |

The comment in lib/startup64.S identifies the usual entry path as firmware/ARM stub in EL2. It also handles an entry already in EL1. Our entry must have the same defensive posture: read CurrentEL, mask exceptions, establish aligned stacks, and only then transfer control to the runtime.

In Circle, sysinit clears BSS before constructing objects or calling the application. In our flat image, initialized .data bytes are already present in the firmware-loaded image; .bss/NOLOAD must be cleared explicitly. The Raspberry Pi will not load an ELF file: kernel8.img must be raw binary.

### AArch32 equivalent, reviewed for comparison

| File | Relevant difference |
| --- | --- |
| boot/config32.txt | Selects arm_64bit=0; on Pi 3, firmware can load kernel7.img or an explicitly selected image. |
| Rules.mk | With AARCH=32 and RASPPI=3, uses -mcpu=cortex-a53 -marm -mfpu=neon-fp-armv8, TARGET=kernel8-32, and LOADADDR=0x8000. |
| lib/startup.S | Switches ARM modes (FIQ, IRQ, abort, system), creates a stack per mode, then calls sysinit; there is no AArch64 EL2 → EL1 transition. |
| lib/exceptionstub.S and lib/cache-v7.S | Implement 32-bit vectors/CP15 and cache support. |
| include/circle/memorymap.h | Uses 4 KiB pages, 32-bit/LPAE tables, and MEM_KERNEL_START=0x8000. |

## 2. CPU, memory, stack, and linker requirements

### AArch64 boot contract

Raspberry Pi firmware is still required — “bare metal” means without Linux, not without GPU/boot-ROM firmware. The boot FAT partition must contain compatible official firmware, config.txt, and kernel8.img; there will be no root partition or Linux kernel. Official documentation confirms kernel8.img as the Pi 3 64-bit kernel image and arm_64bit=1 as the mode selector.

The minimum AArch64 bootstrap must:

1. Enter through _start at 0x80000, read CurrentEL, and mask DAIF.
2. Keep cores 1–3 in a safe WFE loop. The first proof of life must not use the spin table, local mailbox, or a multicore scheduler.
3. If it enters in EL2, grant EL1 access to the generic timer, configure HCR_EL2 for AArch64 EL1, and use ERET to enter EL1. If it already enters in EL1, it must not repeat that transition.
4. Create a bootstrap stack with 16-byte alignment; later separate the exception stack (EL1h) from the application stack (EL1t/SP_EL0), or deliberately maintain an EL1h convention during the first milestone.
5. Install a 2 KiB-aligned AArch64 vector table in VBAR_EL1. Before IRQ is enabled, all 16 slots must lead to a safe handler that stops and, once UART exists, reports ESR/ELR/FAR.
6. Clear .bss, preserve .data, configure heap boundaries, and invoke the TinyGo runtime entry point, which runs Go initializers before main.main.

During the first milestone, leave MMU, I-cache, D-cache, and IRQ/FIQ disabled after explicit architectural initialization and the appropriate barriers. This allows physical/identity MMIO and reduces failure variables. Enabling the MMU later will require MAIR_EL1, TCR_EL1, TTBR0_EL1, page tables, distinct attributes for normal RAM and Device-nGnRE, TLB/cache invalidation, and DSB/ISB.

### Memory and required symbols

The custom AArch64 linker script must materialize at least:

~~~text
0x00080000  .text.boot / _start and vector table
            .text, .rodata, .data
            .bss (NOLOAD; cleared by bootstrap)
            reserved stacks (NOLOAD, aligned)
            static heap [ _heap_start, _heap_end )
~~~

In addition to BSS bounds, it must export at least _heap_start, _heap_end, _globals_start, _globals_end, and _stack_top. The installed TinyGo src/runtime/baremetal.go declares these symbols externally: they define the fixed heap and globals inspected by the runtime. Their names and semantics must be checked against the pinned fork commit rather than invented.

_heap_end cannot be fixed at 1 GiB. Pi 3B RAM is shared with the GPU, and the ARM allocation varies with gpu_mem/firmware. The first implementation must obtain the ARM memory region through the property mailbox (GET_ARM_MEMORY, exposed in Circle as PROPTAG_GET_ARM_MEMORY) before releasing heap memory, or use a documented, conservative temporary limit. The image, stacks, an aligned mailbox buffer, and future DMA memory must remain outside the heap.

## 3. TinyGo and the need for a custom target

### What actually exists

The installed TinyGo has an ARM64 backend: src/runtime/arch_arm64.go, src/runtime/asm_arm64.S, and targets/nintendoswitch.json exist. That target, however, specifies cpu: cortex-a57, a PIC model, and Nintendo Switch Horizon calls. It does not include Raspberry Pi boot code, BCM2837 MMIO, UART, exception vectors, or the Raspberry Pi firmware memory contract; reusing it would be incorrect.

Bare-metal Cortex-M targets are useful only as a structural pattern. Their JSON specifies build tags, linker, compiler-rt, picolibc, a script, and extra assembly; a target-specific runtime such as src/runtime/runtime_atsamd51.go then exports Reset_Handler, calls preinit()/run(), and supplies clock, serial, and termination support. A Cortex-M reset vector does not apply to Cortex-A53.

### Decision

**A custom target is required.** It will have two pieces pinned to the same TinyGo revision:

1. an rpi3b-aarch64 target JSON, linker script, and boot/vector assembly;
2. an extension to TinyGo's internal runtime package, selected by the rpi3baremetal tag, for entry, heap, time, putchar, waiting, and critical/atomic operations on AArch64.

This requires a **TinyGo fork** or an upstream contribution. Go code in the application repository cannot add files to the standard runtime package compiled from TINYGOROOT. A project-local JSON can reference build artifacts, but it cannot replace the internal Go runtime.

The JSON should follow the existing bare-metal pattern, using implementation-time-validated values: an AArch64 LLVM target, cortex-a53 CPU, goarch: arm64, ld.lld, compiler-rt, picolibc, baremetal/arm64/rpi3baremetal tags, and custom script/assembly. goos: linux may only serve standard-library selection compatibility, as it does for Cortex-M targets; the baremetal tag excludes code paths that require Linux. It does not place Linux in the image.

The proposed initial profile is -scheduler=none -gc=leaking:

- scheduler=none removes goroutines/channels and reduces variables before timers and IRQs;
- gc=leaking provides simple allocation without freeing; it suits only short tests with a heap budget;
- gc=none is useful for locating allocations, but fails to link wherever allocation is required;
- conservative GC should wait until AArch64 stacks, exceptions, and global symbols have been validated.

## 4. AArch64 compared with AArch32

| Criterion | AArch64 (recommended) | AArch32 (contingency) |
| --- | --- | --- |
| Circle image/entry | kernel8.img at 0x80000; usual EL2 entry and transition to EL1. | Normally kernel7.img, or kernel8-32.img; 0x8000 entry, classic ARM modes. |
| Pi 3B CPU | Native 64-bit Cortex-A53 use, x0–x30 registers. | Same chip in 32-bit mode, 32-bit registers, VFP/NEON, and CPSR. |
| Circle reference | startup64.S, exceptionstub64.S, memorymap64.h. | startup.S, exceptionstub.S, cache-v7.S, memorymap.h. |
| Verified TinyGo state | ARM64 backend, but no Pi bare-metal target. | Many Cortex-M targets, but no Pi/A-profile target; their startup and vectors do not fit A53. |
| Complexity | Requires EL2/EL1 and a 2 KiB vector table, but uses the target architecture natively. | Avoids the AArch64 transition, but adds CPU modes, a 32-bit ABI, and still requires a custom target. |
| Decision | Main path. | Plan B only if a concrete TinyGo ARM64 limitation is demonstrated. |

There is no evidence of direct BCM2837 support in either architecture. AArch32 does not remove the need for boot, linker, runtime, and machine support.

## 5. Windows 11 tools

### State confirmed on this machine

| Tool | Status | Use |
| --- | --- | --- |
| TinyGo 0.41.1 | Installed through Scoop. | Study baseline; it will be replaced or run from the pinned fork. |
| Go 1.26.5 | Installed. | Build the TinyGo fork. |
| Git 2.49.0 | Installed. | Clone/pin TinyGo and record provenance. |
| OpenOCD 0.12.0 | Installed. | Later JTAG work, with a probe and Pi configuration. |
| CMake 4.0.1, Python, and mingw32-make | Installed. | Foundation for source builds of TinyGo/LLVM. |
| gdb-multiarch | Installed. | AArch64 ELF debugging, to be validated with JTAG. |
| aarch64-none-elf-* and qemu-system-aarch64 | Not found in PATH. | Recommended for objdump/GDB and boot testing. |

The make.exe found in PATH is Borland Delphi's Make, not GNU Make. It must not be used to build TinyGo/Circle. Building the fork should use Git Bash/MSYS2 with GNU Make, CMake, Ninja, and Python. TinyGo's manual Windows build documentation requires a Unix-like environment such as Git Bash.

An Arm GCC toolchain is not mandatory to **compile the image** if the fork remains configured for TinyGo-provided ld.lld/compiler-rt. Nevertheless, Arm GNU Toolchain with the aarch64-none-elf- prefix is recommended for objdump, GDB, and comparison with Circle. Do not use aarch64-linux-gnu, which targets hosted binaries.

Use a .bin output, for example tinygo build -o build/kernel8.bin ..., and publish that file as kernel8.img only after validation. TinyGo uses the extension to select its output format: .bin is raw binary, while a generic extension such as .img produces a regular executable/ELF. This prevents placing an ELF on the SD card while expecting firmware to load a flat image.

Hardware required later: a writable microSD card and reader, stable Pi 3B power supply, and a 3.3 V USB-to-TTL UART adapter for the PL011 console. JTAG additionally requires a compatible probe and physical wiring, beyond the first boot milestone.

## 6. Proposed initial architecture

~~~text
.Circle_Tinygo/
  ANALISE_INICIAL.md
  docs/                       # decisions, memory map, procedures
  toolchain/tinygo-rpi3/      # pinned TinyGo fork (submodule or clone)
    targets/rpi3b-aarch64.json
    targets/rpi3b-aarch64.ld
    targets/rpi3b-boot.S
    targets/rpi3b-vectors.S
    src/runtime/runtime_rpi3b_aarch64.go
  platform/rpi3b/             # small public Go API
    mmio.go                   # volatile access and barriers
    mailbox.go                # later: ARM RAM discovery
    uart.go                   # later: polling PL011
    timer.go                  # later: generic timer
  cmd/hello/                  # minimal acceptance program
  boot/                       # config templates/firmware file list
  scripts/                    # build, ELF/BIN validation, manual SD copy
~~~

The fork must be explicit and pinned, not a silent copy of the installation under C:\Users\...\tinygo. Applications should consume platform/rpi3b APIs and never import runtime, assembly, or linker symbols. No Circle driver belongs in this initial architecture.

## 7. Implementation plan by stage

### Stage 0 — provenance and environment

1. Create a pinned TinyGo fork and record Go/LLVM versions.
2. Record the Raspberry Pi firmware version used on the test card.
3. Prepare a script that produces a debug ELF and raw BIN, checks type, entry, and size, and does not write cards automatically.

**Acceptance:** reproducible artifacts and a map report; no boot yet.

### Stage 1 — AArch64 target, one-core boot, and flat image

1. Add the target JSON, linker, _start, secondary-core parking, BSS clearing, stack, and minimal vector table.
2. Integrate only the runtime entry support needed to reach main.main; leave MMU/cache/IRQ disabled.
3. Produce raw kernel8.img and test first in QEMU raspi3b when available, then from physical SD media.

**Acceptance:** the Pi reaches main; a safe exception handler exists; the layout does not overlap reserved memory.

### Stage 2 — console and minimal time support

1. Implement polling PL011, GPIO 14/15, and runtime.putchar.
2. Expose println/panic through the console and read the generic counter with a validated scale.
3. Keep scheduler=none and gc=leaking; measure heap consumption.

**Acceptance:** cmd/hello prints a signature, version, and current EL through UART; a controlled panic does not corrupt state.

### Stage 3 — ARM RAM, exceptions, and interrupts

1. Query the mailbox for ARM RAM and configure the real heap.
2. Implement handlers with ESR/ELR/FAR reporting and the BCM2837 interrupt controller.
3. Configure the generic timer and only then evaluate the tasks scheduler.

**Acceptance:** an intentional exception generates diagnostics, the timer does not lose ticks, and the heap respects GPU memory.

### Stage 4 — MMU/cache and driver foundation

1. Design tables/attributes and enable MMU/cache with barrier tests.
2. Convert **one** peripheral at a time into an independent Go driver, using BCM2837 documentation and Circle only as behavioral reference.
3. Only afterwards assess GPIO, I2C, SPI, framebuffer, SD, and DMA.

**Acceptance:** RAM and MMIO have distinct attributes; each driver has an isolated hardware test.

### Stage 5 — multicore, GC, and stability

1. Wake secondary cores using the spin table, cache maintenance, and correct barriers.
2. Validate atomics, per-core stacks, and conservative GC.
3. Make QEMU, UART, and JTAG part of documented regression testing.

**Acceptance:** four cores run controlled workloads without corruption, and the target is reproducible on hardware.

## 8. Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| No ready TinyGo Pi 3 AArch64 support. | High: no supported-board fallback. | Pinned fork, minimal target/runtime, validation at every milestone. |
| TinyGo/runtime internal ABI changes. | High: failure may occur before main. | Pin the revision; treat upgrades as tested migrations. |
| Confusing ELF with a flat image. | High: firmware will not start the application. | Generate .bin, rename only after magic/layout checks, retain ELF for GDB. |
| Initial EL/cache/core state varies by firmware/emulator. | High: silent lockups. | Read CurrentEL, mask exceptions, park secondary cores, use barriers, test on real Pi. |
| Heap overlaps GPU, mailbox, stacks, or DMA. | High: non-deterministic corruption. | Query ARM RAM, reserve regions in linker, defer DMA. |
| UART depends on firmware clock/configuration. | Medium: diagnostics unavailable. | Polling PL011, documented clock setup, JTAG fallback. |
| QEMU does not emulate every Pi 3 peripheral. | Medium: false positives. | Use QEMU only for boot/EL/linker; accept on physical hardware. |
| Copying Circle assembly/drivers. | Legal and technical: local Circle is GPLv3 and this stage prohibits conversion. | Implement independently; if derivation occurs, preserve licensing and review distribution. |
| Assuming Go packages behave like hosted Go. | Medium: syscalls, files, and parts of stdlib need platform support. | Begin with a minimal runtime and own APIs; expand only after tests. |

## References

- Local Circle sources: README.md, Rules.mk, circle.ld, boot/README, boot/config64.txt, lib/startup64.S, lib/sysinit.cpp, lib/exceptionstub64.S, lib/multicore.cpp, include/circle/memorymap64.h, and doc/memorymap.txt. Repository: https://github.com/rsta2/circle.
- [Raspberry Pi configuration and kernel files](https://www.raspberrypi.com/documentation/computers/configuration.html).
- [Raspberry Pi 3 in AArch64/EL2 — Arm Trusted Firmware](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/plat/rpi3.rst).
- [TinyGo Windows installation](https://tinygo.org/getting-started/install/windows/), [build and output formats](https://tinygo.org/docs/reference/usage/subcommands/), and [targets, GC, and scheduler](https://tinygo.org/docs/reference/usage/important-options/).
- [TinyGo manual Windows build](https://tinygo.org/docs/guides/build/manual-llvm/) and [bare-metal additional requirements](https://tinygo.org/docs/guides/build/additional-requirements/).

