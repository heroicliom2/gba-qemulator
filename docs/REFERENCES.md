# Reference material

## QEMU in-tree patterns this design is based on

Found via research into this specific checkout (`c:\Users\Musa\Documents\gameboy\qemu`, upstream `master` as of clone time, commit `35500e5c41`). Line numbers may drift as upstream moves; re-grep if a referenced function has moved.

| Need | Reference file | What to copy |
|---|---|---|
| CPU definition pattern for a feature-limited ARM core | `target/arm/tcg/cpu32.c` (`ti925t_initfn`, ~line 691) | Set only the feature flags a real ARM7TDMI has (`ARM_FEATURE_V4T`); register in `arm_tcg_cpus[]` |
| Machine init: CPU creation, ROM/RAM mapping, custom-device IRQ wiring into the CPU | `hw/arm/integratorcp.c` (`integratorcp_init`, ~line 593) | `object_new(machine->cpu_type)` → `qdev_realize` → `ARM_CPU()` cast; `memory_region_init_rom`/`_ram` + `memory_region_add_subregion`; `qdev_get_gpio_in(DEVICE(cpu), ARM_CPU_IRQ)` (`ARM_CPU_IRQ` = 0, in `target/arm/cpu-qom.h`) as the IRQ sink for a custom PIC device |
| Machine registration | `include/hw/core/boards.h` (`DEFINE_MACHINE` macro, ~line 537); `hw/arm/Kconfig` + `hw/arm/meson.build` (see `CONFIG_INTEGRATOR` entries) | `config GBA` Kconfig stanza (`depends on TCG && ARM`); one line in `meson.build`: `arm_common_ss.add(when: 'CONFIG_GBA', if_true: files('gba.c'))`; `DEFINE_MACHINE("gba", gba_machine_init)` at the bottom of `gba.c` |
| Display controller: register MemoryRegion + separate VRAM MemoryRegion, both sysbus-mapped | `hw/display/pl110.c`, `hw/display/g364fb.c` | `memory_region_init_io` for registers via `sysbus_init_mmio`; `memory_region_init_ram` for VRAM as a second `sysbus_init_mmio` region; machine maps both with `sysbus_mmio_map` at fixed addresses |
| Display controller: console + periodic redraw | `hw/display/pl110.c` (`pl110_gfx_ops`, `pl110_update_display`) | `qemu_graphic_console_create(dev, 0, &ops, s)`; `GraphicHwOps{.invalidate, .gfx_update}`; `qemu_console_surface`/`qemu_console_resize`/`dpy_gfx_update` |
| Display controller: periodic HBlank/VBlank IRQ | `hw/display/pl110.c` (`pl110_vblank_interrupt`) | `timer_new_ns(QEMU_CLOCK_VIRTUAL, cb, s)`; callback sets a status bit, re-arms via `timer_mod`, and does `qemu_irq_raise`/`lower` gated by an interrupt-mask register |
| Flat interrupt merger (IE/IF/IME-style) | `hw/intc/imx_avic.c` | `qdev_init_gpio_in(dev, set_irq_cb, N)` — one GPIO per source; GPIO callback sets/clears a pending bit; a single `_update()` function computes `pending & enabled` and calls `qemu_set_irq` once; also called from the MMIO write handler |
| Countdown timer with overflow IRQ | `hw/timer/cmsdk-apb-timer.c` | `ptimer_init` (`include/hw/core/ptimer.h`) with `PTIMER_POLICY_*` flags; `ptimer_set_period`/`ptimer_set_freq`; all mutators wrapped in `ptimer_transaction_begin/commit`; overflow callback sets an IRQ status bit. No native cascade support in QEMU's `ptimer` — chain manually between device instances. |
| Audio output | `hw/audio/pcspk.c` | **This QEMU version uses the newer `audio_be_*` API, not the old `AUD_*` API** (`AUD_register_card`/`AUD_open_out` no longer exist). Use `audio_be_open_out(...)`, a pull-based callback that synthesizes samples on demand, `audio_be_write(...)`, `audio_be_set_active_out(...)`; property via `DEFINE_AUDIO_PROPERTIES(State, audio_be)` |

## GBA hardware reference material (public documentation, for register-level accuracy)

- **GBATEK** — the de facto complete GBA/NDS hardware reference (memory map, all I/O registers, PPU modes, APU, DMA, timers, SIO, BIOS function behavior). Primary reference for exact register bit layouts throughout every phase.
- **TONC** — a GBA homebrew programming tutorial series with accompanying freely-distributable demo ROMs covering tiled/affine backgrounds, sprites, DMA, timers, and sound. Used as the test-ROM source throughout `PROGRESS.md`'s verification steps — no copyrighted commercial ROMs are used.
- **mGBA / NanoBoyAdvance** — existing accurate open-source GBA emulators, useful as behavioral cross-references (screenshot/waveform comparison) when validating PPU/APU output, not as source-code references to copy from.

## Legal note

The real GBA BIOS is copyrighted Nintendo firmware. It is never bundled, redistributed, or reimplemented byte-for-byte in this project — the machine always requires the user to supply their own dump via `-bios`.
