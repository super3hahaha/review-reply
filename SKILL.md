---
name: review-reply
description: >
  为 Google Play 应用评论生成回复。支持两种调用：
  (A) 批量 —— 输入是一个 JSON 文件路径（含 target_language + groups[].reviews[]），
      输出同名 *.candidates.json。命中模板→返回 1 条（翻译好的模板）；未命中→matched:false，留给用户单独处理。
  (B) 单条 —— 用户在对话里粘贴一条评论 + 指定 app 和目标语言，skill 列出候选让用户选。
  触发词：「评论回复」「批量回复评论」「Play 评论回复」「review reply」，或显式
  「/review-reply」斜杠命令；用户提到 XFolder / MP3 Cutter / Video to MP3 / ringwall / xplayer
  且涉及"回复用户"也应触发。
---

# Review Reply Skill

为五个 app 的 Google Play 用户评论生成回复。**以"匹配模板"为主**：

- **命中模板（高置信 ≥0.9）** → 直接用该模板，**只翻两份**（目标语言正文 + 中文预览），**不生成、不凑多条**。
- **没命中** → **跳过**，标记 `unmatched`，**不生成**。这些评论由用户在 app 里单独处理。

> 为控成本，路径 A 的匹配阶段只读紧凑索引 `data/index.json`（全部模板的 id+category，无全文），命中后才按 id 从 `data/templates.json` 取该模板全文翻译。避免把全量模板全文塞进上下文、也不再给每条评论生成多条候选（早期那样做实测 6 条评论 ~$1 / ~9 分钟）。

---

## 第零步：识别调用路径

进入 skill 后，第一件事是判断本次是**路径 A（批量）**还是**路径 B（单条对话）**：

| 信号 | 路径 |
|------|------|
| 调用消息里包含一个以 `.json` 结尾的文件路径，且该 JSON 顶层有 `groups` 字段 | **A：批量** |
| 调用消息是自然语言粘贴的单条评论 / 用户在对话里描述评论 | **B：单条** |

无法判断时，**直接询问用户**走哪条路径，不要瞎猜。

---

## 路径 A：批量回复（被 tester-app 调用）

### A.1 输入

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

### A.2 处理步骤

1. **读 `data/package_map.json`**：对每个 `package_name`，查到对应 `product`（"XFolder" / "MP3 Cutter" / "Video to MP3" / null）。
   - `product === null`（ringwall / xplayer）→ 无模板可匹配 → 该 group 所有评论一律 `unmatched`（跳过，不生成）。
2. **读 `data/index.json`**（**不是全量 templates.json**）：这是紧凑匹配索引，每产品只含每条模板的 `id` + `category`（中文类别名即"关键词/主题"），**没有模板全文**。匹配只用它，上下文很小。
3. **读 `references/matching_rules.md`**：理解匹配口径。
4. **匹配阶段——对每条评论判定命中哪条模板**：
   - 评论已带中文译文（输入的 `text` 字段，因 tester-app 用 `translationLanguage: "zh-CN"`）。用 `text`（中文）配合 `original_text` 做语义匹配，看它命中该产品 index 里哪个 `category`。
   - **只认高置信命中（confidence ≥ 0.9）**：评论主题与某 category 明确对症才算命中，记下该 `template_id`。否则该评论 = `unmatched`。
   - **不要勉强**：模糊、宽泛、多主题、纯好评但无对应类别 → 一律 `unmatched`，交给用户单独处理。宁可漏判也不要错配。
5. **取全文 + 翻译——只对"命中"的评论**：
   - 把所有命中的 `template_id` 收集起来，用**一条 Bash 命令**从 `data/templates.json` 按 id 取出这些模板的英文全文（这样全量模板全文**不进入上下文**）。例如：
     ```
     python -c "import json,sys; d=json.load(open(r'<skill>/data/templates.json',encoding='utf-8')); m={t['id']:t['text'] for p in d['products'].values() for t in p['templates']}; [print(i+'\t'+m.get(i,'')) for i in sys.argv[1:]]" <id1> <id2> ...
     ```
   - 确定该评论回复语言 `lang`：`target_language` 是具体 ISO 码 → `lang=target_language`；`== "auto"` → 取 `reviewer_language`，空/不可信则据 `original_text`(无则 `text`) 判定，判不了退回 `"en"` 并记 `warnings`。
   - 把命中模板的英文全文：`lang=="en"` 直接用原文；否则**忠实翻译**到 `lang`（不改语义、不增删，**邮箱/版本号/OEM 操作步骤/产品名/emoji 一字不改**）。
   - 再生成一份**中文预览** `text_zh`（`lang=="zh-CN"` 时可与 `text` 一致或留空）。
   - 该命中评论输出**恰好 1 条候选**（`source: "template"`，含 `template_id` / `category` / `confidence` / `language` / `text` / `text_zh`）。
6. **写出候选文件**：与输入 JSON 同目录，**把输入文件名的 `.json` 后缀替换成 `.candidates.json`**。
   例：输入 `pending-reviews-1733600000.json` → 输出 `pending-reviews-1733600000.candidates.json`（**不是** `....json.candidates.json`）。

6. **【强制】写完后自检 JSON 合法性**（调用方用 `serde_json` 严格解析，非法直接报错丢弃整批）：
   - 用 Bash 跑一次校验（把 `<out>` 换成实际路径）：
     ```
     python -c "import json,sys; json.load(open(sys.argv[1],encoding='utf-8')); print('JSON OK')" "<out>"
     ```
   - **报错就必须修好再重写**，直到打印 `JSON OK` 为止。最常见的坑见下。

#### 写 JSON 的硬性纪律（务必遵守，否则整批作废）

- **`text` / `text_zh` 里出现的双引号必须转义成 `\"`**。这是最高频的失败点——翻译里经常出现 `Change the "Photos access" option` 或中文 `点"删除更新"` 这种带引号的内容。
- **更稳妥的做法：回复正文里尽量不用 ASCII 直引号 `"`**。要引用 UI 选项名时，中文用 `「」` 或 `『』`，英文用单引号 `'...'` 或直接去掉引号。这样从源头避免转义错误。
- 正文里的换行写成 `\n`，不要写裸换行。
- 不要在 JSON 里加注释、尾逗号。
- 写完**一定**执行上面的 python 校验；只有 `JSON OK` 才算完成这条命令。

### A.3 输出 JSON schema

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
- `source`: 恒为 `"template"`（本流程不再生成原创）。
- `template_id` / `category` / `confidence`: 命中的模板 id、类别、匹配置信度（≥0.9）。
- `language`: 该候选实际使用的语言 ISO 代码。`target_language == "auto"` 时**逐条评论可能不同**（各自跟随评论语言）。
- `text`: 模板对齐到 `language` 的正文（`en` 直接用原文，否则忠实翻译）。
- `text_zh`: 中文预览，给用户看用，不会发出；`language == "zh-CN"` 时可与 `text` 一致或留空。
- **每条 review 都要在 `results` 里有一项**（命中或未命中都要），不要漏。

### A.4 路径 A 的纪律

1. **不发任何 chat 文本**：路径 A 是非交互调用，最终输出只是写出 candidates.json 文件，然后回报一行 "Written: <path>"。不要在 chat 里复述结果。
2. **对每条 review 都必须在 `results` 里有一项**：命中→`matched:true`+1 条；拿不准/未命中→`matched:false`+`candidates:[]`。**绝不为了凑数而臆造模板或现编回复**。
3. **绝不现生成回复**：本流程只"匹配 + 翻译命中模板"。没有合适模板就跳过，交给用户。
4. **不要修改原 JSON**，只读不写。

---

## 路径 B：单条对话回复

### B.1 用户没说全时先问

需要 2 个最少信息：
1. **app**（产品）：XFolder / MP3 Cutter / Video to MP3 / ringwall / xplayer
2. **评论内容**：文本，最好附带星级

缺什么就**问什么**，一次问完，不要一项一项追问。

**target_language 的默认规则**：用户没显式指定时，**默认用评论本身的语言**回复（es 评论 → 回 es；fa 评论 → 回 fa；zh-CN 评论 → 回 zh-CN）。不要追问。仅当评论本身语言难以判定（混合多语 / 全是 emoji）时才问一次。用户显式写 `+en` / `+<lang>` 等参数则按其指定。

### B.2 流程

1. 按用户给的 app 查 `data/templates.json` 拿该产品的模板池（ringwall / xplayer 直接跳到第 4 步）。
2. 按 `references/matching_rules.md` 匹配满足长度限制、confidence ≥ 0.3 的 Top-N 模板（N ≤ 3）。
3. **若 Top1 模板 confidence ≥ 0.9** → 跳过原创，直接展示这 1-3 条模板；否则用**不同方向**的原创补齐到 4 条总候选（参见路径 A.2 的方向枚举）。
4. 把所有 4 条都对齐到 target_language 并**翻译一份中文预览**。
5. **在 chat 里展示 4 条候选**，每条都同时展示"目标语言版"和"中文预览"，格式如下：

```
匹配结果（app: XFolder，目标语言: en，渠道: gp，352/350 检查通过）：

【候选 1 · 模板「广告太多」· 置信 0.92 · 198 字符】
EN:  Thank you for your feedback! We understand that ads may affect...
中文: 感谢您的反馈！我们理解广告可能会影响您的体验……

【候选 2 · 模板「广告无法关闭」· 置信 0.68 · 220 字符】
EN:  Hello there! We apologize for the inconvenience caused...
中文: 您好！很抱歉这条广告给您造成了不便……

【候选 3 · 原创 · 方向: 道歉式 · 240 字符】
EN:  We're really sorry about the ad you ran into...
中文: 很抱歉您遇到这条广告……

【候选 4 · 原创 · 方向: 给反馈通道 · 215 字符】
EN:  Could you send the ad screenshot to filemanager.feedback@gmail.com...
中文: 能否把这条广告的截图发到 filemanager.feedback@gmail.com……

请选一条（回 1/2/3/4），或要求改写、换个角度。
```

6. 等用户选 → 输出最终文案（**只输出目标语言版的文案本体，不要中文预览，不要前后多余话**），可直接复制粘贴到 Play Console。
7. 若用户对所有候选都不满意 → 让用户描述想要的方向，重新生成 1-2 条原创版本。

### B.3 ringwall / xplayer 特殊处理

这俩 app 没有模板 sheet。直接跳过匹配，**生成 4 条不同方向的原创候选**（道歉 / 解释 / 操作指引 / 邀请反馈），每条都附中文预览，让用户选。

---

## 通用纪律（两路径都适用）

### 0. 字符长度限制（按 channel 区分）

| channel | 限制 | 默认 |
|---------|------|------|
| `gp`（Google Play 回复） | **每条候选 ≤ 350 字符**（含空格、标点、emoji） | ✅ 默认走这条 |
| `email` | 无限制 | 仅当调用方显式传 `channel: "email"` 时启用 |

**路径 A**：从输入 JSON 顶层读 `channel`，缺省视为 `"gp"`。
**路径 B**：默认按 `"gp"` 处理；用户提到"邮件回复"/"发邮件"/"email reply"等明显信号时切到 `"email"` 模式。

**在 `gp` 模式下的处理**：

1. **原创回复（`source: "generated"`）**：写完后**自己数一遍字符数**（`len(text)` 在 Python 里就是字符数；emoji 占 1-2 字符按实际算）。超 350 必须重写更短的版本，**不可输出**。
2. **模板候选（`source: "template"`）**：
   - 模板原文（英文）已超 350 → **直接跳过**，不纳入候选池（很可能是邮件用模板）
   - 模板原文 ≤ 350 但翻译到 target_language 后预估超 350 → 翻译时**适度精简**保留主旨；若实在压不到 350 以下也跳过。
   - 不对模板做"砍后半句"式硬切，要么完整翻译，要么换一条。
3. **如果所有匹配模板都超长或不达 0.3 置信**：candidates 仍要凑足 4 条，全部用**不同方向**的原创。在 `warnings` 里写一笔 `review_id xxx: 无合格模板（超长/低置信），全部使用原创补齐`。

**在 `email` 模式下**：模板原文照搬，翻译可自由展开，原创回复也不卡 350。

### 1. 不编造

模板里出现版本号（"version 1.4.8"）、邮箱（"filemanager.feedback@gmail.com"）等具体值时：
- 邮箱**原样保留**，不要换。
- 版本号：若用户评论或 `app_version_name` 字段能确证版本相关，可微调；不能确证就保留模板原值。
- 不知道的事实（团队名、CEO 名、价格、未发布功能）**绝不编造**。

### 2. 翻译纪律

把英文模板翻译到 target_language 时：
- 保持语义和语气（道歉/感谢/求评分等）。
- 表情符号 / emoji 全部原样保留。
- 邮箱、版本号、专有名词（"XFolder"、"Android"、"Google Play"）**不翻译**。
- 不要因为目标语言习惯不同就改变模板的称谓策略（"Hi friend" / "Dear user" / "Hello there" 等保留风格）。

### 3. 类别去重

挑 Top3 时类别尽量分散，不要 3 条都是同一大区（如全是广告系）。除非这条评论真的就是纯粹广告投诉 —— 此时优先 2 条不同的广告类模板 + 1 条相邻类别。

### 4. 错误回报

无法处理（如模板文件丢失、JSON 损坏）时：
- 路径 A：写 candidates.json 时把 `results` 留空，`warnings` 里写明原因。
- 路径 B：直接在 chat 里说清楚错在哪。

---

## 数据文件清单

| 文件 | 用途 | 修改流程 |
|------|------|---------|
| `data/source/*.xlsx` | 模板编辑源（全量） | 直接改 xlsx |
| `data/index.json` | **路径 A 匹配阶段读这个**（全部模板 id+category，无全文，build 产物） | 改 xlsx 后跑 `python scripts/build_templates.py` |
| `data/templates.json` | 全量模板全文（命中后按 id 取全文翻译；路径 B 也读它） | 改 xlsx 后跑 `python scripts/build_templates.py` |
| `data/package_map.json` | packageName → 产品 | 新 app 上线时手动加 |
| `references/matching_rules.md` | Claude 的匹配启发 | 调匹配效果时改 |

> `index.json` 是 `templates.json` 的"瘦身视图"（只留 id+category），由 build 自动同步产出。匹配口径在 `references/matching_rules.md` 调。

---

## 触发示例

**路径 A（tester-app 调用）**：
```
/review-reply C:\Users\chenj\.tester-app\pending-reviews-1733600000.json
```
（输入参数就是一个 JSON 文件路径，skill 知道走批量）

**路径 B（用户对话）**：
```
/review-reply
评论: "The ads are way too long, especially in offline mode"
app: XFolder
target: en
```
或更随意：
```
帮我回复一条 XFolder 的评论，用英文回：
"ads are too long offline"
```
