# Case Study: Fixing a GLIBC Crash in a Linux Electron Package

## Context

The project adapts the official macOS Codex Desktop DMG into a runnable Linux application and packages it as `.deb`, `.rpm`, and pacman artifacts. The Linux build extracts `app.asar`, rebuilds native Node modules, downloads a Linux Electron runtime, generates a launcher, and packages the result.

The target machine was Ubuntu 22.04 with:

```text
ldd (Ubuntu GLIBC 2.35-0ubuntu3.13) 2.35
Node.js v20.20.2
Electron 40.0.0
```

## Failure

After installing the generated Debian package, Codex Desktop failed immediately on launch:

```text
Codex failed to start.

/lib/x86_64-linux-gnu/libm.so.6: version `GLIBC_2.38' not found
(required by /opt/codex-desktop/resources/app.asar.unpacked/
node_modules/better-sqlite3/build/Release/better_sqlite3.node)
```

This meant the packaged `better-sqlite3.node` binary had been compiled on a newer Linux distribution than the target system.

## Root Cause

The installer already attempted to rebuild native modules, but the npm install/rebuild path did not force source compilation. For packages such as `better-sqlite3`, npm can install or retain prebuilt native binaries. In this case, a binary requiring `GLIBC_2.38` was packaged into the Linux build.

Two additional environment problems appeared during the fix:

- The default `/usr/bin/g++` was GCC 9.4.0 and did not accept `-std=c++20`, which `better-sqlite3@12.8.0` requires.
- The system `7z` was old p7zip 16.02 and could not extract the current APFS-based DMG.

## Fix

The installer was updated in three areas.

First, native modules are now forced to build from source:

```bash
npm install "better-sqlite3@$bs3_ver" "node-pty@$npty_ver" --ignore-scripts --build-from-source

npm_config_build_from_source=true \
    npx --yes @electron/rebuild -v "$ELECTRON_VERSION" --force --build-from-source
```

Second, the installer now chooses a C++20-capable compiler instead of assuming `g++` is modern enough. On the target machine, it selected:

```text
CXX=/usr/bin/g++-11
CC=/usr/bin/gcc-11
```

Third, the build now checks generated `.node` files for GLIBC symbol requirements and fails early if any native module requires a newer glibc than the host provides.

## Validation

The rebuilt package was checked at several levels:

- Shell syntax validation for installer and package scripts.
- Rust updater validation with `cargo check` and `cargo test`.
- Debian metadata and file layout inspection with `dpkg-deb`.
- GLIBC symbol scan across Linux `.node` files.
- Runtime module loading through Electron's real `app.asar` path.

The final package contained Linux native modules requiring at most `GLIBC_2.34`, which is compatible with Ubuntu 22.04's `GLIBC_2.35`.

## Outcome

The new Debian package was generated at:

```text
dist/codex-desktop_2026.05.08.094147_amd64.deb
```

The package is not stored in this portfolio repository because it is a large generated artifact and contains upstream application binaries. The reusable value is the source-level installer fix and the debugging workflow documented here.

