---
name: gen-images
description: Generate or edit images with gpt-image-2 and save the resulting files. Use when the user asks to "使用 gpt-image-2", "生成图片", "文生图", "修改图片", "编辑图片", "改图", or explicitly invokes `$gen-images`.
---

# gen-images

使用这个 skill 处理通过调用 `gpt-image-2` 的图片生成和图片编辑任务。支持自动触发，也支持用户显式输入 `$gen-images ...`。

## 目标

- 识别用户是要文生图还是改图
- 从用户自然语言中提取可用字段
- 缺少关键字段时先追问用户，不要盲目执行
- 字段足够时调用 `scripts/gen_images.py`
- 完成后向用户输出图片路径和实际使用的关键参数

## 资源

- `references/fields.md`：字段、默认值、自然语言映射、交互规则
- `scripts/gen_images.py`：真正执行图片接口调用和文件保存

## 显式调用

用户显式调用时，直接读取 `$gen-images` 后面的自然语言需求：

```text
$gen-images 生成一张透明背景的猫咪头像，1024x1024，png
$gen-images 把 ./input.png 改成水彩风，保留主体，输出 webp
```

如果是通过自动触发进入本 skill，则直接根据用户原始消息理解需求。

## 任务分类

先判断任务类型：

### 文生图
出现这类意图时，按文生图处理：
- `生成图片`
- `文生图`
- `画一张图`
- `用 gpt-image-2 生成`

### 改图
出现这类意图时，按改图处理：
- `修改图片`
- `编辑图片`
- `改图`
- `把这张图改成...`

如果同时出现图片来源（本地路径、URL、data URL）和修改意图，优先按改图处理。

## 字段提取规则

先参考 `references/fields.md` 的规则提取字段。

### 必填字段

#### 文生图
- `prompt`

如果缺少 `prompt`，先向用户追问：

`请补充图片提示词，例如你想生成什么画面。`

#### 改图
- `prompt`
- 图片来源

图片来源支持：
- 本地路径
- URL
- data URL

如果缺少图片来源，先向用户追问：

`请提供要编辑的图片来源：1）本地路径 2）图片 URL / data URL`

如果缺少修改要求，先向用户追问：

`请补充修改要求，例如你想把图片改成什么效果。`

### 可选字段

如果用户自然语言中包含以下信息，尽量提取并传给脚本：
- `size`
- `quality`
- `background`
- `output_format`
- `n`
- `moderation`
- `output_compression`
- `partial_images`
- `input_fidelity`（改图）

如果用户没有提供，不要为了可选字段反复追问，直接用默认值。

### 自然语言映射

优先识别这些自然语言：
- `高清` -> `quality=high`
- `透明背景` -> `background=transparent`
- `1024x1024`、`1:1` -> `size=1024x1024`
- `1024x1536`、`3:4` -> `size=1024x1536`
- `1536x1024`、`4:3` -> `size=1536x1024`
- `2048x2048` -> `size=2048x2048`
- `3840x2160`、`16:9` -> `size=3840x2160`
- `2160x3840`、`9:16` -> `size=2160x3840`
- `4k横向`、`16:9` -> `size=3840x2160`
- `4k竖向`、`9:16` -> `size=2160x3840`
- `auto` -> `size=auto`
- `png/jpg/jpeg/webp` -> `output_format=...`
- `生成3张` -> `n=3`

如果用户明确要求保存格式，按用户要求保存；否则默认保存为 `png`。

## 执行步骤

### 1. 整理参数

从用户当前消息中整理出：
- `mode`: `generate` 或 `edit`
- `prompt`
- `image`（改图时）
- `mask`（如果用户明确提供）
- 其他可选字段

### 2. 缺字段就停下来问

缺少必填字段时，不要调用脚本。

### 3. 调用脚本

使用 Codex 当前提供的 shell 工具调用 Python 脚本。脚本路径应通过 skill 所在目录推导，不要写死绝对路径。

Windows 优先使用 `py` 启动器；其他系统优先使用 `python3`。启动器不可用时再尝试 `python`。

推荐调用方式：

```text
py "<skill-dir>/scripts/gen_images.py" --mode generate --prompt "..."
```

或：

```text
py "<skill-dir>/scripts/gen_images.py" --mode edit --prompt "..." --image "..."
```

### 3.1 等待与超时

- 只启动一次前台命令并等待它结束，不要放到后台运行。
- 不要轮询、反复询问或重复执行命令来检查是否完成。
- Python 脚本会同步等待接口返回，图片生成成功或接口失败后立即结束。
- 图片请求固定在 600 秒（10 分钟）后超时并返回 JSON 错误。
- Codex shell 工具的外层超时设置为略高于脚本超时，例如 `timeout_ms=610000`，让脚本有时间输出结构化超时结果。

根据已提取到的字段，继续附加参数：
- `--size`
- `--quality`
- `--background`
- `--output-format`
- `--n`
- `--moderation`
- `--output-compression`
- `--partial-images`
- `--input-fidelity`
- `--mask`

## 脚本行为

`scripts/gen_images.py` 会：
- 先基于 `scripts/gen_images.py` 自身所在目录向上逐级检查祖先目录名是否为 `.codex` 或 `.claude`，据此判断当前调用者
- 找到 `.codex/` 时按 Codex 调用处理，优先读取 `~/.codex/config.toml` 中当前 `model_provider` 的 `base_url` 和 `env_key`
- 找到 `.claude/` 时按 Claude 调用处理，读取 `~/.claude/settings.json` 中的 `env.ANTHROPIC_BASE_URL` 与 `env.ANTHROPIC_AUTH_TOKEN`
- Codex 调用时从 provider 指定的环境变量、`OPENAI_API_KEY` 环境变量或 `~/.codex/auth.json` 中读取令牌，并使用 `OPENAI_BASE_URL` 作为备用地址
- 如果无法判定当前调用者，则按回退顺序先尝试 Claude 配置，再尝试 Codex 配置
- 使用 `Authorization: Bearer <token>` 调用接口
- 根据 `base_url` 是否已经以 `/v1` 结尾构造图片接口地址，避免重复添加版本路径
- 单次请求同步等待最多 600 秒，不轮询接口状态
- 将返回图片保存到当前工作目录下的 `./gen-images/`
- 输出 JSON 结果

## 结果处理

脚本成功时会输出 JSON，例如：

```json
{"ok": true, "paths": ["..."], "used_params": {"model": "gpt-image-2", "size": "1024x1024", "quality": "high", "output_format": "png", "n": 1}}
```

脚本失败时会输出 JSON，例如：

```json
{"ok": false, "error": "缺少 prompt"}
```

### 成功回复格式

向用户输出：

- `图片已生成, 图片路径: <路径>`
- `实际使用的关键参数: model=..., size=..., quality=..., output_format=..., n=...`

如果生成多张图片，列出所有路径。

### 失败回复格式

向用户输出：

- `生成失败: <简短错误原因>`

## 注意事项

- 不要在缺少必填字段时猜测用户意图
- 不要为可选字段做冗长说明
- 改图时，本地路径、URL、data URL 都要支持
- 除非用户明确要求，不要增加接口里没有的自定义字段
- 图片生成命令启动后保持等待，不要通过重复询问或轮询检查进度
- 调用完成后，优先返回结果，不要输出多余解释
