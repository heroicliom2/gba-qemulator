# Progress tracker

Update this file at the end of every phase. This is the resumability mechanism for future sessions/agents — read this first when picking the project back up.

## Status: Phase 1 complete, ready to start Phase 2

- [x] WSL2 Ubuntu 22.04 toolchain installed: git, gcc, ninja, pkg-config, meson (user-local at `~/.local/bin`), libglib2.0-dev, libpixman-1-dev, libfdt-dev, zlib1g-dev, libslirp-dev, libcap-ng-dev, libattr1-dev, python3-pip.
- [x] QEMU cloned (shallow, `--depth 1` from `https://gitlab.com/qemu-project/qemu.git`) into `c:\Users\Musa\Documents\gameboy\qemu`, branch `gba-machine` created off `origin/master` (commit `35500e5c41` at clone time).
- [x] Confirmed via research: no ARM7TDMI CPU model exists in this QEMU tree; `ti925t` is the closest ARMv4T analog; CP15 cannot be fully disabled for A/R-profile cores but is harmless for GBA software.
- [x] Docs folder created (`ARCHITECTURE.md`, `PROGRESS.md`, `SETUP.md`, `REFERENCES.md`).
- [x] **Build environment resolved**: this QEMU checkout requires Python >= 3.12, but WSL Ubuntu 22.04's system Python is 3.10 and no sudo-free apt package provides 3.12. Fixed with a dedicated conda env: `~/anaconda3/bin/conda create -n qemu312 python=3.12 -y`, then QEMU's configure is invoked with `--python=$HOME/anaconda3/envs/qemu312/bin/python3`. See `SETUP.md`.
- [x] **Phase 1 — ARM7TDMI CPU core** (`target/arm/tcg/cpu32.c`: `arm7tdmi_initfn` added right before `ti925t_initfn`, registered in `arm_tcg_cpus[]` as `"arm7tdmi"`; `target/arm/cpu.h`: `ARM_CPUID_ARM7TDMI` added next to `ARM_CPUID_TI925T`). Sets only `ARM_FEATURE_V4T`, MIDR `0x41007700`, `reset_sctlr = 0x00000070` (matches the convention used by `ti925t`/`sa1100` for pre-v6 cores).
  - Built successfully: `meson`/`ninja` via `configure --target-list=arm-softmmu --python=<conda python 3.12>` then `ninja -C build qemu-system-arm` (see `SETUP.md` for the exact commands; the first full build takes long enough that it can hit a 10-minute tool timeout partway through — just re-run `ninja -C build qemu-system-arm`, it resumes incrementally).
  - Verified: `./build/qemu-system-arm -cpu help` lists `arm7tdmi`; `./build/qemu-system-arm -M none -cpu arm7tdmi -S -nographic -monitor none -serial none` starts and idles without crashing (no CPU-reset/instantiation errors).
  - Committed on `gba-machine` branch: `66fcfbf1ef` "target/arm: add arm7tdmi CPU type for GBA machine support". WSL git identity was unset and had to be configured to match the Windows-side identity (`heroiclion2` / `heroiclion2@gmail.com`) before committing — see `SETUP.md`.
  - Pushed to GitHub: created `https://github.com/heroicliom2/gba-qemu` (public), unshallowed the clone (full upstream QEMU history, `git fetch --unshallow origin` — a shallow clone can't be pushed cleanly to a fresh repo), then pushed the `gba-machine` branch there. `origin` remains upstream `qemu-project/qemu` on GitLab (read-only reference, never push there); `github` remote added pointing at `https://github.com/heroicliom2/gba-qemu.git` for this fork. See `SETUP.md` for the credential-bridging steps used (WSL has no `gh` installed, so a token was bridged from the Windows-side authenticated `gh` CLI for the one-time push).
- [ ] Phase 2 — Machine skeleton (`hw/arm/gba.c`, Kconfig, meson.build) — **next up**
- [ ] Phase 3 — Interrupt controller + keypad
- [ ] Phase 4 — Timers
- [ ] Phase 5 — DMA controller
- [ ] Phase 6 — PPU/display controller
- [ ] Phase 7 — Audio (APU)
- [ ] Phase 8 — Serial I/O + cartridge backup
- [ ] Phase 9 — Integration polish

## Known deferred items (tracked, not forgotten)

- DMA cycle-stealing / bus timing accuracy — Phase 5 lands functional DMA first; cycle-exact CPU-stall timing is a follow-up pass once the machine is otherwise complete.
- EWRAM/IWRAM wait-state timing — memory regions are functionally correct from Phase 2, but exact wait-state cycle costs are not modeled until a timing pass.
- SIO Multiplayer/UART modes — best-effort register-level behavior only; no second QEMU instance link-cable support planned initially.
- Real BIOS is user-supplied at every phase (copyrighted, not redistributed, not reimplemented).

## Notes for whoever resumes this

- All QEMU source work happens in `c:\Users\Musa\Documents\gameboy\qemu` (Windows path) which is `/mnt/c/Users/Musa/Documents/gameboy/qemu` from WSL. This is intentional — the user wants everything visible/editable from VSCode on the Windows side, even though building on `/mnt/c` is slower than native WSL ext4.
- See `SETUP.md` for the exact invocation pattern needed to run WSL commands reliably from a Windows-side agent session (there's a Git-Bash/MSYS path-mangling gotcha).
