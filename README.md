# latent-media

一个零依赖的命令行封装，把 [Latent Audio](https://audio.latent.moe/docs/api) 的语音合成和 [Latent](https://latent.moe/docs/api) 的文生图串成两条命令；同时是一个可直接被 agent 加载的 skill（`SKILL.md`）。

- **语音**：`POST /v1/audio/speech`，OpenAI 兼容，音色按 slug 指定，台词里可内嵌 `[笑]` `[叹气]` `[耳语]` 等气声事件。
- **图像**：`POST /api/generate` 异步排队 → 轮询 → 自动下载原图；另有公开图库的 `resolve` 接口可按 tag 找参考图。
- 纯标准库，Python 3.8+ 直接跑，无需 `pip install`。

## 快速开始

```bash
# 1. 密钥（写进 keys.env，该文件已被 .gitignore 忽略）
cp keys.env.example keys.env
python scripts/latent.py keys --audio lat_tk_xxx --image lat_sk_xxx

# 2. 自检
python scripts/latent.py doctor

# 3. 出声
python scripts/latent.py tts --voice <slug> --text "灯笼已经点上了，她才走到桥边。" -o line.mp3

# 4. 出图
python scripts/latent.py image --prompt "1girl, solo, watercolor pencil (medium)" -o pic.png
```

key 也可以走环境变量 `LATENT_AUDIO_KEY` / `LATENT_IMAGE_KEY`，或用 `--key` 单次传入。

## 命令

```
tts      --voice <slug> (--text T ... | --text-file F [--split-lines]) [-o OUT] [--format mp3|wav|opus]
image    --prompt P [--negative N] [--resolution square|portrait|landscape] [--steps 8-12] [--seed N] [-o OUT] [--no-wait]
job      <id> [--download OUT]         # 查任务 / 取图
cancel   <id>
status                                 # 图像队列与 worker 状态
resolve  --tag T [...] [--source ...] [--download OUT]
doctor | keys
```

`tts` / `image` 支持 `--json`，输出里带 `file`、`seed`、`duration_ms`、`artwork_id`，方便脚本接。

## 细节与坑

都在 [`SKILL.md`](./SKILL.md) 里，包括：

- 语音 slug 只能从语音库拿，**API 没有 voices 列表接口**，且未发布的语音一律 `404`；
- 气声事件全表、配额（14400 秒/周、4 并发、单次 300 秒）与图像限制（1 任务、8–12 步、三档尺寸）；
- 幂等键的意义：重试复用同一个 `Idempotency-Key` 才不会重复扣额度；
- 语音 `504` 时改走 `/api/jobs/{id}/audio` 领取；
- 错误码对照表（`403` 少 User-Agent、死代理端口等）。

## 安装成 skill

把整个目录放进 skill 目录即可：

```bash
# 用户级
cp -r latent-media ~/.workbuddy/skills/
# 或项目级
cp -r latent-media <project>/.workbuddy/skills/
```

之后 agent 会在需要配音/出图时自动读到 `SKILL.md`。

## 密钥安全

- 真实 key 只存在于 `keys.env`、环境变量或 `--key`，三者都不进仓库（见 `.gitignore`）。
- 提交前自查：`git grep -nE "lat_(tk|sk)_[A-Za-z0-9_-]{10,}"` 应该什么都搜不到。

## 声明

第三方封装，与 Latent 官方无隶属关系。额度、条款与内容政策以站点为准（语音模型权重为 CC-BY-NC，商用需另行确认）。
