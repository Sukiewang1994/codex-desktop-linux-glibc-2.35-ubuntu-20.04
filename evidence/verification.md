# Verification Evidence

## Host Runtime

```text
ldd (Ubuntu GLIBC 2.35-0ubuntu3.13) 2.35
Node.js v20.20.2
npm 10.8.2
```

## Package Built

```text
dist/codex-desktop_2026.05.08.094147_amd64.deb
Size: 152M
Version: 2026.05.08.094147
Architecture: amd64
```

## GLIBC Symbol Scan

The final Linux ELF native modules in the Debian package required at most `GLIBC_2.34`:

```text
GLIBC_2.34 resources/app.asar.unpacked/node_modules/better-sqlite3/bin/linux-x64-143/better-sqlite3.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/better-sqlite3/build/Release/better_sqlite3.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/better-sqlite3/build/Release/obj.target/better_sqlite3.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/node-pty/bin/linux-x64-143/node-pty.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/node-pty/build/Release/obj.target/pty.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/node-pty/build/Release/pty.node
```

No `GLIBC_2.38` requirement was found in the Linux `.node` files.

## Runtime Native Addon Load Test

The modules were loaded through Electron from the real `app.asar` path:

```text
sqlite 1
node-pty function
```

## Test Commands

```bash
bash -n install.sh
bash -n scripts/build-deb.sh
bash -n scripts/build-rpm.sh
bash -n scripts/build-pacman.sh
cargo check -p codex-update-manager
cargo test -p codex-update-manager
dpkg-deb -I dist/codex-desktop_2026.05.08.094147_amd64.deb
dpkg-deb -c dist/codex-desktop_2026.05.08.094147_amd64.deb
```

## Rust Test Result

```text
test result: ok. 48 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

