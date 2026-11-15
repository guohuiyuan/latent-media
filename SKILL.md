---
name: latent-media
description: "用 Latent 的两个云端 API 出声音和出图：audio.latent.moe 的音色库（165+ 个社区音色，按 slug 指定）+ /v1/audio/speech 做多角色配音（一句台词里可内嵌 [笑] [叹气] [耳语] 等气声事件），latent.moe 的 /api/generate 做文生图（异步排队，正方形/竖版/横版三档）。当用户要求「用某个声音念这段」「给这句话配音」「换个音色」「生成一张插画/配图」「按 tag 找张参考图」时使用。密钥与 cookie 放 keys.env，不入库。"
display_name: "Latent 生成（语音·图像）"
agent_created: true
version: 1.1.0
tags: [tts, 语音合成, 配音, 声音克隆, 音色库, text-to-speech, 文生图, image-generation, latent, audio.latent.moe, latent.moe]
---

# Latent 生成（语音 + 图像）

一个零依赖脚本包住两个云端服务：

| 服务 | 干什么 | 端点 |
|---|---|---|
| Latent Audio | 165+ 个社区音色的文本转语音，台词内可嵌情绪/气声事件 | `https://audio.latent.moe/v1/audio/speech` |
| Latent | 文本 → 图像（异步排队），另可读公开图库 | `https://latent.moe/api/generate` |

脚本：`scripts/latent.py`，只用标准库，Python 3.8+ 直接跑，**不需要 pip install**。

## 先跑 30 秒自检

```bash
python scripts/latent.py doctor
```

`audio.ok` / `image.ok` 都是 `true` 才往下走。key 报错时看最后一节「故障对照」。

## 密钥

优先级：`--key` > 环境变量 > `keys.env`（本目录） > `~/.config/latent/keys.env`

```bash
# 写进 keys.env（已被 .gitignore 忽略）
python scripts/latent.py keys --audio lat_tk_xxx --image lat_sk_xxx
```

- 语音 key 前缀 `lat_tk_`，图像 key 前缀 `lat_sk_`，**两者不能混用**。
- 语音 key 30 天过期，重新登录站点会续期；一个账号最多 5 把。
- 绝不要把 key 或 cookie 写进 SKILL.md、示例命令或提交记录里。

## 命令

| 命令 | 用途 |
|---|---|
| `voices` | 列音色库（`-q` 搜、`--lang`、`--sex`、`--json`），读本地缓存 |
| `tts` | 文本转语音，可多段一次出多个文件 |
| `image` | 提交渲染 → 轮询 → 自动下载 |
| `job <id>` | 查某个图像任务，`--download` 顺带取图 |
| `cancel <id>` | 取消图像任务 |
| `status` | 图像队列/worker 状态（提交前先看一眼） |
| `resolve` | 按 tag 从公开图库挑一张现成的图（唯一免 key 的接口） |
| `doctor` | 检查两把 key 与两个服务 |
| `keys` | 查看/写入 `keys.env`（含 `--cookie`） |

`tts` 和 `image` 都默认把产物落到 `./out/`，都能加 `--json` 拿到机器可读结果（含 `file`、`seed`、`duration_ms`、`artwork_id`）。

## 音色库：先挑声音

```bash
python scripts/latent.py voices                      # 前 30 个（按上传时间倒序）
python scripts/latent.py voices -q 台湾              # 按 slug/名字/作者/描述搜
python scripts/latent.py voices --lang zh --sex Female --limit 10
python scripts/latent.py voices --json               # 全量结构化，含 uuid
```

- 音色库是 audio.latent.moe 的首页，条目地址形如 `/v/<slug>`，**slug 就是 `--voice` 要的值**；也可以用条目 uuid。
- 列表要**站点登录 cookie**（Discord 登录后复制），只此一用：合成、出图都只认 API key，不需要 cookie。
- 首次运行会自动抓全量并缓存到 `voices.json`（不入库）。之后读缓存，秒回；新音色用 `--refresh` 重抓，或临时 `python scripts/latent.py keys --cookie '...'` 写进 `keys.env`。
- **API 没有公开的 voices 列表接口**，列表只能靠登录态解析首页；直接猜 slug 会得到 `404 voice_not_found`。

## 例子

**用某个声音念一句台词**

```bash
python scripts/latent.py tts --voice voice-158 \
  --text "灯笼已经点上了，她才走到桥边。" -o line.mp3
```

**带情绪的台词**（事件标签写在文本里，全角方括号，原样照抄）

```bash
python scripts/latent.py tts --voice voice-158 \
  --text "[叹气] 还能听见你的声音，真好。[轻笑] 你一点都没变。" -o line.mp3
```

**要台湾腔/日语腔就换音色**（`voices -q 台湾` 会告诉你 slug）

```bash
python scripts/latent.py tts --voice voice-178 --text "欸，你怎麼這麼慢啦。" -o tw.mp3
python scripts/latent.py tts --voice voice-176 --text "またね、先輩。" -o jp.mp3
```

**一次配多段旁白**（几个 `--text` 就是几段，自动编号）

```bash
python scripts/latent.py tts --voice voice-158 \
  --text "第一段旁白。" --text "第二段旁白。" -o out/narration/
```

**从脚本文件批量配**（每行一段，`--split-lines` 必须带）

```bash
python scripts/latent.py tts --voice voice-158 --text-file lines.txt --split-lines -o out/narration/
```

**直接产 wav 交给视频流水线**

```bash
python scripts/latent.py tts --voice voice-158 --text "开场白。" --format wav -o out/vo.wav
```

**生成一张竖版水彩插画**

```bash
python scripts/latent.py image \
  --prompt "1girl, solo, watercolor pencil (medium), soft light" \
  --resolution portrait --steps 8 -o out/pic.png
```

**同一构图换种子再来一版**

```bash
python scripts/latent.py image --prompt "1girl, solo, watercolor" --seed 1234567 -o out/pic-b.png
```

**只要排队不要等**（长队列时先拿 id，回头取）

```bash
python scripts/latent.py image --prompt "..." --no-wait      # → {"id": "..."}
python scripts/latent.py job <id> --download out/
```

**找一张参考图**

```bash
python scripts/latent.py resolve --tag 1girl --tag watercolor --source comfyui
python scripts/latent.py resolve --tag "nahida_(genshin_impact)" --download out/ref.webp
```

## 语音：气声事件

写在 `input` 里，用**全角**方括号，大小写敏感；不在表里的方括号内容会被当标点读出来。

`[笑]` `[轻笑]` `[咯咯笑]` · 呼吸 `[叹气]` `[惊呼]` `[吸气]` `[呼气]` `[喘息]` · 语气 `[耳语]` `[喊叫]` `[惨叫]` `[唱歌]` `[哼歌]` `[结巴]` `[停顿]` · 哭与痛 `[哭声]` `[抽泣]` `[呜咽]` `[呻吟]` `[哀号]` · 喉鼻 `[清嗓]` `[咳嗽]` `[吸鼻子]` · 口部 `[咂嘴]` `[啧嘴]` · 反射 `[打哈欠]` `[打喷嚏]` `[打嗝]` `[打饱嗝]` `[吞咽]` `[干呕]` · 反应 `[闷哼]` `[冷笑]` `[哼鼻子]`

事件本身计入时长估算，也就是计费。

## 配额与硬限制

| 项 | 语音 | 图像 |
|---|---|---|
| 额度 | 14400 秒/周（滚动），只计成功 | 独立周额度，只计成功 |
| 并发 | 4 个请求 | 1 个任务 |
| 单次上限 | 300 秒估算音频（超了先切段） | prompt 1–2000 字符 |
| 尺寸 | — | `square` 1024×1024 / `portrait` 920×1536 / `landscape` 1536×920 |
| 步数 | — | 8–12，超出被夹紧 |
| 结果保留 | 30 天 | 存成**私有**作品，只有同一把 key 能取 |

- `sampler`：`euler` / `res_multistep` / `er_sde`；`scheduler`：`sgm_uniform` / `beta` / `beta57` / `linear_quadratic`。
- `model` 字段服务端接受但忽略，脚本固定填 `latent-audio`。
- 图像没有站点默认负向词，要排除什么得自己写 `--negative`。

## 重试与幂等

脚本每次调用都生成一个 `Idempotency-Key` 并打印在 `--json` 输出里。**重试请复用同一个 key**，否则会重复排队、重复扣额度：

```bash
python scripts/latent.py image --prompt "..." --idem 7f3c...   # 上一次超时，接着这个 key 再发
```

- 语音 `429 concurrent_limit` / `503 queue_full`：脚本按 `Retry-After` 自动重试 3 次。
- 语音 `504`：响应里带 `job_id`，脚本自动改走 `GET /api/jobs/{job_id}/audio` 领取，不会再扣一次。
- 图像 `409 too_many_active`：同账号只能有一个任务，等它跑完。

## 故障对照

| 现象 | 原因 | 处理 |
|---|---|---|
| `401 invalid_api_key` | key 抄错 / 已撤销 / 两个前缀用反 | 重新生成，注意 `lat_tk_` vs `lat_sk_` |
| `404 voice_not_found` | slug 不对或语音未发布 | `voices -q 关键词` 查真 slug |
| `voices` 报 cookie 失效 | 站点登录态过期 | 浏览器重新登录后复制 cookie，`keys --cookie '...'` |
| `403` 且浏览器能开 | 少了 User-Agent 被 CDN 拦 | 用本脚本（已带 UA），别用裸 urllib |
| `429 quota_exhausted` | 本周额度用完 | 等滚动周刷新 |
| 全站超时、`doctor` 卡住 | 环境里 `http_proxy` 是死端口 | 脚本已先直连、失败才走代理；也可 `unset http_proxy https_proxy` |
| 图/音频下了但打不开 | `-o` 给的扩展名和服务端格式不一致 | 看 stderr 的提示，或省略 `-o` 让脚本按 `Content-Type` 命名 |

## 与本地 VoiceStudio 的关系

本机 `127.0.0.1:3900` 已有 VoiceStudio（OmniVoice，GPU，可克隆音色、离线、免费）。选择顺序：

- **要复刻某个具体音色 / 大批量 / 不想联网** → VoiceStudio。
- **要现成多角色音色、要一句里带笑声叹气、懒得本地跑模型** → 本 skill。

两者产物都是标准 wav/mp3，可以直接喂给 `sweety-handdraw-video` 的 `inputs/narration/`。
