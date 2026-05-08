# Codex Desktop Linux GLIBC 兼容性修复

这是一个真实的桌面应用兼容性修复案例。我在 Ubuntu 22.04 上安装自己从源码构建的 Codex Desktop Linux Debian 包后，应用启动失败。最终通过 AI 辅助排查和本地验证，定位到 Electron 原生 Node 模块的 glibc 版本不兼容问题，并修改构建流程生成了可在当前机器运行的新 Debian 包。

这个仓库不是完整源码镜像，也不包含官方 DMG、解包后的应用目录或 `.deb` 大文件；它是一个面向作品集和面试展示的工程复盘仓库，重点展示问题分析、修复思路、AI 协作方式和验证证据。

## 问题背景

原项目的目标是把官方 macOS Codex Desktop DMG 转换成 Linux 可运行版本，并打包成 `.deb`、`.rpm` 和 pacman 包。

在我的机器上安装生成的 Debian 包后，Codex Desktop 启动时报错：

```text
Codex failed to start.

/lib/x86_64-linux-gnu/libm.so.6: version `GLIBC_2.38' not found
(required by /opt/codex-desktop/resources/app.asar.unpacked/
node_modules/better-sqlite3/build/Release/better_sqlite3.node)
```

当前系统环境是：

```text
Ubuntu 22.04
GLIBC 2.35
Node.js v20.20.2
Electron 40.0.0
```

## 根因分析

Electron 应用里的部分 Node.js 依赖是原生模块，会编译成 `.node` 动态库。这里出问题的是 `better-sqlite3.node`。

原构建流程虽然已经尝试使用 `@electron/rebuild` 重建原生模块，但没有强制从源码编译。结果 npm 可能保留或下载了在更新 Linux 系统上编译好的预编译二进制，这个二进制依赖 `GLIBC_2.38`。当它被打进 Debian 包后，在 Ubuntu 22.04 的 `GLIBC_2.35` 上就会直接加载失败。

修复过程中还暴露了两个本机环境问题：

- 默认 `/usr/bin/g++` 是 GCC 9.4.0，不支持 `better-sqlite3@12.8.0` 需要的 `-std=c++20`。
- 系统自带 p7zip 16.02 太旧，无法解包当前 APFS 格式的 DMG。

## 修复方案

我把修复放在构建源头，也就是 `install.sh`，而不是手动替换某个生成后的 `.node` 文件。

关键改动包括：

- 强制 `better-sqlite3` 和 `node-pty` 从源码安装和编译。
- 使用 `@electron/rebuild` 针对 Electron 40 ABI 重新构建 native addons。
- 自动选择支持 C++20 的编译器，例如本机的 `/usr/bin/g++-11`。
- 构建后扫描 `.node` 文件中的 GLIBC 符号版本，如果产物要求高于本机 glibc，就在打包前失败。
- 修复旧 p7zip 16.02 的检测逻辑，提前提示需要新版 `7zz`。

核心构建逻辑如下：

```bash
npm install "better-sqlite3@$bs3_ver" "node-pty@$npty_ver" --ignore-scripts --build-from-source

npm_config_build_from_source=true \
    npx --yes @electron/rebuild -v "$ELECTRON_VERSION" --force --build-from-source
```

## 最终结果

重新生成的 Debian 包：

```text
codex-desktop_2026.05.08.094147_amd64.deb
```

修复前：

```text
better_sqlite3.node requires GLIBC_2.38
host provides GLIBC_2.35
```

修复后：

```text
Linux native modules require at most GLIBC_2.34
host provides GLIBC_2.35
```

因此原来的 `GLIBC_2.38 not found` 启动崩溃被解决。

## 验证方式

我没有只依赖“能构建成功”，而是从多个层面验证：

- `bash -n` 检查安装脚本和打包脚本语法。
- `cargo check -p codex-update-manager` 验证 Rust updater 编译。
- `cargo test -p codex-update-manager` 跑完 48 个测试。
- `dpkg-deb -I` 和 `dpkg-deb -c` 检查 Debian 包元数据和文件布局。
- 扫描包内 Linux `.node` 文件，确认没有 `GLIBC_2.38`。
- 使用 Electron 从真实 `app.asar` 路径加载 `better-sqlite3` 和 `node-pty`。

验证结果摘要：

```text
GLIBC_2.34 resources/app.asar.unpacked/node_modules/better-sqlite3/build/Release/better_sqlite3.node
GLIBC_2.34 resources/app.asar.unpacked/node_modules/node-pty/build/Release/pty.node

sqlite 1
node-pty function

test result: ok. 48 passed; 0 failed
```

## AI 工具是如何参与的

这次不是简单让 AI 写代码，而是把 AI 当作调试搭档使用：

1. 根据启动报错建立假设：问题可能来自 native addon 的 glibc 版本。
2. 搜索项目构建链路，定位到 `install.sh` 的 native module rebuild 流程。
3. 识别隐藏问题：`@electron/rebuild` 不等于一定从源码编译。
4. 在构建失败时继续排查本机环境，例如旧 p7zip 和 GCC 9。
5. 把一次性修复沉淀成可重复执行的安装脚本逻辑。
6. 用本地命令验证每个判断，而不是只相信模型推理。

这体现了我使用 AI 工具时的工作方式：让 AI 提高定位和实现效率，但最终以可复现的命令输出、测试结果和包内二进制检查作为依据。

## 可以在简历中这样描述

使用 AI 辅助完成 Electron Linux 打包兼容性修复：定位 `better-sqlite3` 原生模块因预编译二进制依赖 `GLIBC_2.38` 导致 Ubuntu 22.04 启动失败的问题；修改构建脚本强制 native addons 针对 Electron ABI 从源码编译，增加 C++20 编译器自动选择与 GLIBC 符号检查；重新生成 Debian 包并通过 Electron runtime、dpkg、cargo test 等验证，最终将运行时 GLIBC 需求降至 `GLIBC_2.34`。

## 仓库内容

- [docs/summary-zh.md](docs/summary-zh.md)：中文项目总结，可用于简历和面试表达。
- [docs/case-study.md](docs/case-study.md)：英文完整技术复盘。
- [docs/ai-workflow.md](docs/ai-workflow.md)：AI 辅助开发流程说明。
- [docs/release-artifact.md](docs/release-artifact.md)：Debian 包本地产物和 GitHub Release 发布说明。
- [evidence/verification.md](evidence/verification.md)：验证命令和结果。
- [patches/install-sh-key-changes.diff](patches/install-sh-key-changes.diff)：关键补丁摘要。

## English Version

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
