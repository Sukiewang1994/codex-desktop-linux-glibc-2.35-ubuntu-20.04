# 技术复盘：修复 Linux Electron 包的 GLIBC 启动崩溃

## 背景

这个项目把官方 macOS Codex Desktop DMG 转换成 Linux 可运行应用，并打包成 `.deb`、`.rpm` 和 pacman 包。构建流程大致包括：

1. 解包 `Codex.dmg`。
2. 提取并修改 `app.asar`。
3. 为 Linux 重新构建原生 Node.js 模块。
4. 下载 Linux Electron runtime。
5. 生成 Linux 启动脚本。
6. 打包成 Debian/RPM/pacman 安装包。

目标机器环境：

```text
Ubuntu 22.04
ldd (Ubuntu GLIBC 2.35-0ubuntu3.13) 2.35
Node.js v20.20.2
Electron 40.0.0
```

## 故障现象

安装生成的 Debian 包后，应用启动失败：

```text
Codex failed to start.

/lib/x86_64-linux-gnu/libm.so.6: version `GLIBC_2.38' not found
(required by /opt/codex-desktop/resources/app.asar.unpacked/
node_modules/better-sqlite3/build/Release/better_sqlite3.node)
```

这个错误说明包内的 `better_sqlite3.node` 是一个 Linux 原生动态库，并且它依赖的 glibc 版本高于当前系统提供的版本。

## 根因

`better-sqlite3` 是 Node.js 原生模块。Electron 应用使用它时，需要有和 Electron ABI 匹配的 `.node` 二进制。

原来的 `install.sh` 已经尝试运行 `@electron/rebuild`，但 npm 安装和 rebuild 流程没有强制源码编译。对于 `better-sqlite3` 这类模块，npm 可能使用预编译二进制。这个预编译二进制是在更新的 Linux 系统上构建的，因此要求 `GLIBC_2.38`。

当这个 `.node` 被打进 Debian 包并安装到 Ubuntu 22.04 后，系统只有 `GLIBC_2.35`，动态链接器无法加载它，应用启动阶段直接崩溃。

修复过程中还发现两个构建环境问题：

- 默认 `/usr/bin/g++` 是 GCC 9.4.0，不支持 `better-sqlite3@12.8.0` 需要的 `-std=c++20`。
- 系统 `7z` 是 p7zip 16.02，无法解包当前 APFS 格式的 DMG。

## 修复方案

修复重点放在构建源头 `install.sh`，而不是手工替换已生成的 `better_sqlite3.node`。

### 1. 强制 native modules 从源码编译

```bash
npm install "better-sqlite3@$bs3_ver" "node-pty@$npty_ver" --ignore-scripts --build-from-source

npm_config_build_from_source=true \
    npx --yes @electron/rebuild -v "$ELECTRON_VERSION" --force --build-from-source
```

这样可以避免 npm 下载或保留不适配当前系统 glibc 的预编译二进制。

### 2. 自动选择 C++20 编译器

`better-sqlite3@12.8.0` 需要 C++20。脚本现在会检测 `g++-14`、`g++-13`、`g++-12`、`g++-11`、`clang++` 等候选编译器，选择真正能编译 `-std=c++20` 的版本。

在本机实际选择的是：

```text
CXX=/usr/bin/g++-11
CC=/usr/bin/gcc-11
```

### 3. 构建后扫描 GLIBC 符号

新增检查逻辑会读取 `.node` 文件中的 `GLIBC_x.y` 符号版本，并和本机 glibc 版本比较。如果产物需要的 glibc 比本机更新，脚本会在打包前失败。

这能防止问题再次进入 `.deb` 包。

### 4. 改进 7-Zip 检测

旧版 p7zip 16.02 不能解包当前 DMG。脚本原来的检测因为输出首行为空而没有拦住旧版本。修复后会正确检测并提示安装新版 `7zz`。

## 验证

修复后重新生成 Debian 包：

```text
codex-desktop_2026.05.08.094147_amd64.deb
```

包内 Linux ELF `.node` 文件最高只要求：

```text
GLIBC_2.34
```

目标机器提供：

```text
GLIBC_2.35
```

因此符合运行条件。

验证内容包括：

- `bash -n install.sh`
- `bash -n scripts/build-deb.sh`
- `bash -n scripts/build-rpm.sh`
- `bash -n scripts/build-pacman.sh`
- `cargo check -p codex-update-manager`
- `cargo test -p codex-update-manager`
- `dpkg-deb -I` 查看包元数据
- `dpkg-deb -c` 查看包内容
- `strings *.node | grep GLIBC_` 扫描 GLIBC 符号
- 使用 Electron 从真实 `app.asar` 路径加载 `better-sqlite3` 和 `node-pty`

## 结果

原来的启动错误：

```text
GLIBC_2.38 not found
```

被修复为：

```text
Linux native modules require at most GLIBC_2.34
host provides GLIBC_2.35
```

这个修复让生成的 Debian 包可以在当前 Ubuntu 22.04 机器上运行，也让后续自动更新构建流程更可靠。

## English Version

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

The package is not stored in git because it is a large generated artifact and contains upstream application binaries. The reusable value is the source-level installer fix and the debugging workflow documented here.

