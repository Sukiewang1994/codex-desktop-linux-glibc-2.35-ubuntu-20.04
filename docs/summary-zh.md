# 中文项目总结

## 项目背景

这个项目是一次真实的 Linux 桌面应用兼容性修复案例。原项目把官方 macOS Codex Desktop DMG 转换成 Linux 可运行版本，并打包成 Debian/RPM/pacman 包。

我在自己的 Ubuntu 22.04 机器上安装源码编译出的 Debian 包后，应用启动失败，报错指出 `better-sqlite3` 原生模块需要 `GLIBC_2.38`，但系统只有 `GLIBC_2.35`。

## 问题本质

Electron 应用里的部分 Node.js 依赖是原生模块，会编译成 `.node` 动态库。`better-sqlite3.node` 如果是在更新的 Linux 发行版上编译，就可能依赖更高版本的 glibc。这个二进制被打进 Debian 包后，在旧一点的系统上会直接加载失败。

## 修复方案

我把修复重点放在项目的源头构建流程，而不是手动替换某个已生成文件：

- 修改 `install.sh`，强制 `better-sqlite3` 和 `node-pty` 从源码编译。
- 使用 `@electron/rebuild` 针对 Electron 40 的 ABI 重新构建原生模块。
- 自动选择支持 C++20 的编译器，解决系统默认 GCC 9 无法编译 `better-sqlite3@12.8.0` 的问题。
- 编译完成后扫描 `.node` 文件中的 GLIBC 符号版本，如果产物要求高于本机 glibc，就在打包前失败。
- 修复旧版 p7zip 16.02 检测，避免无法解包新版 APFS DMG 时才暴露问题。

## 验证结果

最终重新生成了 Debian 包：

```text
codex-desktop_2026.05.08.094147_amd64.deb
```

包内 Linux 原生模块最高只要求：

```text
GLIBC_2.34
```

目标机器提供的是：

```text
GLIBC_2.35
```

因此原来的 `GLIBC_2.38 not found` 崩溃被解决。

同时完成了以下验证：

- Shell 脚本语法检查通过。
- Rust updater `cargo check` 通过。
- Rust updater 48 个测试全部通过。
- Debian 包元数据和文件布局检查通过。
- 使用 Electron 从真实 `app.asar` 路径加载 `better-sqlite3` 和 `node-pty` 成功。

## AI 工具使用方式

这次不是简单让 AI 生成代码，而是把 AI 当作调试搭档：

- 根据报错建立假设。
- 搜索并阅读项目构建链路。
- 定位 native module rebuild 流程中的隐性问题。
- 处理构建过程中出现的本机环境问题。
- 把一次性修复沉淀成可重复执行的安装脚本逻辑。
- 每一步都用本地命令验证，而不是只相信推理。

## 可以在简历中描述为

使用 AI 辅助完成 Electron Linux 打包兼容性修复：定位 `better-sqlite3` 原生模块因预编译二进制依赖 `GLIBC_2.38` 导致 Ubuntu 22.04 启动失败的问题；修改构建脚本强制 native addons 针对 Electron ABI 从源码编译，增加 C++20 编译器自动选择与 GLIBC 符号检查；重新生成 Debian 包并通过 Electron runtime、dpkg、cargo test 等验证，最终将运行时 GLIBC 需求降至 `GLIBC_2.34`。

