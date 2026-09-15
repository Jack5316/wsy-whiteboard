---
name: wsy-whiteboard
description: |
  use this when the user wants a Chinese whiteboard hand-drawn video. Three entry points: (1) they give an SRT, follow the original srt-whiteboard-animation flow; (2) they give artwork/source material without usable subtitles, rewrite spoken 旁白 to match what is actually drawn, MiniMax-clone TTS, then render the hand-drawn video on those audio timestamps; (3) they give audio/video and want the original voice kept — ASR with real timestamps, draw from spoken intent (not a video frame), render, mux original audio. Also use when they mention 白板手绘、字幕做动画、按画面重写讲解、按口播出图、原声白板、玉树芝兰笔杆.
---

# 玉树芝兰白板动画（SRT / 素材 / 音视频原声）

把内容做成暖米黄纸张底的白板手绘成片。入口有三种，先判断再开做，不要混用。

| 用户给了什么 | 走哪条 |
|---|---|
| **SRT 字幕**（可加参考视频） | **路径 A**：原 `srt-whiteboard-animation` 流程（读字幕 → 策略 → 线稿 → 标注 → 渲染） |
| **素材**（线稿/插图/分镜图/已有画面，没有可直接用的 SRT） | **路径 B**：按画面重写中文讲解词 → MiniMax 克隆声 cue-split → 用音频时间戳写 annotation → 再手绘渲染并 mux |
| **音频/视频**（要保留原声，按说的话画） | **路径 C**：抽 wav → ASR 真实时间戳 → 校对 SRT → parse_srt 分镜 → 按口播意图出线稿（禁止描帧）→ 渲染 → mux **原声** |

路径 B 就是「Nate harness 中文版」那种做法：旁白描述**画面上实际画了什么**，不是翻译原片英文；时序以 TTS 实测时长为准，不是先拍 31 秒再硬套音频。

路径 C 反过来：时序以 **ASR 真实时间戳**为准，画面表达**嘴里说的物体/关系**，成片叠**原声**。不要用路径 B 的重写旁白 + MiniMax 换声，除非用户明确要求换声。

依赖（已有则调用，不要复制实现）：

- 渲染：`srt-whiteboard-animation`（`render_stream_whiteboard.py` 等）
- 配音（仅路径 B）：`minimax-tts` 的 **方式 C** `tts_cued.py`
- 字幕（路径 C 优先）：`video-subtitler`（若可用）；否则 faster-whisper；再否则 DashScope

渲染器目录优先级：`~/.claude/skills/srt-whiteboard-animation/` → 当前项目里已 clone 的副本 → 再 `git clone https://github.com/geeklee/srt-whiteboard-animation`。配音脚本固定 `~/.claude/skills/minimax-tts/scripts/tts_cued.py`。

## 笔杆标识（强制，三条路径都要）

默认手部素材是本 skill 的 `assets/drawing-hand.png`，笔杆只写 **玉树芝兰** 四字。

- 禁止使用上游默认贴图里的「江哥是老登啊」或任何第三方口头禅/水印。那是 `geeklee/srt-whiteboard-animation` 作者印在默认 `drawing-hand.png` 上的，不是场景内容，也不是用户品牌。
- 渲染命令的手部 PNG 参数必须指向本 skill 这份贴图，不要指向上游仓库的 `assets/drawing-hand.png`，除非已经确认笔杆是「玉树芝兰」。
- 开渲前看一眼笔杆。还是上游那句就停，先换图再渲。
- 场景源图仍然禁止出现文字；笔杆品牌是唯一允许叠在手部素材上的字。用户若点名别的四字品牌，再换，不要擅自改回上游。
- **不要覆盖**本 skill 的 `assets/drawing-hand.png`，除非用户明确要求换品牌。

## 密钥（不要打印）

```bash
set -a
# 本机云电脑常见：medium-secrets.zsh；VPS Medium：~/.config/secrets.zsh
[ -f "$HOME/.config/secrets.zsh" ] && . "$HOME/.config/secrets.zsh"
[ -f "$HOME/.config/medium-secrets.zsh" ] && . "$HOME/.config/medium-secrets.zsh"
set +a
test -n "$MINIMAX_TTS_API_KEY" && echo tts_key=yes || echo tts_key=no
test -n "$MINIMAX_VOICE_ID" && echo voice_id=yes || echo voice_id=no
test -n "$DASHSCOPE_API_KEY" && echo dashscope=yes || echo dashscope=no
```

- 路径 B 克隆声走**国际版** `$MINIMAX_TTS_API_KEY` → `https://api.minimax.io/v1/t2a_v2`
- **不要**用 `$MINIMAX_API_KEY`（国内 `api.minimax.chat`，不支持克隆声，会 2049）
- 音色 `$MINIMAX_VOICE_ID`
- 路径 C 的 DashScope 兜底用 `$DASHSCOPE_API_KEY`（Paraformer / Fun-ASR 文件转写）
- 禁止把 key / voice id 写进 `/workspace`、成片旁路文件或对话

## 路径 A：用户给了 SRT

按 `srt-whiteboard-animation` 的原流程做，确认关卡也按它的 SKILL.md（每步停下来等用户确认，除非用户这轮已经明确说跳过确认、直接做完）。

摘要：

1. `parse_srt.py` 按 25–35 秒/幕出策略，停，等确认。
2. 按它的「统一出图视觉规范」出 16:9 暖米黄线稿，画面上不要字。停，等确认。
3. 先读该幕字幕再看图，写 `<图名>.annotation.json`，打开预览台。停，等确认。
4. 预览图 → 预览台微调 → 命令行渲染。
5. 手部 PNG 换成**本 skill** 的 `assets/drawing-hand.png`。
6. 若用户还要配音：用 SRT 文本做 `tts_cued.py`（每区域一句），再按路径 B 的第 4–6 步对时序和 mux。不要拿原片英文音轨硬叠。

## 路径 B：用户给了素材（无可用 SRT）

用户给图、分镜、或「按这张图画白板 + 中文讲解」。不要去找原片 SRT 来驱动时间轴。

### 1. 看懂画面

打开每张源图。列出从左到右、绘制顺序上独立的 3–6 个区域（人物、关键物体、关系/框架）。每个区域给稳定 `id`（英文短横线，如 `worker-context`）。**讲解词必须 1:1 对着这些区域**，禁止写画面上没有的东西。

### 2. 重写中文讲解词

口语、王树义风格：短句、把关系说清楚、不堆术语。每区域一句（可逗号连两小句，仍算一个 cue）。除用户点名保留的专名（如 harness）外不写英文；专名加 `pronunciation_dict`。

写出 `spec.json`，**不要**把 key 或 voice_id 写进去：

```json
{
  "slug": "scene-01-name",
  "silence_gap_ms": 180,
  "pronunciation_dict": ["harness/哈尼斯"],
  "cues": [
    {"id": "worker-context", "text": "……"},
    {"id": "model-brain-jar", "text": "……"}
  ]
}
```

`cues[].id` 必须和 annotation 元素 `id` 一致。

### 3. MiniMax 克隆声（方式 C）

```bash
python3 ~/.claude/skills/minimax-tts/scripts/tts_cued.py \
  <spec.json> <out_dir>
```

成功标志：`<slug>.mp3` 体积 > 10KB，`<slug>.cues.json` 里每句都有正的 `start`/`end`。网络失败就停，不要用 espeak 或其他 TTS 冒充。

### 4. 用音频时间戳写 annotation

对每个与 cue id 相同的元素：

- `reveal.startMs` = round(cue.start * 1000)
- `reveal.durationMs` = round((cue.end - cue.start) * 1000)
- 串行、不重叠；180ms 换气留白可以保留
- `sceneDurationMs` = round(音频秒数 * 1000) + 500（收尾凝视）
- `subtitle` = 该 cue 的中文原文
- `canvas` 必须等于原图像素宽高；`region` 整数像素，先看图再标

### 5. 手绘渲染（无声）

```bash
<ENV_PY> <renderer>/scripts/render_stream_whiteboard.py \
  <图.png> <图.annotation.json> <无声.mp4> \
  ~/.claude/skills/wsy-whiteboard/assets/drawing-hand.png \
  --ink-path grid --color-fill contour-wipe
```

`<ENV_PY>` 用渲染器目录下 `.venv/bin/python`（opencv/numpy/av/Pillow）。没有就跑它的 `scripts/prepare_env.py`。

### 6. mux

```bash
ffmpeg -y \
  -i <无声.mp4> -i <out_dir>/<slug>.mp3 \
  -c:v copy -c:a aac -b:a 128k -shortest \
  <成片.mp4>
```

`-shortest` 会切掉片尾 500ms 静帧，可接受。ffprobe 必须同时有视频和音频。

## 路径 C：用户给了音频/视频（保留原声）

用户给 m4a / wav / mp3 / mp4 等，要「按说的话画白板」并且**保留原声**。短于约 35 秒可以一幕。

### 1. 抽单声道 wav

```bash
ffmpeg -y -i <输入音视频> -vn -ac 1 -ar 16000 asr/source.wav
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 asr/source.wav
```

记下 `audio_ms`（秒 × 1000）。视频输入也只抽音轨，**不要**把某一视频帧当线稿来描。

### 2. ASR → 带真实时间戳的 SRT

**禁止**把全文均分、按字数切假时间轴。引擎按这个顺序试，记下哪个成功：

1. **video-subtitler**（若本机 skill 可用，且素材是口播视频/适合它的词级 ASR 流程）→ 产出带真实时间戳的 SRT。
2. 否则 **faster-whisper**（装在渲染器 `.venv`；中文 `--language zh`，否则 auto。要 word timestamps，再按语义切成短 cue 写成 SRT）。短片可用 `small`/`base`；更长口播再上 `large-v3-turbo`。
3. 否则 **DashScope** Paraformer / Fun-ASR（`$DASHSCOPE_API_KEY`，文件转写，用返回的句级时间戳写 SRT）。

全部失败就**停并报告**，不要编 SRT。

写出 `asr/raw.srt`，再轻度校对为 `asr/final.srt`：只修同音字、专名、简繁；**不改写**句子；**保留时间戳**。校对时可参考无 VAD 的二次识别，用来补被 VAD 切掉的词，但时间仍须来自引擎，不是拍脑袋。

### 3. 分镜

```bash
<ENV_PY> <renderer>/scripts/parse_srt.py asr/final.srt --target-sec 30 --min-sec 25 --max-sec 35
```

按 25–35 秒/幕。整段短于约 35 秒就一幕。

### 4. 按口播意图出线稿（禁止描帧 / rotoscope）

读 `final.srt`。决定 2–6 个视觉区域，表达**嘴里说的物体和关系**，不是原视频里碰巧出现的画面。16:9，米色纸 `#F5EBD7`，深灰线稿，红/橙/蓝仅少量点缀，**场景图上不要字**。区域之间留白，方便后面画框。

用 GPT Image 2（YouMind / ListenHub / `gpt-image-2-router`）出图。不要用 matplotlib/PIL 手搓当默认。

### 5. annotation

先看图再标。`canvas` = 该 PNG **实际像素**宽高。每个元素：

- 稳定 `id`（英文短横线）
- `region` 整数像素，覆盖对应可见主体
- `reveal.startMs` / `durationMs` 覆盖该区域对应的 SRT 跨度；幕内**串行、不重叠**（可留 100–300ms 换气）
- `subtitle` = 该区域的口播中文（来自 final.srt）
- `sceneDurationMs` = `audio_ms + 500`（收尾凝视）

### 6. 手绘渲染（无声）+ mux 原声

手部 PNG 必须是本 skill 的 `assets/drawing-hand.png`（笔杆 **玉树芝兰**）。

```bash
<ENV_PY> <renderer>/scripts/render_stream_whiteboard.py \
  <图.png> <图.annotation.json> <无声.mp4> \
  ~/.claude/skills/wsy-whiteboard/assets/drawing-hand.png \
  --ink-path grid --color-fill contour-wipe

ffmpeg -y \
  -i <无声.mp4> -i asr/source.wav \
  -c:v copy -c:a aac -b:a 128k -shortest \
  <成片.mp4>
```

**不要** MiniMax 替换原声，除非用户明确要求换声。ffprobe 必须同时有视频和音频。

## 视觉规范（三条路径共用）

与上游白板 skill 一致：米色纸 `#F5EBD7`、深灰线、红/橙/蓝仅少量点缀；场景源图无文字。画法 ink:color = 2:1，默认 `--ink-path grid`、`--color-fill contour-wipe`。

## 目录

```text
assets/whiteboard/<项目>/
  asr/source.wav
  asr/raw.srt
  asr/final.srt
  scene-01-<名>.png
  scene-01-<名>.annotation.json
  scene-01-<名>-whiteboard.mp4   # 无声手绘
  scene-01-<名>.mp4              # 成片（路径 B 含 TTS；路径 C 含原声）
  vo/spec.json                   # 仅路径 B
  vo/<slug>.mp3
  vo/<slug>.cues.json
```

## 同步（写完这个 skill 之后）

本 skill 的真相源是 `~/.claude/skill-registry/wsy-whiteboard/`。改完必须：

1. 同步到 `~/.claude/skills/wsy-whiteboard/`（**保留** `assets/drawing-hand.png`，不要覆盖成上游那张）
2. 在 `skill-registry` 里 commit + push
3. 本机 `chezmoi apply`（让 after 脚本拉齐 skills）——若刚刚改过 chezmoi 管理的配置、apply 会误伤，就改成：registry `git pull --ff-only` 再把 `SKILL.md` 拷到 `~/.claude/skills/wsy-whiteboard/`，不动 assets
4. SSH 到 VPS Medium：优先 `cd ~/.claude/skill-registry && git pull --ff-only`，再拷 `SKILL.md` 到 `~/.claude/skills/wsy-whiteboard/`（保留 assets）。不要在刚改过的机器上盲目 `chezmoi apply` / `chezmoi update --apply`

不要只改 `~/.claude/skills/` 副本，下次 skill-sync 会冲掉。
