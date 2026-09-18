# niri 26.04 SHM 屏幕共享 backport
> 版本大于26.04-1可以直接跳到 "验证 & 必要配置"

将 niri 上游 PR [niri-wm/niri#1791](https://github.com/niri-wm/niri/pull/1791)
（**Support shm sharing**）backport 到 Arch Linux 官方包 `niri 26.04` 的本地构建仓库。

> ⚠️ 这是临时方案。PR #1791 已于 2026-09-12 合入上游 main，将包含在下个正式版
> （v26.08 前后）。届时请直接升级官方包并停止使用本仓库。

## 背景

niri 的屏幕共享（Mutter ScreenCast portal 实现）此前只支持 DMABUF 方式向
PipeWire 客户端传帧。部分客户端（如腾讯会议）无法协商 DMABUF，导致：

```
no more input formats
```

共享屏幕失败/黑屏。该 PR 为 screencast 增加了 SHM（共享内存）回退路径：
客户端不支持 DMABUF 时自动协商到 SHM，帧渲染到普通内存缓冲再共享。

该改动不在任何已发布的 niri 版本中（截至 26.04），故有此 backport。

## 仓库内容

| 文件 | 说明 |
|------|------|
| `PKGBUILD` | 基于官方 `niri 26.04-1` 的 PKGBUILD 修改：加入补丁、`pkgrel` 改为 `1.1` 以区分本地重打包 |
| `niri-shm-sharing.patch` | PR #1791 的完整改动，按上游逐个 commit cherry-pick 到 v26.04 后导出 |

## 安装必要依赖
```bash
 sudo pacman -S xdg-desktop-portal xdg-desktop-portal-gnome xdg-desktop-portal-gtk gnome-keyring
```

## 构建步骤

依赖：`base-devel`、`rust`、`clang`（PKGBUILD 的 `makedepends` 会自动检查，
首次构建需联网下载源码和 cargo 依赖，编译 Rust release 需要较长时间）。

```bash
git clone https://github.com/bt-ash/niri-backport.git
cd niri-patched

# 校验补丁（可选）：确认来源
# https://github.com/niri-wm/niri/pull/1791

makepkg -f --nocheck
```

构建完成后得到 `niri-26.04-1.1-x86_64.pkg.tar.zst`（及 debug 包），并安装。
```bash
sudo pacman -U ~/build/niri-backport/niri-26.04-1.1-x86_64.pkg.tar.zst
```

## 验证 & 必要配置

```bash
# 版本应为 26.04-1.1/版本>26.04-1.（官方包是 26.04-1，.1 后缀即本地补丁版）
pacman -Qi niri

# 确保有以下配置
cat ~/.config/xdg-desktop-portal/niri-portal.conf
  [preferred]
  default=gnome;gtk;
  org.freedesktop.impl.portal.Access=gtk;
  org.freedesktop.impl.portal.Notification=gtk;
  org.freedesktop.impl.portal.Secret=gnome-keyring;
  org.freedesktop.impl.portal.ScreenCast=gnome;
  org.freedesktop.impl.portal.Screenshot=gnome;
```

在 ~/.config/niri/config.kdl配置启动项启动项
```conf
spawn-sh-at-startup "dbus-update-activation-environment --systemd WAYLAND_DISPLAY XDG_CURRENT_DESKTOP=niri & /usr/lib/xdg-desktop-portal-gnome"
```

flatpak 启动腾讯会议
```bash
# 强制xwayland启动腾讯会议
flatpak override --user --env=QT_QPA_PLATFORM=xcb com.tencent.wemeet
# xwayland临时生效强制xwayland启动腾讯会议
flatpak run --env=QT_QPA_PLATFORM=xcb com.tencent.wemeet

```
实际测试：在会议软件中共享屏幕，观察是否还出现
`no more input formats`；也可看 PipeWire 协商日志：

```bash
journalctl --user -u xdg-desktop-portal -e
```

## 维护注意事项

1. **防止被升级覆盖**：在 `/etc/pacman.conf` 中加入：
   ```ini
   IgnorePkg = niri
   ```
   上游发布包含 #1791 的正式版后移除此行并 `pacman -Syu`。

2. **上游出新版时**：更新 `PKGBUILD` 的 `pkgver`（若补丁能干净应用），
   并同步修改 `.gitignore` 中的源码包文件名；若上游已含 #1791，本仓库
   即可归档。

3. **补丁失效时**：可参考 `niri-shm-sharing.patch` 的生成方式——从
   v26.04 tag 建分支，逐个 cherry-pick PR #1791 的 commit 并适配，然后
   `git format-patch v26.04` 重新导出。
