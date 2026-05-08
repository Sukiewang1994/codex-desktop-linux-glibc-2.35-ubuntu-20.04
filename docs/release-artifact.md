# Debian 包下载和发布说明

## 本地产物

这次修复后生成的 Debian 包已经复制到本地作品集目录：

```text
/home/user/codex-desktop-linux-glibc-fix-portfolio/release-artifacts/codex-desktop_2026.05.08.094147_amd64.deb
```

SHA256：

```text
b257994f69535a3b08404a8491aeff586b14adcf9ecdf7dc87f9be8c2ece821e
```

## 为什么不直接提交到 git 仓库

这个 `.deb` 文件大小约 152MB。普通 GitHub git 仓库不适合保存这种大二进制文件，超过 100MiB 的单文件通常无法 push。

另外，这个包是由上游 Codex Desktop DMG 转换生成的，里面包含上游应用二进制。除非确认再分发授权，否则不建议把它直接作为公开仓库文件发布。

## 推荐发布方式

如果确认可以公开分发，推荐把 `.deb` 作为 GitHub Release 附件上传，而不是提交到 git 历史。

可以在 GitHub 网页操作：

1. 打开仓库页面。
2. 进入 `Releases`。
3. 点击 `Draft a new release`。
4. Tag 填写 `v2026.05.08.094147`。
5. Release title 填写 `Codex Desktop Linux GLIBC 2.35 Fix`。
6. 上传本地文件：

   ```text
   release-artifacts/codex-desktop_2026.05.08.094147_amd64.deb
   ```

7. 在 release notes 中写明 SHA256：

   ```text
   b257994f69535a3b08404a8491aeff586b14adcf9ecdf7dc87f9be8c2ece821e
   ```

## 使用者安装方式

下载 `.deb` 后：

```bash
sudo dpkg -i codex-desktop_2026.05.08.094147_amd64.deb
sudo apt -f install
```

## English Version

# Debian Package Download and Release Notes

The fixed Debian package has been copied locally to:

```text
/home/user/codex-desktop-linux-glibc-fix-portfolio/release-artifacts/codex-desktop_2026.05.08.094147_amd64.deb
```

SHA256:

```text
b257994f69535a3b08404a8491aeff586b14adcf9ecdf7dc87f9be8c2ece821e
```

The file is not committed to git because it is about 152MB and contains upstream application binaries. If redistribution is allowed, upload it as a GitHub Release asset instead of storing it in the repository history.

