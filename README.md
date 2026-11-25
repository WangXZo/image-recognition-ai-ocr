# image-recognition-ai-ocr

> 仅做图片识别，拿到图片，返回文字。业务判断归下游。

把**图片**变成**可判断的结构化数据**：

```
图片（截图 / 海报 / 表格 / 文档照片 / 游戏画面）
   ↓
结构化结果（文字 / 标签 / 置信度 / 特征 / 分类 / 降级说明）
   ↓
下游自己判断怎么用
```

**它不替你决定**：要不要读、读到什么算够、结果怎么用 —— 都是调用方的事。

---

## 三类任务

| 任务 | 你给什么 | 它回什么 | 耗时 |
|---|---|---|---|
| `match` **判断** | 意图清单 + 图片 | 每意图的**置信度** + 一句话依据（≤20 字） | 最短，≈2s/图 |
| `text` **读字** | 图片 | **全量文字**（不限字数）+ 标签 + 字数 | ≈2s/图 |
| `feature` **认特征** | 图片 + 任务说明（+ 模板库） | 特征列表 + 描述（默认 ≤200 字，`maxDescChars: 0` = 不限） | 5~8s/图 |

**不传任务就是 auto**：先 OCR（免费），文字够（≥20 字）就当文字图出 `text`，文字不够就当画面图出 `feature` 描述；
你给了 `intents` 就再补一次 `match`。想强制某一种就显式传 `task`。

## 快速开始

```js
import { recognize } from './scripts/index.mjs';

// auto：只管给图，它自己判断该读字还是该描述
const r = await recognize({ images: ['https://x/a.jpg', 'https://x/b.jpg'] });
for (const row of r.results) console.log(row.image, row.tasks, row.text, row.description);
```

```js
// 判断：只要判断，不读字、不描述
const r = await recognize({
  task: 'match',
  images: ['https://x/a.jpg'],
  intents: [
    { id: 'error_dialog',  label: '错误弹窗' },
    { id: 'activity_page', label: '活动页面' },
  ],
});
for (const row of r.results) console.log(row.image, row.matches[0].label, row.matches[0].confidence);
```

## 输出

`results` **永远是数组、一图一条**：

```json
{ "ok": true, "mode": "auto",
  "results": [ { "image": "img1", "tasks": ["text"], "text": "...", "chars": 214,
                 "labels": ["含中文"], "matches": null, "features": [], "description": "" } ],
  "meta": { "methods": [], "degraded": [], "skipped": [], "ms": 0 } }
```

没跑的任务字段是 `null` / 空，不存在的块。
**一切识别失败降级不抛错**，`ok:false` 只在参数非法时出现。

## 安装

```bash
git clone https://github.com/WangXZo/image-recognition-ai-ocr.git
cp -r image-recognition-ai-ocr ~/.dsh/skills/     # DSH
cp -r image-recognition-ai-ocr ~/.openclaw/skills/ # OpenClaw
```

本地服务（OCR / 模板匹配 / 缩图）：

```bash
python scripts/server.py            # 默认 127.0.0.1:8791
curl http://127.0.0.1:8791/health    # {"ok":true,"engine":"rapidocr","opencv":true}
```

看图渠道（VLM）三选一，按优先级自动回落：
① 注入 `options.vision.chatVision` → ② 自备 KEY（`IR_VISION_*` / `OPENAI_API_KEY`） → ③ 复用 DSH 模型路由

依赖与排障详见 [docs/install.md](docs/install.md)。

## 文档

| 文档 | 内容 |
|---|---|
| [SKILL.md](SKILL.md) | skill 说明书（给 AI 读的） |
| [docs/overview.md](docs/overview.md) | 介绍：做什么 / 不做什么 / 为什么这么设计 |
| [docs/install.md](docs/install.md) | 安装、依赖、服务启动、渠道配置、排障 |
| [docs/api.md](docs/api.md) | 完整输入输出契约（给下游对接用） |

## 自测

```bash
node test/test-all.mjs      # 209 断言，全离线（mock fetch，不打网络）
```

## 与主 skill 的关系

`image-recognition-ai-ocr` 是**通用能力**，与任何业务无关。
`WangXZo/taptap-pvz2-forum-moderator`（TapTap 论坛版主助手）是它的**一个使用方**：
那边自己做业务门控（求码帖不读图）、注入自己的看图 prompt 与模板库，再把识别结果回灌到帖子正文。

两边可各自独立安装与使用。
