# GBA machine for QEMU — architecture

## Goal

A full-accuracy Game Boy Advance machine type (`-M gba`) inside upstream QEMU (`qemu-system-arm`), built as a new machine + a set of new devices, following existing QEMU device-model conventions rather than inventing new infrastructure.

## Key CPU decision

QEMU no longer ships an ARM7TDMI (pure ARMv4T) CPU model — the oldest surviving core is `arm926` (ARMv5TEJ). The closest analog, `ti925t` in `target/arm/tcg/cpu32.c`, sets only `ARM_FEATURE_V4T`, which is the exact ISA subset GBA software needs (ARMv4T: ARM + Thumb, no Thumb-2, no VFP). QEMU's translator (`target/arm/tcg/translate.c`) still fully implements ARMv4T behind that feature flag — no decoder changes needed, only a new CPU *definition*.

CP15 (coprocessor 15 / MMU control) cannot be fully removed from an A/R-profile CPU in this QEMU version — `register_cp_regs_for_features()` in `target/arm/cpu.c` only skips CP15 for `ARM_FEATURE_M` (Cortex-M profile). This is harmless: GBA software never executes `MRC`/`MCR p15`, so a dormant, unused CP15 has no observable effect on emulation correctness.

## Memory map (GBA-accurate)

| Region | Base | Size | Notes |
|---|---|---|---|
| BIOS ROM | `0x0000_0000` | 16 KiB | read-only, user-supplied image (copyrighted, not redistributed) |
| EWRAM (board WRAM) | `0x0200_0000` | 256 KiB | 2-wait-state 16-bit bus (timing modeled in a later pass) |
| IWRAM (chip WRAM) | `0x0300_0000` | 32 KiB | 0-wait-state 32-bit bus |
| I/O registers | `0x0400_0000` | ~0x400 | PPU/APU/DMA/timer/keypad/serial/intc registers |
| Palette RAM | `0x0500_0000` | 1 KiB | BG (512B) + OBJ (512B) |
| VRAM | `0x0600_0000` | 96 KiB | mode-dependent layout |
| OAM | `0x0700_0000` | 1 KiB | 128 sprite entries |
| Cartridge ROM | `0x0800_0000`–`0x0DFF_FFFF` | up to 32 MiB | mirrored 3x (wait-state regions 0/1/2) |
| Cartridge backup | `0x0E00_0000` | up to 128 KiB | SRAM / Flash / EEPROM (type auto-detected from ROM header) |

## Phase plan

Each phase produces a runnable, testable machine — full accuracy is only verifiable incrementally.

- **Phase 0 — Docs** (this folder). Done once, updated continuously.
- **Phase 1 — ARM7TDMI CPU core.** New `arm7tdmi_initfn` in `target/arm/tcg/cpu32.c` (modeled on `ti925t_initfn`): `ARM_FEATURE_V4T` only, MIDR `0x41007700`. New `ARM_CPUID_ARM7TDMI` in `target/arm/cpu.h`.
- **Phase 2 — Machine skeleton.** New `hw/arm/gba.c` following `hw/arm/integratorcp.c`'s CPU/memory-map wiring pattern. BIOS/EWRAM/IWRAM/cart-ROM mapped, no devices yet. `hw/arm/Kconfig` + `hw/arm/meson.build` updated. `DEFINE_MACHINE("gba", ...)`. **Done.** Design note: reuses QEMU's existing generic `-bios`/`-kernel` options (`machine->firmware` / `machine->kernel_filename`, following the pattern in `hw/m68k/mcf5208.c`) for the BIOS and cartridge images instead of a bespoke `-M gba,cart=...` property — no new option-parsing code needed. Cart ROM is sized to the loaded file (not a fixed 32MiB), with the wait-state-1/2 windows (`0x0A000000`, `0x0C000000`) as `memory_region_init_alias` mirrors of the same backing ROM.
- **Phase 3 — Interrupt controller + keypad.** `hw/intc/gba_intc.c` (flat IE/IF/IME merge, modeled on `hw/intc/imx_avic.c`). `hw/input/gba_keypad.c` (KEYINPUT/KEYCNT).
- **Phase 4 — Timers.** `hw/timer/gba_timer.c`, 4x `ptimer`-based counters (pattern: `hw/timer/cmsdk-apb-timer.c`), manual cascade linking between instances.
- **Phase 5 — DMA.** `hw/dma/gba_dma.c`, 4 channels, immediate/VBlank/HBlank/special start modes. DMA timing (cycle-stealing) deferred to a later accuracy pass — tracked explicitly in `PROGRESS.md`, not silently dropped.
- **Phase 6 — PPU/display.** `hw/display/gba_ppu.c` (pattern: `hw/display/pl110.c` + `hw/display/g364fb.c`), scanline-timer-driven HBlank/VBlank, all 6 video modes, sprites, windowing, blending.
- **Phase 7 — Audio (APU).** `hw/audio/gba_apu.c` (pattern: `hw/audio/pcspk.c`'s `audio_be_*` pull-callback API), 4 legacy GB channels + 2 direct-sound PCM FIFOs.
- **Phase 8 — Serial I/O + cartridge backup.** `hw/char/gba_sio.c` (General-Purpose/Normal modes; Multiplayer/UART best-effort, no real link partner). `hw/block/gba_cart_backup.c` (SRAM/Flash/EEPROM, auto-detected, persisted via a QEMU block device).
- **Phase 9 — Integration polish.** gdbstub check, machine option surface (`-M gba,bios=...`, cart/save paths, save-type override), `-M help`/`-cpu help` listing.

See `PROGRESS.md` for current status, `SETUP.md` for the build environment, `REFERENCES.md` for the QEMU in-tree patterns and GBA hardware references this design is based on.
