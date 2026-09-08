# GBA QEMUlator

A full-accuracy Game Boy Advance machine type (`-M gba`) built as a new machine + set of devices inside [QEMU](https://www.qemu.org/), targeting `qemu-system-arm`.

## Status

**Phase 1 of 9 complete** — see [`docs/PROGRESS.md`](docs/PROGRESS.md) for the live checklist. A new `arm7tdmi` CPU type has been added and verified building/booting; the machine itself (memory map, PPU, APU, DMA, timers, interrupt controller, keypad, serial, cartridge backup) is still to come.

## Repo layout

- **[`docs/`](docs)** — the project's design and process documentation, kept up to date across sessions:
  - [`ARCHITECTURE.md`](docs/ARCHITECTURE.md) — memory map, CPU decisions, the full 9-phase build plan
  - [`PROGRESS.md`](docs/PROGRESS.md) — current status, what's done, what's deferred
  - [`SETUP.md`](docs/SETUP.md) — build environment (WSL2/Ubuntu toolchain, Python version pinning, build commands, git/GitHub setup notes)
  - [`REFERENCES.md`](docs/REFERENCES.md) — the existing QEMU device patterns this design reuses, plus GBA hardware documentation pointers
- **[`qemu/`](qemu)** — a git submodule pointing at [heroicliom2/gba-qemu](https://github.com/heroicliom2/gba-qemu) (branch `gba-machine`), a fork of upstream QEMU with the GBA machine work applied. See that repo's own README for QEMU-specific build instructions.

## Getting the code

```bash
git clone --recurse-submodules https://github.com/heroicliom2/gba-qemulator.git
```
(or `git submodule update --init` after a plain clone)

## Why a fork of QEMU rather than a from-scratch emulator

QEMU already provides a mature ARM instruction-set emulator (TCG), device model, display/audio backends, and debugging tooling (gdbstub, tracing). Building the GBA as a new QEMU machine type reuses all of that instead of reimplementing a CPU interpreter and UI layer from zero — see `docs/ARCHITECTURE.md` for the specific existing QEMU devices each new GBA component is modeled on.

## Legal

QEMU is licensed primarily under GPLv2 (with some LGPLv2.1/BSD-licensed files) — this project's changes to it are a GPLv2 derivative work, same as any QEMU contribution. The real GBA BIOS and any commercial ROMs are Nintendo's copyrighted property and are never bundled, redistributed, or reimplemented here; the machine always requires the user to supply their own BIOS dump via `-bios`.
