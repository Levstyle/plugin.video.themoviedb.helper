# ISO STRM 播放时 artwork 错误回源事故与修复

## 文档状态

- 记录日期：2026-09-17
- 修复分支：`codex/strm-sidecar-artwork`
- 修复提交：`c3850bf4`
- 发布版本：`6.17.1.1`
- 发布标签：`douban-6.17.1.1`
- 发布地址：<https://github.com/Levstyle/plugin.video.themoviedb.helper/releases/tag/douban-6.17.1.1>
- 当前状态：代码、自动打包和离线行为测试已完成；CoreELEC 实机回归待设备可访问时执行

## 摘要

CoreELEC/Kodi 播放 NAS 上的 `.iso.strm` 时，播放器 artwork 可能从 `.strm` 内容中的 LitePan 播放 URL 派生，而不是从 NAS 上 `.strm` 文件自身的 SMB 路径派生。

派生出的 poster URL 虽然把文件名改成了 `*.iso-poster.jpg`，但 URL 中用于定位资源的 `file_key` 与 ISO 播放 URL 完全相同。LitePan 按 `file_key` 查找真实对象，文件名主要用于展示和 MIME 推导，因此 poster 请求最终仍返回完整 ISO。TMDbHelper 随后把响应复制到图片处理临时文件，导致 CoreELEC 存储空间快速耗尽。

本次修复仅修改 TMDbHelper：播放源为 STRM 时，从 Kodi 保留的原始 `ListItem.Path` 获取 SMB `.strm` 路径，并在同目录查找旁路 poster、fanart 和 clearlogo。STRM 分支不再使用可能由解析后播放 URL 派生的 `Player.Art(...)`。

## 相关项目

| 项目 | 仓库 | 本次分析中的角色 |
| --- | --- | --- |
| TMDbHelper 修复仓库 | <https://github.com/Levstyle/plugin.video.themoviedb.helper> | 本次修复和发布位置 |
| TMDbHelper fork 来源 | <https://github.com/wabisabi926/plugin.video.themoviedb.helper> | 当前 `douban` 分支来源 |
| TMDbHelper 原始上游 | <https://github.com/jurialmunkey/plugin.video.themoviedb.helper> | 原始插件项目 |
| LitePan | <https://github.com/Ponphil/LitePan> | 生成播放 URL，并按 `file_key` 提供文件内容 |
| FastVFS | <https://github.com/Levstyle/vfs.stream.fast> | 实际打开 LitePan URL；日志暴露了异常 poster 请求和返回大小 |
| CoreELEC | <https://github.com/Levstyle/CoreELEC> | Kodi 运行环境和 Kodi 版本来源 |
| Kodi | <https://github.com/xbmc/xbmc> | 解析 STRM、保存播放条目和提供 `Player.Art` |
| faststrm | <https://github.com/wabisabi926/faststrm> | 对照检查的另一个 STRM 项目 |

CoreELEC 当前配置的 Kodi 源码版本为：

```text
c0d95ea08ccf831c46590028217d81ece8ea7ec4
```

## 正常文件布局

NAS 上的媒体文件布局如下：

```text
smb://NAS/Movies/Identity 2003.iso.strm
smb://NAS/Movies/Identity 2003.iso-poster.jpg
smb://NAS/Movies/Identity 2003.iso-fanart.jpg
```

`.strm` 文件只有一行播放地址，示意如下：

```text
http://192.168.1.18:5211/api/strm/play/1/<file_key>/t/<token>/n/Identity%202003.iso
```

预期行为是从 `.strm` 自身所在的 SMB 目录查找 artwork：

```text
Identity 2003.iso.strm
        │ 去掉 .strm
        ▼
Identity 2003.iso
        │ 添加 artwork 后缀
        ▼
Identity 2003.iso-poster.jpg
```

## 故障现象

CoreELEC 上曾在以下目录产生三个超大临时图片文件：

```text
/storage/.kodi/userdata/addon_data/plugin.video.themoviedb.helper/blur_v3/
```

三个 `temp_*.jpg` 总计约 114.7 GB。其中一个文件大小为：

```text
66058387456 bytes
```

该大小与正在播放的 61.52 GB ISO 完全相同。文件不是合法 JPEG，文件头也不符合图片格式。

FastVFS 日志同时出现了类似记录：

```text
打开 .../n/Identity%202003.iso
大小 66058387456

打开 .../n/Identity%202003.iso-poster.jpg
大小 66058387456
```

两个 URL 的可见文件名不同，但嵌入 URL 的 `<file_key>` 完全相同。

## 根因链路

### 1. Kodi 在播放前解析 STRM

Kodi 在 `CPlayListPlayer::Play()` 中先调用 `CPlayList::Expand()`，然后才调用 `PlayFile()`。`Expand()` 会读取 `.strm` 内容，并有意保存两种路径：

| Kodi 字段 | STRM 播放时的值 | 用途 |
| --- | --- | --- |
| `CFileItem::Path` | 原始 `smb://.../Identity 2003.iso.strm` | 媒体条目的稳定标识和来源路径 |
| `CFileItem::DynPath` | `.strm` 内的 LitePan HTTP URL | 实际打开和播放的动态地址 |

对应 Kodi 源码：

- [`CPlayList::Expand()`](https://github.com/xbmc/xbmc/blob/c0d95ea08ccf831c46590028217d81ece8ea7ec4/xbmc/playlists/PlayList.cpp#L462-L496)
- [`Player.getPlayingFile()`](https://github.com/xbmc/xbmc/blob/c0d95ea08ccf831c46590028217d81ece8ea7ec4/xbmc/interfaces/legacy/Player.cpp#L374-L390)

因此播放回调发生时，STRM 已经解析完成，但原始 SMB `.strm` 路径仍然保存在 `getPlayingItem().getPath()` 中。

### 2. ISO 会经过额外的光盘解析

ISO/蓝光播放还可能经过 playlist 选择和 `bluray://` 路径转换。这个过程会进一步更新动态播放路径以及 `VideoInfoTag` 中的文件路径。`Player.FilenameAndPath` 优先读取 `VideoInfoTag`，所以对 ISO STRM 来说不适合作为原始 `.strm` 来源。

这也解释了为什么普通 MKV STRM 通常表现更好：MKV 不需要经过 ISO/蓝光 playlist 解析，路径状态更简单。问题并不表示 ISO 文件损坏。

### 3. artwork URL 保留了相同的 file_key

异常 poster URL 形如：

```text
.../<same_file_key>/.../Identity%202003.iso-poster.jpg
```

虽然末尾文件名变成了 poster，但 LitePan 的资源身份由 `<file_key>` 决定。服务端仍找到原 ISO，并返回 ISO 数据。

这个行为对通用文件播放服务是合理的：URL 中的显示名称不应改变 `file_key` 所标识的资源。为了 TMDbHelper 的单一 artwork 场景在 LitePan 中增加文件名特判，会改变通用播放 API 的语义，并可能影响其他调用方。

### 4. TMDbHelper 未验证复制对象是否为图片

旧逻辑读取：

```text
Player.Art(poster/fanart/clearlogo)
```

随后图片处理代码通过 `xbmcvfs.copy()` 把响应复制到临时文件。该链路没有在复制完整响应前验证 MIME、图片文件头或合理大小，所以 ISO 数据可以落入 blur 临时目录。

## 责任边界

本次问题不是以下组件的单独故障：

- 不是 ISO 内容损坏；同一 ISO 可以正常播放。
- 不是 FastVFS 主动生成 poster URL；FastVFS 只是打开 Kodi/插件交给它的 URL。
- LitePan 按 `file_key` 返回文件符合其通用播放接口设计。

最合适的功能修复位置是 TMDbHelper：

- TMDbHelper 的播放 artwork/blur 功能单一，能够明确区分图片需求与媒体播放需求。
- Kodi 已经保留原始 STRM 路径，TMDbHelper 无需反向推断 LitePan URL。
- 不需要修改 LitePan、FastVFS 或 ISO。
- 不会把 LitePan 的通用接口绑定到某一种 Kodi artwork 命名规则。

## 修复设计

实现位置：

```text
resources/tmdbhelper/lib/monitor/player.py
```

核心步骤：

1. 在 `onAVStarted()` / `onAVChange()` 后的 artwork 更新阶段调用 `getPlayingItem().getPath()`。
2. 仅当该路径以 `.strm` 结尾时启用 STRM 分支。
3. 去掉末尾 `.strm`，依次查找以下旁路文件：

```text
<base>-poster.jpg
<base>-poster.png
<base>-fanart.jpg
<base>-fanart.png
<base>-clearlogo.jpg
<base>-clearlogo.png
```

4. 使用 `xbmcvfs.exists()` 在 SMB 上确认候选文件存在。
5. STRM 分支中不读取 `Player.Art(...)`。
6. 普通 MKV、直接 SMB 视频和插件播放保持原有 artwork 选择逻辑。

简化后的 poster 选择逻辑：

```text
是否识别为 STRM？
├── 否：保持原来的 Player.Art → TMDb artwork 顺序
└── 是：
    ├── SMB 同目录 -poster.jpg/png 存在 → 使用 SMB poster
    ├── SMB poster 不存在，TMDb 在线 poster 存在 → 使用 TMDb poster
    └── 两者都不存在 → 不生成 poster blur
```

### SMB poster 缺失时是否还会请求 LitePan

不会。

`get_strm_sidecar_artwork()` 返回空值后，代码仍处于 STRM 分支，只会继续检查 TMDbHelper 自己获取的在线 artwork。它不会跳转到非 STRM 分支，也不会读取以下字段：

```text
Player.Art(poster)
Player.Art(fanart)
Player.Art(clearlogo)
Player.Icon
```

因此：

```text
SMB poster 404/不存在
→ xbmcvfs.exists() 返回 false
→ 使用 TMDb 在线图片，或保持为空
→ 不访问 LitePan poster URL
→ 不会把 ISO 下载到图片缓存
```

## 为什么不使用 Player.FilenameAndPath

可用接口的语义不同：

| 接口 | 返回内容 | 本修复是否使用 |
| --- | --- | --- |
| `getPlayingItem().getPath()` | Kodi 保存的原始 STRM 路径 | 是 |
| `getPlayingFile()` | 解析后的动态播放地址 | 否 |
| `Player.FilenameAndPath` | 通常优先取 `VideoInfoTag`，ISO 时可能已经是 `bluray://` 或实际播放地址 | 否 |
| JSON-RPC `Player.GetItem.file` | 可能受 `VideoInfoTag` 路径影响 | 否 |

直接读取 `ListItem.Path` 可以避免对 LitePan URL 结构、`file_key` 或文件名参数建立依赖。

## 已知边缘风险

当前实现依赖 Kodi 在播放回调中成功提供原始 `ListItem.Path`。根据本次使用的 Kodi 22 源码，原生 `.strm` 播放会保留该路径。

但仍需在实机验证以下情况：

- `getPlayingItem()` 在特定播放入口抛出异常。
- 某个插件绕过 Kodi 的 STRM playlist 展开，直接调用解析后的 URL。
- 当前条目的 `Path` 被第三方插件替换，不再以 `.strm` 结尾。

这些情况下当前代码会按非 STRM 播放处理，并恢复原来的 `Player.Art` 行为。后续可增加两层通用防御：

1. 使用 `STRM / NON_STRM / UNKNOWN` 三态识别；来源未知时避免处理与实际视频 URL 同源的 artwork。
2. 在 `ImageFunctions` 下载层增加图片文件头、MIME 和最大文件大小校验，作为所有协议共享的最后防线。

不建议通过硬编码 LitePan host、端口或 `file_key` 路径格式解决该边缘风险。

## 验证记录

### 已完成

- Python 全量语法检查通过。
- Kodi 模块桩行为测试通过：
  - ISO STRM 能派生 SMB `-poster.jpg`。
  - SMB poster 缺失时不读取 `Player.Art`。
  - 缺失的 fanart 能回退到 TMDb 在线 artwork。
  - 非 STRM 播放保持原逻辑。
- 本地模拟 Kodi ZIP 打包成功，压缩包完整性检查通过。
- GitHub Actions 远程打包成功。
- 从 GitHub Release 重新下载 ZIP 后校验成功，包内包含本次修复。

远程构建：

<https://github.com/Levstyle/plugin.video.themoviedb.helper/actions/runs/35176290988>

安装包：

<https://github.com/Levstyle/plugin.video.themoviedb.helper/releases/download/douban-6.17.1.1/plugin.video.themoviedb.helper-6.17.1.1.zip>

安装包 SHA-256：

```text
214f626d3f3f557be5787f6493b329d882d24fee716ad9cdcd85cec8b24b5c02
```

### 待完成的 CoreELEC 实机回归

1. 安装 `plugin.video.themoviedb.helper-6.17.1.1.zip`。
2. 确认 NAS 同目录存在 `*.iso-poster.jpg`。
3. 播放对应 `.iso.strm`。
4. 确认界面 blur 正常显示。
5. 检查 Kodi/FastVFS 日志，确认不再打开 LitePan `*.iso-poster.jpg` URL。
6. 检查 `blur_v3`，确认没有接近 ISO 大小的 `temp_*.jpg`。
7. 删除或临时改名 SMB poster 后再次播放，确认只使用 TMDb 图片或不生成 poster blur，且仍不请求 LitePan poster URL。

建议重点监控：

```text
/storage/.kodi/userdata/addon_data/plugin.video.themoviedb.helper/blur_v3/
```

## 变更与发布记录

| 提交 | 内容 |
| --- | --- |
| `c3850bf4` | 从原始 STRM 路径解析旁路 artwork，并禁止 STRM 回退到 `Player.Art` |
| `dfb8be58` | 增加 GitHub Actions Kodi ZIP 打包和 Release 发布流程 |

当前功能分支：

<https://github.com/Levstyle/plugin.video.themoviedb.helper/tree/codex/strm-sidecar-artwork>

## 回滚

若实机出现兼容性问题，可以临时恢复到上游 `6.17.1`，或回滚提交 `c3850bf4`。回滚只会恢复旧的 artwork 选择逻辑，不涉及 LitePan、FastVFS、STRM 文件或媒体库数据变更。

