# Build environment setup

## WSL distro

WSL2, distro `Ubuntu` (22.04 LTS "jammy"). Check with `wsl.exe -l -v` from Windows.

## Installed packages

Via `apt`: `git build-essential ninja-build pkg-config libglib2.0-dev libpixman-1-dev python3-pip python3-venv flex bison libfdt-dev zlib1g-dev libslirp-dev libcap-ng-dev libattr1-dev`

Via `python3 -m pip install --user meson` (Ubuntu 22.04's apt `meson` is too old for current QEMU; the pip-installed one lands at `~/.local/bin/meson`, version 1.12.0 at install time).

**`~/.local/bin` must be on `PATH`** for `meson` to resolve in non-interactive shells — this was appended to `~/.bashrc`:
```bash
export PATH="$HOME/.local/bin:$PATH"
```
For any one-off script invocation that doesn't source `.bashrc` in a way that picks this up, either call `~/.local/bin/meson` directly or `source ~/.bashrc` first.

## Repo location

Cloned at `/mnt/c/Users/Musa/Documents/gameboy/qemu` (i.e. `c:\Users\Musa\Documents\gameboy\qemu` from Windows) — deliberately on the Windows-mounted filesystem, not WSL's native ext4, so the tree is visible/editable in VSCode. This makes builds slower than native WSL (`/mnt/c` goes through 9p/DrvFs), but that's the accepted tradeoff.

Branch: `gba-machine`, based on `origin/master`. Clone is shallow (`--depth 1`) — no full history. If full history is ever needed: `git fetch --unshallow` from within the repo (large download, upstream QEMU history is big).

## Python version requirement

This QEMU checkout (11.1.50-dev, Sept 2026 master) requires **Python >= 3.12**. WSL Ubuntu 22.04's system Python is 3.10, and there is no `python3.12` in the default apt repos without adding a PPA (which needs sudo we don't have in this session). Fix used: create an isolated conda env (no sudo needed, `anaconda3` was already installed for the user):
```bash
~/anaconda3/bin/conda create -n qemu312 python=3.12 -y
```
This gives `~/anaconda3/envs/qemu312/bin/python3`. Pass it to QEMU's `configure` via `--python=`.

## Build commands

QEMU's own `configure` script (not a raw `meson setup`) is what accepts `--target-list`; it invokes meson internally. From `/mnt/c/Users/Musa/Documents/gameboy/qemu`:
```bash
mkdir -p build && cd build
../configure --target-list=arm-softmmu --python=$HOME/anaconda3/envs/qemu312/bin/python3
ninja qemu-system-arm
```
Only `arm-softmmu` is needed (`qemu-system-arm` binary) — no need to build every QEMU target.

Resulting binary: `build/qemu-system-arm`.

**Note on build time**: a full first build compiles ~2150 objects and can take longer than a single 10-minute background-command window in this session's tooling — if a build run gets killed partway through with no visible compiler error (just stops mid-stream), it was a timeout, not a real failure. Simply re-run `ninja qemu-system-arm` from the `build/` directory — ninja is incremental and picks up where it left off.

## Verifying the build

```bash
cd build
./qemu-system-arm -cpu help | grep arm7tdmi   # confirms the new CPU type is registered
./qemu-system-arm --version
```

## Running the WSL toolchain from this Windows-side agent session

The Bash tool available in this session is **Git Bash**, not WSL — WSL must be invoked explicitly via `wsl.exe`. Two gotchas discovered while setting this up:

1. **Nested quoting through `wsl.exe -lc '...'` is unreliable** — semicolons, `$` expansion, and multi-statement strings silently break (e.g. `wsl.exe -d Ubuntu -- bash -lc 'x=hello; echo $x'` produced no output at all, no error). **Fix: write the script to a file, then invoke `wsl.exe -d Ubuntu -- bash <path>`** rather than passing multi-statement strings inline.
2. **Git Bash's MSYS path conversion mangles POSIX-looking paths** passed as arguments to native Windows executables like `wsl.exe` — e.g. `/mnt/c/Users/...` gets rewritten to `C:/Program Files/Git/mnt/c/Users/...`. **Fix: prefix the command with `MSYS_NO_PATHCONV=1`**:
   ```bash
   MSYS_NO_PATHCONV=1 wsl.exe -d Ubuntu -- bash "/mnt/c/Users/Musa/Documents/gameboy/qemu/some_script.sh"
   ```

Combine both: write any nontrivial command sequence to a `.sh` file first, then run it with the `MSYS_NO_PATHCONV=1 wsl.exe -d Ubuntu -- bash "<path>"` pattern.

## Git identity / GitHub

WSL's git had no `user.name`/`user.email` configured globally. Set to match the Windows-side identity already used for commits on this machine: `git config --global user.name "heroiclion2"` / `git config --global user.email "heroiclion2@gmail.com"`.

The `qemu` repo's `origin` remote is upstream `https://gitlab.com/qemu-project/qemu.git` — **do not push there**, it's not this user's repo.

Fork repo: **`https://github.com/heroicliom2/gba-qemu`** (public), created via the Windows-side `gh` CLI (`gh repo create gba-qemu --public`, run after the user did `gh auth login` on Windows). A `github` remote pointing at `https://github.com/heroicliom2/gba-qemu.git` has been added in the WSL clone alongside `origin`.

**Important cross-environment gotcha**: don't run git commands that touch the working tree (checkout, commit, add) from Windows-side git on this repo — the repo lives on `/mnt/c` and was cloned by WSL git, and Windows git shows spurious modifications in symlinked files under `subprojects/libvduse/`, `subprojects/libvhost-user/`, and `rust/bindings/*/build.rs` (Windows can't resolve the symlinks the same way WSL/Linux does). Read-only inspection from Windows (`git status`, `git log`) is harmless; always make commits/pushes from WSL git.

**Pushing from WSL requires a credential bridge** since `gh` is not installed in WSL (only on Windows) and WSL git has no stored GitHub credential. One-time approach used so far: get a token from the already-authenticated Windows `gh` CLI (`gh auth token`) and use it directly in the push URL, e.g.:
```bash
git push "https://x-access-token:<token>@github.com/heroicliom2/gba-qemu.git" gba-machine:gba-machine
```
Never write that token into `git remote set-url` or any committed file — use it inline for a single push, then rely on the plain `github` remote (`https://github.com/heroicliom2/gba-qemu.git`, no embedded credential) for everything else. **For a durable fix**, install `gh` in WSL (`sudo apt install gh` — needs a PPA on Ubuntu 22.04, or download the `.deb` — see https://github.com/cli/cli/blob/trunk/docs/install_linux.md) and run `gh auth login` there directly; this hasn't been done yet, so every future push from WSL needs either a fresh bridged token or that one-time setup.

**Note on unshallowing**: the initial clone was shallow (`--depth 1`), which cannot be pushed cleanly to a brand-new empty GitHub repo (`error: remote unpack failed`). Fixed with `git fetch --unshallow origin` (pulls full upstream history, ~900MB+, took a few minutes) before the first push succeeded. The clone is now a full, non-shallow history.

## sudo / privileged operations

This session cannot supply a WSL sudo password (non-interactive shell, no TTY). Any `apt install` or other sudo-requiring step must be handed to the user to run manually in their own WSL terminal.
