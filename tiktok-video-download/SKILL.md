---
name: tiktok-video-download
description: 解析 TikTok 分享链接，下载视频和图集（photo-mode）。当用户提供 TikTok 链接（vm.tiktok.com、vt.tiktok.com、www.tiktok.com/@user/video/...）时使用。首选 yt-dlp 的 TikTok extractor（通常免登录、无水印），被风控时回退到本机浏览器已登录会话的 CDP 提取法。⚠️ 下载逻辑适配自 douyin-video-download 的 CDP 法，TikTok 侧尚未实测；需能访问 tiktok.com（部分地区需代理）。
---

# TikTok 视频下载

解析 TikTok 分享链接 → 下载视频 / 图集（photo-mode）。

> **与抖音的关键差异（务必先读）**：抖音 2025 年中起对 yt-dlp / 纯 HTTP 全面上反爬，CDP 登录态提取是唯一可行路径。**TikTok 恰好相反**——yt-dlp 内置 `tiktok` extractor，社区长期维护，**通常免登录即可下载且拿到无水印原流**。所以这里的首选路径是 yt-dlp，CDP 法只是被风控时的兜底。不要把抖音的"CDP 唯一路"结论直接套过来。

## 状态声明（诚实标注，别当已验证事实）

本 skill 的下载层是从 `douyin-video-download` 的 CDP 法**适配**而来，**尚未在真实 TikTok 环境实测**。首次使用前需实测确认以下三点，任一条与本文不符就更新本文：

1. **yt-dlp 能否直连下载**：本机是否已装 yt-dlp、网络能否访问 tiktok.com（大陆直连不通，需代理）。
2. **水印**：yt-dlp 的 tiktok extractor 一般给无水印流；CDP 兜底读到的网页播放流**可能带水印**，需实测。
3. **视频 vs photo-mode**：`video.currentSrc` 给的是单条可下载 mp4，还是 MSE/DASH 分离流（决定要不要 Network 监听 + ffmpeg 合并）。

## 前置条件

- 能访问 `www.tiktok.com`（视地区可能需要代理/科学上网；本机 git 那类代理常年失效，先确认代理真的在听）
- 路径 A：本机有 `yt-dlp`（`pip install -U yt-dlp`，按下载规则手动装，不自动拉取）
- 路径 B：本机 Edge/Chrome 里 TikTok **已登录**，且有 Python（含 `websockets`）、curl、ffmpeg
- 只用用户自己已有的访问权限；**不绕过验证码、登录墙、年龄门或地区限制**

## 执行流程 · 路径 A：yt-dlp（首选）

1. 解析短链：`vm.tiktok.com/xxxx`、`vt.tiktok.com/xxxx` 会 302 到完整视频页，yt-dlp 能直接吃短链，一般不用手解。
2. 下载：
   ```bash
   yt-dlp -o "%(id)s.%(ext)s" "https://www.tiktok.com/@user/video/<id>"
   ```
3. 图集（photo-mode）：yt-dlp 新版会把 slideshow 识别成多 item 播放列表，必要时加 `--playlist-items`；下完用 ls 核对张数。
4. ffprobe 核对视频分辨率/时长/编码。
5. 若 yt-dlp 报 403 / "Login required" / 空数据 / extractor 报错 → 转路径 B。

## 执行流程 · 路径 B：CDP 登录态提取法（兜底）

原理与站点无关：让浏览器自己把视频渲染出来，再读 `<video>` 标签的真实地址。步骤沿用抖音法，只把域名/Referer/直链特征换成 TikTok：

1. **强杀浏览器进程**（Edge 为例：`Stop-Process -Name msedge -Force`），否则调试参数不生效。
2. **带调试端口重启并打开目标页**（端口避开常被屏蔽的 9222，用 9333）：
   ```
   msedge.exe --remote-debugging-port=9333 --remote-allow-origins=* "https://www.tiktok.com/@user/video/<id>"
   ```
   ⚠️ **Chromium/Edge 136+ 禁止在默认 user-data-dir 上开调试端口**（参数被静默忽略）。绕过法：`mklink /J C:\edgeprof "%LOCALAPPDATA%\Microsoft\Edge\User Data"` 建一个**无空格 junction** 指向真实 profile，再带 `--user-data-dir=C:\edgeprof` 启动即可保留登录态；**收尾必须 `rmdir C:\edgeprof` 只删联接**（删前用 `fsutil reparsepoint query C:\edgeprof` 确认 Attributes 含 ReparsePoint；严禁 `rm -rf`/`Remove-Item -Recurse`，那会顺着联接毁掉用户真实 Edge 配置）。
3. 等 14~20 秒，访问 `http://127.0.0.1:9333/json` 拿 page target 的 `webSocketDebuggerUrl`。
4. CDP `Runtime.evaluate` 执行 `document.querySelector('video').currentSrc` → 得到 TikTok CDN 直链（域名形如 `vXX-webapp.tiktok.com`、`*.tiktokcdn.com`、`*.tiktokcdn-us.com` 等）。
5. 若 `currentSrc` 是 `blob:`（MSE/DASH 分离流）：改用 CDP **Network 域**监听收集上述 CDN 域的 media 请求（视频、音频若分离则各下一条，`ffmpeg -c copy` 合并）。
6. 下载时带浏览器 UA + **`Referer: https://www.tiktok.com/`**；下完 ffprobe 核对。
7. **收尾**：关闭调试实例并正常重启浏览器；把保存路径告知用户；不输出、不留存签名直链和 Cookie。

## 输出

- 视频：`<id>.mp4`（下载到任务临时目录或用户指定目录）
- 图集：photo-mode 逐张图片，走同一会话提取 `img` 资源地址下载

## 失败处理

- 打不开 tiktok.com：多半是网络/代理不通 → 如实报告，先让用户确认浏览器里能手动打开该视频。
- 登录墙 / 年龄门 / 私密 / 已删除：如实报告，不尝试绕过。
- `currentSrc` 为空且 Network 无 media 请求：页面没加载完或登录态失效 → 让用户手动打开确认可播再重试一次。
- 9333 端口无响应：确认浏览器进程真被杀干净、调试参数生效（`netstat -ano | findstr 9333`）。
- 用户手动给了本地视频路径：直接当本地文件进后续流程，不必走下载。
