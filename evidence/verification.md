# 验证证据

## 本机运行环境

```text
ldd (Ubuntu GLIBC 2.35-0ubuntu3.13) 2.35
Node.js v20.20.2
npm 10.8.2
```

## 构建出的 Debian 包

```text
文件名: codex-desktop_2026.05.08.094147_amd64.deb
大小: 152M
版本: 2026.05.08.094147
架构: amd64
```

本地保存路径：

```text
release-artifacts/codex-desktop_2026.05.08.094147_amd64.deb
```

SHA256：

```text
b257994f69535a3b08404a8491aeff586b14adcf9ecdf7dc87f9be8c2ece821e
```

说明：这个 `.deb` 文件已复制到本地作品集目录，但没有提交进 git。原因是普通 GitHub 仓库不适合提交 152MB 的二进制包，而且包内包含上游应用二进制。若确认有分发授权，更合适的方式是作为 GitHub Release asset 上传。

## GLIBC 符号扫描

最终 Debian 包里的 Linux ELF 原生模块最高只要求 `GLIBC_2.34`：

```text
GLIBC_2.34 resources/app.asar.unpacked/node_modules/better-sqlite3/bin/linux-x64-143/better-sqlite3.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/better-sqlite3/build/Release/better_sqlite3.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/better-sqlite3/build/Release/obj.target/better_sqlite3.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/node-pty/bin/linux-x64-143/node-pty.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/node-pty/build/Release/obj.target/pty.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/node-pty/build/Release/pty.node
```

没有发现 `GLIBC_2.38`：

```text
No GLIBC_2.38 requirement was found in Linux .node files.
```

## Electron 运行时加载测试

通过 Electron 从真实 `app.asar` 路径加载原生模块：

```text
sqlite 1
node-pty function
```

这说明 `better-sqlite3` 可以被 Electron 加载，并且 SQLite 查询可以正常执行；`node-pty` 也能被加载并暴露 `spawn` 方法。

## 执行过的验证命令

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

## Rust 测试结果

```text
test result: ok. 48 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## English Version

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
SHA256: b257994f69535a3b08404a8491aeff586b14adcf9ecdf7dc87f9be8c2ece821e
```

The `.deb` file is copied locally under `release-artifacts/` but is not committed to git. A 152MB binary package is not appropriate for normal git storage, and the package contains upstream application binaries. If redistribution is allowed, it should be uploaded as a GitHub Release asset instead.

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

