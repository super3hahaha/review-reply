---
name: review-reply
description: >
  为 Google Play 应用评论批量生成回复。输入是一个 JSON 文件路径（含 target_language +
  groups[].reviews[]），输出同名 *.candidates.json。命中模板→返回 1 条（翻译好的模板）；
  未命中→matched:false，留给用户在 app 里单独处理。由 tester-app 通过
  「/review-reply <json路径>」调用；只做批量匹配，不在对话里现编回复。
---

# Review Reply Skill

为五个 app 的 Google Play 用户评论**批量**生成回复。**以"匹配模板"为主**：

- **命中模板（高置信 ≥0.9）** → 直接用该模板，**只翻两份**（目标语言正文 + 中文预览），**不生成、不凑多条**。
- **没命中** → **跳过**，标记 `unmatched`，**不生成**。这些评论由用户在 app 里单独处理（app 内有「🤖 AI 回复」单条现生成，不走本 skill）。

> 为控成本，匹配阶段只读紧凑索引 `index.json`（全部模板的 id+category，无全文），命中后才按 id 从 `templates.json` 取该模板全文翻译。避免把全量模板全文塞进上下文、也不再给每条评论生成多条候选（早期那样做实测 6 条评论 ~$1 / ~9 分钟）。

> **模板数据目录由调用方（tester-app）在 prompt 里给出**，形如「模板数据目录：/Users/.../.tester-app/templates」。`index.json` / `templates.json` / `package_map.json` 三个文件都从**这个目录**读，不再用 skill 自带的 `data/`。该目录由 tester-app 的「模板管理」页维护（增删改 + 自动重建索引）。

> 本 skill 是**非交互的批量调用**：唯一入口是一个以 `.json` 结尾、顶层有 `groups` 字段的文件路径（由 tester-app 写好后通过 prompt 传入）。不在对话里逐条现编回复。

---

## 输入

调用方（tester-app）写一个 JSON 文件，路径通过 prompt 传给 skill。文件结构：

```json
{
  "target_language": "en",
  "channel": "gp",
  "groups": [
    {
      "package_name": "files.fileexplorer.filemanager",
      "display_name": "File Manager",
      "reviews": [
        {
          "review_id": "abc123",
          "text": "广告太多，烦死了",
          "original_text": "Too many ads",
          "star_rating": 1,
          "reviewer_language": "en",
          "app_version_name": "2.3.1",
          "device": "Pixel 7",
          "android_os_version": 13
        }
      ]
    }
  ]
}
```

字段说明：
- `target_language`：ISO 代码（en / zh-CN / ru / pt / ...）或 **`"auto"`**。
  - 具体 ISO 代码 → 本次所有候选回复统一用这个语言。
  - **`"auto"`（逐条跟随评论语言）** → **每条评论各自决定回复语言**：优先用该评论的 `reviewer_language` 字段；`reviewer_language` 为空/不可信时，从 `original_text`（无则 `text`）判断评论本身的语言。这样一次调用就能给跨语言的一批评论各回各的母语。判定不了（混合多语 / 全 emoji）的那一条退回英文，并在 `warnings` 记一笔。
- `channel`：`"gp"`（Google Play 回复，**默认值**，受 350 字符限制）或 `"email"`（邮件回复，无字符限制）。字段缺省时按 `"gp"` 处理。
- `text`：评论文本（可能被 tester-app 翻译成中文，因 `translationLanguage: "zh-CN"`）
- `original_text`：用户原始输入，**优先用这个判断语义**；为 null 时退回到 `text`

---

## 处理步骤

> 下面步骤里的 `<模板目录>` = 调用方在 prompt 里给出的「模板数据目录」绝对路径。

1. **读 `<模板目录>/package_map.json`**：对每个 `package_name`，查到对应**专属 product**（"XFolder" / "MP3 Cutter" / "Video to MP3" / null）。
   - **「公共」产品对所有 app 生效**：无论专属 product 是什么（**含 `null`**），匹配候选都要并入「公共」产品的模板（跨 app 的通用回复，如求好评、感谢反馈、通用排查引导）。
   - 因此 `product === null`（ringwall / xplayer）**不再一律 `unmatched`**——它们没有专属模板，但仍可命中「公共」模板。
2. **读 `<模板目录>/index.json`**（**不是全量 templates.json**）：紧凑匹配索引，每产品含每条模板的 `id` + `category`（中文类别名即"关键词/主题"），无全文。取两组候选：**「公共」product 的模板**（所有 app 都用）和**该 app 专属 product 的模板**（若有）。按「**先公共、后专属**」匹配（见步骤 4）。匹配只用 index，上下文很小。
3. **读 `references/matching_rules.md`**：理解匹配口径。
4. **匹配阶段——对每条评论判定命中哪条模板（先公共、后专属）**：
   - 评论已带中文译文（输入的 `text` 字段，因 tester-app 用 `translationLanguage: "zh-CN"`）。用 `text`（中文）配合 `original_text` 做语义匹配。
   - **第一步，先匹配「公共」**：评论是否高置信命中「公共」产品里某 `category`（confidence ≥ 0.9）→ 命中就用该公共模板（`common-*`），**到此为止，不再看专属**。
   - **第二步，公共没对症，再匹配专属**：评论高置信命中该 app 专属产品某 `category` → 用专属模板（如 `xfolder-*` / `mp3cutter-*`）。
   - 两边都没高置信对症 → `unmatched`。
   - **只认高置信（≥ 0.9），不要勉强**：模糊、宽泛、多主题、纯好评但无对应类别 → `unmatched`。宁可漏判也不要错配。
5. **取全文 + 对齐回复语言——只对"命中"的评论**：
   - 把所有命中的 `template_id` 收集起来，用**一条 Bash 命令**从 `<模板目录>/templates.json` 按 id 取出这些模板的 **源语言 `lang` + 正文 `text` + 预存译文 `translations`**（这样全量模板全文**不进入上下文**）。模板是中/英双源（`lang`=`en` 或 `zh-CN`，缺省 `en`），且**大多已预翻译好各语言**存在 `translations`（语言码→译文，app 原生码如 `ru`/`zh-rCN`）。输出每个 id 的 JSON：
     ```
     python -c "import json,sys; d=json.load(open(r'<模板目录>/templates.json',encoding='utf-8')); m={t['id']:{'lang':t.get('lang','en'),'text':t['text'],'translations':t.get('translations',{})} for p in d['products'].values() for t in p['templates']}; print(json.dumps({i:m.get(i) for i in sys.argv[1:]},ensure_ascii=False))" <id1> <id2> ...
     ```
   - 确定该评论回复语言 `lang`：`target_language` 是具体 ISO 码 → `lang=target_language`；`== "auto"` → 取 `reviewer_language`，空/不可信则据 `original_text`(无则 `text`) 判定，判不了退回 `"en"` 并记 `warnings`。
   - **【优先吃预存译文，命中就不要再翻译】** 把回复语言 `lang` 归一成模板语言码 `mlang`：`zh-CN`/`zh-Hans`→`zh-rCN`；`zh-TW`/`zh-Hant`→`zh-rTW`；`id`→`in`；其余取主子标签（`pt-BR`→`pt` 等）。然后：
     1. `mlang` 归一后 == 模板源语言（源 `en` 回 `en`，或源 `zh-CN` 回 `zh-rCN`）→ **直接用模板原文 `text`**。
     2. 否则 `translations[mlang]` 存在且非空 → **直接用这条预存译文，不要自己翻译**。
     3. 都没有（漏译/新语言）→ **回退实时翻译**：把 `text` 从源语言忠实翻到 `lang`（不改语义、不增删，**邮箱/版本号/OEM 操作步骤/产品名/emoji 一字不改**），并在 `warnings` 记一笔 `review_id xxx: 模板 yyy 缺 mlang 译文，已实时翻译`（便于回头到 app 补全）。
   - 再生成一份**中文预览** `text_zh`（回复语言是 `zh-CN`/`zh-rCN` 时可与 `text` 一致或留空）。预存译文里若已有 `zh-rCN`，可直接用作 `text_zh`。
   - 该命中评论输出**恰好 1 条候选**（`source: "template"`，含 `template_id` / `category` / `confidence` / `language` / `text` / `text_zh`）。
6. **写出候选文件**：与输入 JSON 同目录，**把输入文件名的 `.json` 后缀替换成 `.candidates.json`**。
   例：输入 `pending-reviews-1733600000.json` → 输出 `pending-reviews-1733600000.candidates.json`（**不是** `....json.candidates.json`）。

7. **【强制】写完后自检 JSON 合法性**（调用方用 `serde_json` 严格解析，非法直接报错丢弃整批）：
   - 用 Bash 跑一次校验（把 `<out>` 换成实际路径）：
     ```
     python -c "import json,sys; json.load(open(sys.argv[1],encoding='utf-8')); print('JSON OK')" "<out>"
     ```
   - **报错就必须修好再重写**，直到打印 `JSON OK` 为止。最常见的坑见下。

### 写 JSON 的硬性纪律（务必遵守，否则整批作废）

- **`text` / `text_zh` 里出现的双引号必须转义成 `\"`**。这是最高频的失败点——翻译里经常出现 `Change the "Photos access" option` 或中文 `点"删除更新"` 这种带引号的内容。
- **更稳妥的做法：回复正文里尽量不用 ASCII 直引号 `"`**。要引用 UI 选项名时，中文用 `「」` 或 `『』`，英文用单引号 `'...'` 或直接去掉引号。这样从源头避免转义错误。
- 正文里的换行写成 `\n`，不要写裸换行。
- 不要在 JSON 里加注释、尾逗号。
- 写完**一定**执行上面的 python 校验；只有 `JSON OK` 才算完成这条命令。

---

## 输出 JSON schema

```json
{
  "input": "<原 JSON 文件名>",
  "target_language": "en",
  "results": [
    {
      "review_id": "abc123",
      "package_name": "files.fileexplorer.filemanager",
      "product": "XFolder",
      "matched": true,
      "candidates": [
        {
          "source": "template",
          "template_id": "xfolder-010",
          "category": "广告太多",
          "confidence": 0.92,
          "language": "en",
          "text": "Thank you for your feedback! We understand...",
          "text_zh": "感谢您的反馈！我们理解广告可能会影响您的体验……"
        }
      ]
    },
    {
      "review_id": "def456",
      "package_name": "files.fileexplorer.filemanager",
      "product": "XFolder",
      "matched": false,
      "candidates": []
    }
  ],
  "warnings": []
}
```

字段定义：
- `matched`: bool。`true` = 命中模板，`candidates` 恰好 1 条（`source: "template"`）。`false` = 未命中/跳过，`candidates` 为空 `[]`，由用户单独处理。
- `source`: 恒为 `"template"`（本流程不生成原创）。
- `template_id` / `category` / `confidence`: 命中的模板 id、类别、匹配置信度（≥0.9）。
- `language`: 该候选实际使用的语言 ISO 代码。`target_language == "auto"` 时**逐条评论可能不同**（各自跟随评论语言）。
- `text`: 模板对齐到 `language` 的正文（`en` 直接用原文，否则忠实翻译）。
- `text_zh`: 中文预览，给用户看用，不会发出；`language == "zh-CN"` 时可与 `text` 一致或留空。
- **每条 review 都要在 `results` 里有一项**（命中或未命中都要），不要漏。

---

## 调用纪律

1. **不发任何 chat 文本**：这是非交互调用，最终输出只是写出 candidates.json 文件，然后回报一行 "Written: <path>"。不要在 chat 里复述结果。
2. **对每条 review 都必须在 `results` 里有一项**：命中→`matched:true`+1 条；拿不准/未命中→`matched:false`+`candidates:[]`。**绝不为了凑数而臆造模板或现编回复**。
3. **绝不现生成回复**：本流程只"匹配 + 翻译命中模板"。没有合适模板就跳过，交给用户（app 内单条 AI 回复另有现生成入口）。
4. **不要修改原 JSON**，只读不写。

---

## 通用纪律

### 0. 字符长度限制（按 channel 区分）

| channel | 限制 | 默认 |
|---------|------|------|
| `gp`（Google Play 回复） | **每条候选 ≤ 350 字符**（含空格、标点、emoji） | ✅ 默认走这条 |
| `email` | 无限制 | 仅当输入 JSON 显式传 `channel: "email"` 时启用 |

从输入 JSON 顶层读 `channel`，缺省视为 `"gp"`。

**在 `gp` 模式下对模板候选的处理**：
- 模板原文（英文）已超 350 → **直接跳过**该评论标 `unmatched`（很可能是邮件用模板）。
- 模板原文 ≤ 350 但翻译到 target_language 后预估超 350 → 翻译时**适度精简**保留主旨；若实在压不到 350 以下也跳过标 `unmatched`，并在 `warnings` 记一笔 `review_id xxx: 命中模板翻译后超长，跳过`。
- 不对模板做"砍后半句"式硬切，要么完整翻译，要么跳过。

**在 `email` 模式下**：模板原文照搬，翻译可自由展开，不卡 350。

### 1. 不编造

模板里出现版本号（"version 1.4.8"）、邮箱（"filemanager.feedback@gmail.com"）等具体值时：
- 邮箱**原样保留**，不要换。
- 版本号：若用户评论或 `app_version_name` 字段能确证版本相关，可微调；不能确证就保留模板原值。
- 不知道的事实（团队名、CEO 名、价格、未发布功能）**绝不编造**。

### 2. 翻译纪律

把模板（源语言 `en` 或 `zh-CN`，见模板 `lang` 字段）翻译到 target_language 时：
- 保持语义和语气（道歉/感谢/求评分等）。
- 表情符号 / emoji 全部原样保留。
- 邮箱、版本号、专有名词（"XFolder"、"Android"、"Google Play"）**不翻译**。
- 不要因为目标语言习惯不同就改变模板的称谓策略（"Hi friend" / "Dear user" / "Hello there" 等保留风格）。

### 3. 错误回报

无法处理（如模板文件丢失、JSON 损坏）时：写 candidates.json 时把 `results` 留空，`warnings` 里写明原因。

---

## 数据文件清单

**运行时数据**已迁到 tester-app 管理的 `<模板目录>`（即 `~/.tester-app/templates/`），由 app 的「模板管理」页增删改、写 templates.json 时**自动重建** index.json。skill 只读，不写：

| 文件（在 `<模板目录>` 下） | 用途 |
|------|------|
| `index.json` | **匹配阶段读这个**（全部模板 id+category，无全文） |
| `templates.json` | 全量模板全文（命中后按 id 取全文翻译） |
| `package_map.json` | packageName → 产品 |

仓库内 `data/` 与 `references/`：
| 文件 | 用途 |
|------|------|
| `data/source/*.xlsx`、`data/*.json`、`scripts/build_templates.py` | **历史/初始种子**：app 首次启动时从 skill 同步下来的 `data/*.json` 拷一份做种子，之后以 `<模板目录>` 为准，不再是运行时来源；日常编辑全在 app 里。 |
| `references/matching_rules.md` | Claude 的匹配启发（仍随 skill 走），调匹配效果时改。 |

---

## 触发示例

由 tester-app 调用，输入参数就是一个 JSON 文件路径：
```
/review-reply C:\Users\chenj\.tester-app\pending-reviews-1733600000.json
```
（顶层有 `groups` 字段的 `.json` 文件 → 走批量匹配流程）
