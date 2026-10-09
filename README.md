# tiktok-video-distiller — TikTok 视频蒸馏 Skill

**本地离线跑：TikTok 链接 → 下载（yt-dlp 通常无水印）→ 本地 Qwen 视频理解 → 带时间戳知识卡片。TikTok video distiller: link → download → on-device Qwen2.5-Omni video understanding → timestamped, citable knowledge cards.** 全程不调第三方云端模型做理解、无额度限制、无请求体上限。

把 TikTok 短视频**一步到位**蒸馏成可检索的知识卡片：带时间戳的原视频证据、模型归纳、关键概念、行动项，直接落盘为 Markdown/Wiki 格式。支持 Claude Code / OpenCode 等任何读取 `SKILL.md` 规范的 agent。

> **⚠️ 实测状态（先读）**：蒸馏层（本地 Qwen2.5-Omni + `scripts/analyze_omni.py`）与抖音版同源、平台无关，可直接用。但**下载层是从 `douyin-video-download` 的 CDP 法适配而来，尚未在真实 TikTok 环境实测**，且需本机**能访问 tiktok.com**（部分地区需代理）。首次使用请先按 `tiktok-video-download/SKILL.md` 的「状态声明」验证 yt-dlp 可用性、水印情况和 photo-mode 行为。

## 下载环节：yt-dlp 首选 + CDP 兜底（本仓库自带）

与抖音相反——**TikTok 通常不需要 CDP**。`tiktok-video-download/SKILL.md` 给出两条路径：

- **路径 A · yt-dlp（首选）**：yt-dlp 内置 `tiktok` extractor，社区长期维护，通常**免登录即可下载且拿到无水印原流**。一行 `yt-dlp <链接>` 搞定，短链 `vm.tiktok.com` / `vt.tiktok.com` 能直接吃。
- **路径 B · CDP 登录态提取法（兜底）**：当 yt-dlp 被风控（403 / Login required）时，借用本机浏览器（Edge/Chrome）里已登录的 TikTok 会话，通过 Chrome DevTools Protocol 读 `<video>` 真实地址下载。原理与站点无关，抖音那套照样适用，只是域名/Referer 换成 TikTok。

## 特性

1. **一步到位的正确流程**：链接 → 下载 → 视频理解 → 证据分级 → 知识卡片 → 临时文件清理，失败即如实报告、保留现场，不产生"假摘要"。
2. **本地多模态模型直接吃完整视频**：调用本地 **Qwen2.5-Omni-7B**，视频以带时间戳的帧序列原生送入模型（不是抽帧当图片看），音轨默认一起输入（缺依赖自动降级纯画面，stderr 有 `[warn]`）；理解环节不联网、无额度限制、无请求体大小上限。自带 `scripts/analyze_omni.py`。
3. **长视频的正确处理**：靠采样帧率（`--fps`）和 `--max-pixels`（默认 100352）控制耗时与显存，脚本给出 FFmpeg 切段压缩命令（150s/段、720p、CRF32），长视频也能稳定跑通。
4. **证据分级输出规范**：`原视频证据 / 蒸馏结论 / 待确认` 三层分离 + 时间戳，防止模型把单条视频观点升级成客观事实。

## 前置条件

| 依赖 | 必需 | 说明 |
|---|---|---|
| 能访问 `www.tiktok.com` | ✅（仅链接下载） | 部分地区需代理/科学上网；本地视频文件不需要 |
| [Qwen2.5-Omni-7B](https://huggingface.co/Qwen/Qwen2.5-Omni-7B) 权重 | ✅ | 本地推理；4bit 量化后权重常驻约 6GB 显存，8G 显卡实测可跑 |
| Python 3.10+ | ✅ | 需 `torch` / `transformers>=4.57` / `bitsandbytes` / `qwen-omni-utils` / `torchvision>=0.19` / `accelerate`（device_map 必需）/ `audioread`（qwen-omni-utils 传递依赖，国内镜像常缺、需走官方 PyPI）/ `decord`（视频解码后端：**torchvision>=0.26 已删除 `read_video`，不装 decord 或 torchcodec 会直接崩**） |
| `yt-dlp` | ✅（路径 A） | `pip install -U yt-dlp`；TikTok 首选下载器，通常免登录无水印 |
| 浏览器 + TikTok 网页版登录态 | 仅路径 B | yt-dlp 被风控时的 CDP 兜底才需要；Edge/Chrome 里 www.tiktok.com 已登录 |
| FFmpeg、curl、Python `websockets` | ✅（路径 B） | CDP 提取与流合并用 |

### 获取模型权重（约 22GB，四种方式任选，手动自动都行）

```bash
# 方式1 · huggingface 官方（需可访问 huggingface.co）
hf download Qwen/Qwen2.5-Omni-7B --local-dir ./Qwen2.5-Omni-7B

## English

**TikTok Video Distiller** — an offline Agent Skill: a TikTok link in, timestamped knowledge cards out.

- Pipeline: link → download via yt-dlp (usually watermark-free) → local Qwen video understanding → Markdown knowledge cards with timestamped evidence
- Runs **fully offline**; no cloud API
- Install as an Agent Skill (Claude Code / OpenCode / Qoder): clone into your skills directory and say "蒸馏这个 TikTok 链接"

The Chinese documentation above is the authoritative setup guide.


# 方式2 · aifasthub 国内直链（快，支持断点续传，-C - 即续传）
HF_ENDPOINT=https://aifasthub.com hf download Qwen/Qwen2.5-Omni-7B --local-dir ./Qwen2.5-Omni-7B
# 单文件直链格式（浏览器/curl/wget 均可）：
#   https://aifasthub.com/Qwen/Qwen2.5-Omni-7B/resolve/main/<文件名>
curl -L -C - -O https://aifasthub.com/Qwen/Qwen2.5-Omni-7B/resolve/main/config.json

# 方式3 · hf-mirror 镜像
HF_ENDPOINT=https://hf-mirror.com hf download Qwen/Qwen2.5-Omni-7B --local-dir ./Qwen2.5-Omni-7B

# 方式4 · ModelScope（需 `pip install modelscope`）
modelscope download --model Qwen/Qwen2.5-Omni-7B --local_dir ./Qwen2.5-Omni-7B
```

设置模型路径：

```bash
export OMNI_MODEL_PATH="/path/to/Qwen2.5-Omni-7B"     # Linux / macOS
$env:OMNI_MODEL_PATH="D:\Models\Qwen2.5-Omni-7B"      # Windows PowerShell
```

## 安装

把两个 skill 目录放进你的 skills 目录（下载 skill 是蒸馏流程的依赖，两个都要）：

```
# Claude Code
~/.claude/skills/tiktok-video-distiller/
~/.claude/skills/tiktok-video-download/

# OpenCode
~/.config/opencode/skills/tiktok-video-distiller/
~/.config/opencode/skills/tiktok-video-download/
```

## 使用

对 agent 说：

> 把这个 TikTok 视频蒸馏成知识卡片：https://www.tiktok.com/@user/video/1234567890

或直接处理本地视频：

```bash
python scripts/analyze_omni.py video.mp4 "请完整分析视频并输出带时间戳的转录、画面文字、时间线、明确证据、模型归纳和待确认事项。"

# 音轨默认已输入（口播转写开箱即用）；纯画面任务加 --no-audio 关闭
python scripts/analyze_omni.py video.mp4 "把口播逐字转写"

# 输出复读严重时加 --ngram-rep 4（硬禁 4-gram 复读）

# 纯音频文件会自动走音频通道
python scripts/analyze_omni.py audio.wav "这段音频里有人说话吗？"
```

## 输出格式

```markdown
---
title: ""
type: learning
source: ["原始链接"]
confidence: high | medium | low
status: draft
---
## 一句话结论 / 原视频证据 / 蒸馏结论 / 关键概念 / 可执行行动 / 待确认事项
```

## 性能排障（跑得太慢先看这里）

- **生成慢约 10 倍（<1 tok/s）**：**先查输入 token 数**（脚本 stderr 的 `[input] tokens=` 行），不是先看视频时长，也不是先怀疑显卡。缺省不压像素时 1080p 帧可喂进 5000+ token，慢一个数量级且 GPU 频率看着"异常低"（其实是超大输入的表象）。脚本已默认 `--max-pixels 100352`（=128×28²，qwen_vl_utils 硬下限，**传低于它的值如 100000 会直接断言崩溃**）；token 仍超预算就降 `--fps`。确认 token 正常后才查驱动：`nvidia-smi -l 2` 采样 SM 频率钉死低频且无节流标记时，给 python.exe 设「最高性能优先」或 `nvidia-smi -lgc`（跑完 `-rgc` 解锁），笔记本另查混合显卡/MUX。
- **每段循环重启 python 进程**：模型加载要 1–2 分钟，一个进程里加载一次、循环推理多段。
- **transformers 5.x 警告 pixel cap**：输入 token 会远超预期——确认 `--max-pixels` 生效（脚本默认 100352），不要传低于 100352 的值。

## 已知限制

- **下载层未在 TikTok 实测**：本仓库下载逻辑由抖音 CDP 法适配而来。yt-dlp 的 TikTok extractor 通常免登录无水印，但**首次使用务必验证 yt-dlp 可用性、水印情况、photo-mode 行为**，与本文不符就更新本文。
- **网络可达是硬前提**：tiktok.com 在部分地区不可直连，需代理；代理不可用时下载整条链路走不通。本机 git 那类代理常年失效，先确认代理真的在听。
- CDP 兜底依赖浏览器登录态；Chromium/Edge 136+ 禁止在默认 profile 上开调试端口，绕过法见 `tiktok-video-download/SKILL.md` 路径 B 第 2 步。仍拿不到视频时如实报告，可手动保存视频后走本地文件流程。
- 推理速度取决于显卡：RTX 4060 Laptop（8G）下 fps=1、输出 512 token 约需 2 分钟；纯音频约 30 秒。
- 模型**会幻觉**：约第 10 秒输出起可能退化复读、吐 `Human:` 之类角色标签。脚本已默认 `repetition_penalty=1.1`，仍严重时加 `--ngram-rep 4`；采信前先截断复读段。**覆盖校验**：输出时间戳没到视频末尾的，缺口必须标"未分析"——短视频卡死多半是输入 token 超预算（看 `[input] tokens=` 行），先确认 `--max-pixels` 没被关掉，不是降 fps。
- **音轨链路依赖因机器而异**（ffmpeg/audioread/decord）：缺依赖时脚本不会挂掉，会自动降级为纯画面通道并在 stderr 打 `[warn]`——但这次输出就不含任何语音信息，补齐依赖后重跑才是完整蒸馏。
- 画面硬字幕识别可靠；**模型自报的分段时间戳不可信**（实测偏差可达 30s），精确时间轴要用 ffmpeg 按画面字幕实测校正。

## License

MIT
