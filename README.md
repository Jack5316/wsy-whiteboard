# wsy-whiteboard

玉树芝兰白板动画 Skill - Chinese Whiteboard Hand-drawn Video Generator

## 功能简介

**wsy-whiteboard**（玉树芝兰白板）：白板手绘视频 Skill。给它素材，它还你一段「一只手在白板上一笔一笔画、边画边讲」的视频，笔杆上印着「玉树芝兰」。

### 三种用法

| 用户输入 | 处理路径 | 说明 |
|---------|---------|------|
| **SRT 字幕**（可加参考视频） | 路径 A | 按经典 `srt-whiteboard-animation` 流程做白板手绘动画 |
| **素材**（线稿/插图/分镜图，无 SRT） | 路径 B | 按画面重写中文讲解词 → MiniMax 克隆声配音 → 用音频时间戳渲染 |
| **音频/视频**（保留原声） | 路径 C | ASR 提取真实时间戳 → 按口播意图出线稿 → 渲染并 mux 原声 |

- 路径 B 需要你自己的 **MiniMax 国际版 API Key** 与克隆音色 ID
- 渲染器首次使用时自动获取开源 `srt-whiteboard-animation`（作者 geeklee）

## 安装方法

### 方法一：直接解压安装（推荐）

1. 解压 `wsy-whiteboard.zip` 文件
2. 将解压后的 `wsy-whiteboard/` 文件夹移动到 Claude Code 的 skills 目录：

```bash
# macOS / Linux
mv wsy-whiteboard ~/.claude/skills/

# 或者使用完整路径
mv wsy-whiteboard /Users/<你的用户名>/.claude/skills/
```

3. 重启 Claude Code 或新开一个会话，Skill 即可生效

### 方法二：使用 unzip 命令

```bash
# 1. 解压到 skills 目录
unzip wsy-whiteboard.zip -d ~/.claude/skills/

# 2. 验证安装
ls ~/.claude/skills/wsy-whiteboard/
```

## 使用方法

安装完成后，在 Claude Code 中可以通过以下方式触发：

- 输入 `/wsy-whiteboard` 直接调用
- 或在对话中描述需求，Skill 会自动识别并执行

## 文件结构

```
wsy-whiteboard/
├── README.md           # 本文件
├── SKILL.md            # Skill 主配置文件（供 Agent 读取）
├── 安装说明.md         # 安装与使用说明
└── assets/
    └── drawing-hand.png  # 手部贴图（笔杆印「玉树芝兰」）
```

## 笔杆标识（强制）

- 默认手部素材为本 skill 的 `assets/drawing-hand.png`
- 笔杆只写 **玉树芝兰** 四字
- 禁止使用上游默认贴图里的其他品牌或口头禅
- 渲染命令的手部 PNG 参数必须指向本 skill 的贴图

## 密钥配置

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

## 注意事项

- 确保 `~/.claude/skills/` 目录存在，如不存在请先创建：
  ```bash
  mkdir -p ~/.claude/skills
  ```
- 如果 Skill 不生效，请检查 `SKILL.md` 文件是否完整
- 路径 B/C 需要配置对应的 API Key（MiniMax / DashScope）
- 渲染器依赖会自动获取，无需手动安装

## 相关依赖

- 渲染：`srt-whiteboard-animation`
- 配音（路径 B）：`minimax-tts` 的 `tts_cued.py`
- 字幕（路径 C）：`video-subtitler` / `faster-whisper` / DashScope

## License

本项目基于开源渲染器 `srt-whiteboard-animation` 构建，笔杆品牌归「玉树芝兰」所有。

---

*打包时间：2026-09-03*