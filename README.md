# Codex Desktop Linux GLIBC Compatibility Fix

This repository is a portfolio case study about debugging and fixing a native-module compatibility failure in a community Linux build of Codex Desktop.

The original application failed on Ubuntu 22.04 because the packaged `better-sqlite3` native addon required `GLIBC_2.38`, while the machine provided `GLIBC_2.35`. I used AI-assisted debugging to trace the failure to the native module rebuild flow, changed the installer to compile Linux native addons from source on the target system, added compatibility checks, rebuilt the Debian package, and verified the resulting app under Electron.

## Result

- Built a new Debian package: `codex-desktop_2026.05.08.094147_amd64.deb`.
- Rebuilt `better-sqlite3@12.8.0` and `node-pty@1.1.0` for Electron 40 on Ubuntu 22.04.
- Reduced the Linux native addon requirement from `GLIBC_2.38` to `GLIBC_2.34`, compatible with the target machine's `GLIBC_2.35`.
- Verified native addon loading through Electron's real `app.asar` resolution path.
- Kept generated binaries and upstream app assets out of this portfolio repository.

## Technical Highlights

- Electron native addon rebuilds with `@electron/rebuild`.
- Forced source builds to avoid npm prebuilt binaries compiled against newer glibc.
- Automatic C++20 compiler selection for `better-sqlite3@12.8.0`.
- Defensive GLIBC symbol scanning after native module compilation.
- Debian package validation with `dpkg-deb`.
- Rust updater validation with `cargo check` and `cargo test`.

## Repository Contents

- [docs/case-study.md](docs/case-study.md): full technical write-up.
- [docs/ai-workflow.md](docs/ai-workflow.md): how AI tools were used during the debugging and implementation loop.
- [evidence/verification.md](evidence/verification.md): command outputs and validation evidence.
- [patches/install-sh-key-changes.diff](patches/install-sh-key-changes.diff): summarized installer patch.

## What Is Not Included

This repository intentionally does not include the official DMG, extracted application bundle, generated Debian package, or other large binary artifacts. The focus is the engineering process, debugging method, and source-level fix.

## Skills Demonstrated

- Linux packaging and runtime debugging
- Native Node.js module build systems
- Electron ABI compatibility
- glibc symbol/version analysis
- Shell scripting for reproducible installers
- Rust CLI validation
- AI-assisted software engineering workflow

