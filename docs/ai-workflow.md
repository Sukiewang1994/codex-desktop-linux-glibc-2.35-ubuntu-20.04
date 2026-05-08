# AI 辅助工程流程

这次修复把 AI 当作调试和实现搭档使用，但每一步判断都通过本地命令、构建结果和二进制检查来验证。

## AI 参与了哪些环节

### 1. 根据报错建立假设

启动错误明确指向：

```text
better_sqlite3.node
GLIBC_2.38 not found
```

AI 帮助把这个报错转化成可验证的工程假设：包内 `better-sqlite3` 原生模块可能来自预编译二进制，并且是在比目标机器更新的 Linux 系统上构建的。

### 2. 搜索并阅读构建链路

通过搜索 `better-sqlite3`、`electron-rebuild`、`asar`、`native modules` 等关键词，定位到关键文件：

```text
install.sh
scripts/build-deb.sh
scripts/lib/package-common.sh
```

真正需要修改的是 `install.sh` 中的 `build_native_modules` 流程。

### 3. 识别隐藏假设

原脚本已经调用了 `@electron/rebuild`，但这并不等于一定会从源码编译。

AI 协助识别出关键缺口：

- npm install 阶段没有 `--build-from-source`。
- electron rebuild 阶段没有 `--build-from-source`。
- 没有设置 `npm_config_build_from_source=true`。
- 构建后没有检查 `.node` 文件的 GLIBC 版本。

### 4. 处理本机环境问题

修复过程中遇到两个额外问题：

- p7zip 16.02 无法解包新版 APFS DMG。
- 默认 GCC 9 不支持 C++20。

AI 的作用不是简单绕过它们，而是把它们转化成构建脚本的健壮性改进：

- 正确检测旧版 p7zip 并提示安装新版 `7zz`。
- 自动选择支持 C++20 的编译器，例如 `g++-11`。

### 5. 把一次性修复沉淀成可重复流程

最终不是手工复制一个 `.node` 文件，而是修改构建流程：

- 每次从 DMG 重新生成 Linux app 时都会源码编译 native modules。
- 每次构建都会检查 GLIBC 兼容性。
- 如果未来上游 native module 又引入更高 glibc 依赖，脚本会提前失败。

### 6. 用验证闭环约束 AI 输出

所有判断都用本地命令验证：

- `ldd --version`
- `strings *.node | grep GLIBC_`
- `file *.node`
- `dpkg-deb -I`
- `dpkg-deb -c`
- `cargo check`
- `cargo test`
- `ELECTRON_RUN_AS_NODE=1 electron -e "..."`

## 人的判断

AI 提供定位和实现建议，但关键决策由人来做：

- 不把官方 DMG、解包 app 和 152MB `.deb` 直接提交进 git 仓库。
- 把修复放在 `install.sh`，因为它是项目生成 Linux app 的源头。
- 不做无关重构，避免扩大改动面。
- 使用可复现验证结果作为作品集证据。

## 这次案例体现的能力

- 能读懂跨语言构建链路：Shell、Node.js、Electron、Rust、Debian packaging。
- 能从运行时报错定位到二进制兼容性根因。
- 能使用 AI 提高排查速度，但不依赖 AI 替代验证。
- 能把本机问题抽象成可维护的构建脚本改进。
- 能用文档把问题、方案、结果讲清楚，适合团队协作和面试展示。

## English Version

# AI-Assisted Engineering Workflow

This case study used AI as a pair-programming and debugging assistant, while keeping all decisions grounded in local command output and reproducible validation.

## How AI Was Used

1. **Triage the crash**

   The error message pointed at `better_sqlite3.node` and `GLIBC_2.38`. AI helped convert that into a concrete hypothesis: the package contained a prebuilt native addon compiled against a newer glibc than the target system.

2. **Inspect the build pipeline**

   We searched the repository for native-module, Electron rebuild, and packaging logic. The important path was `install.sh`, especially the `build_native_modules` function.

3. **Identify hidden assumptions**

   The installer assumed that running `@electron/rebuild` was enough. The AI-assisted review surfaced the missing `--build-from-source` and `npm_config_build_from_source=true` controls.

4. **Handle environment-specific failures**

   During implementation, two local machine issues appeared:

   - Old p7zip 16.02 could not extract the modern DMG.
   - Default GCC 9 did not support `-std=c++20`.

   AI helped turn these from one-off local workarounds into installer improvements: better 7-Zip detection and automatic C++20 compiler selection.

5. **Add defensive verification**

   Rather than only rebuilding and hoping the crash was gone, the final script scans generated `.node` files for GLIBC symbol requirements and fails before packaging incompatible binaries.

6. **Verify through the real runtime path**

   The native modules were tested through Electron's `app.asar` path, matching how the packaged app resolves unpacked native addons at runtime.

## Human Judgment Used

- Generated artifacts were not committed to the portfolio repository.
- The package was validated locally but not uploaded as source.
- The fix was kept in `install.sh`, which is the project's source of truth for regenerating `codex-app`.
- The work avoided changing unrelated packaging logic.

## Why This Is a Strong AI Tooling Example

This was not a prompt-only exercise. The AI loop was useful because every proposed explanation was tested against the machine:

- local glibc version
- native addon symbol tables
- compiler behavior
- Electron ABI rebuild behavior
- package contents
- runtime require checks

That makes the result reproducible and reviewable.

