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

