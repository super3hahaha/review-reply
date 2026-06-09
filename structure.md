# structure.md（给冷启动的 Claude 看）

## 这是什么

一个 Claude Code skill 仓库，目录就是 skill 根（SKILL.md 在顶层）。最终会被装到 `~/.claude/skills/review-reply/`。

## 目录

```
review-reply/
├── SKILL.md                 ← skill 主入口，定义两种调用路径
├── README.md                ← 给人看的说明
├── structure.md             ← 本文件
├── data/
│   ├── source/
│   │   └── *.xlsx           ← 模板编辑源（不参与运行）
│   ├── templates.json       ← 运行时读这个（由 build 脚本从 xlsx 生成）
│   └── package_map.json     ← packageName → 产品名
├── scripts/
│   └── build_templates.py   ← xlsx → templates.json
└── references/
    └── matching_rules.md    ← 给 Claude 看的语义匹配启发
```

## 关键约束

1. **`templates.json` 是 build 产物**，但要提交（runtime 读，不能依赖用户先 build）。改 xlsx 后必须 build。
2. **`package_map.json` 里 `product: null`** 表示该 app 没模板（ringwall / xplayer），skill 必须走纯生成，**严禁套用其他产品的模板**。
3. SKILL.md 的两条路径（A 批量 / B 单条）共用模板数据和匹配逻辑，仅在"输入/输出形态"和"是否在 chat 里说话"上不同。
4. xlsx 的 A 列空 → 继承上一行类别（同类多变体），build 脚本已处理。
5. xlsx 的 B 列（英文）偶有"实际是中文"的脏数据（xfolder-181 等），不修，**忠实保留**。运行时 skill 看到非英文模板应当先翻译再用。
6. **模板只存英文**（`text` 字段）。非英文目标语言由 skill 在线翻译。xlsx C 列起的译文已被 build 忽略。

## 数据流

```
[xlsx] → build_templates.py → [templates.json]
                                        ↓
              [tester-app pending-*.json] → Claude with SKILL.md → [*.candidates.json]
                                        ↑
              [package_map.json, matching_rules.md]
```

## 发布 & 分发（已走热更新）

- **公开 repo**：https://github.com/super3hahaha/review-reply ，已发 release `v0.1.0`。
- 已注册进 tester-app 的 [skill_sync.rs](../tester/tester-app/src-tauri/src/skill_sync.rs) `SKILLS` 列表（owner `super3hahaha` / repo `review-reply`）。
- **分发链路**：app 启动 → `/releases/latest` 比对 tag → 不一致就下 zipball 覆盖到 `~/.claude/skills/review-reply/`。
- **维护者发版流程**：改 `data/source/*.xlsx`（或 `copy_shared_templates.py` 的共享清单）→ `python scripts/build_templates.py` 重新生成 `templates.json` → commit & push → **`gh release create vX.Y.Z`**（不打 tag 用户拉不到更新）。

## 已知遗留

- ringwall / xplayer 暂无模板 sheet，依赖纯生成（用户后续补）
- 翻译稀疏：除英文外语种覆盖率 < 10%，绝大多数非英语回复要在线翻
- **xlsx 用 openpyxl 改过一次**（`copy_shared_templates.py`）：Google Sheets 导出的 `__xludf` 翻译公式缓存值已丢（Excel 里译文列显示 #NAME?），build 只读 A/B 不受影响；原始备份在 `data/源文件备份_复制MP3Cutter类别前.xlsx`（已 gitignore）。

## tester-app 接入现状（已接通）

- `BatchReplyPage.vue` 的「✨ AI 批量生成回复」按钮 → Rust `run_reply_skill`（[reply.rs](../tester/tester-app/src-tauri/src/reply.rs)）
  → 写 `~/.tester-app/reviews/pending-reviews-<ts>.json` → 跑 `claude /review-reply <json>`
  → 读回 `*.candidates.json` → 前端按 `review_id` 回填每条评论的候选。
- **调用恒为路径 A（批量）**，顶层 `channel: "gp"`。回复语言由页面下拉决定，默认 **`"auto"`**（逐条跟随评论语言，单次调用覆盖混合语言批次）。
- UI：每条评论预填最优候选（`candidates[0]`），「更多」可展开其余候选（EN 正文 + 中文预览 + 标签）切换。

## 下次改动若涉及……

| 改什么 | 改哪 |
|--------|------|
| 加/改模板内容 | xlsx → 重跑 build_templates.py |
| 加新 app | xlsx 加 sheet + build 脚本加映射 + package_map.json 加条目 |
| 调匹配口径 | references/matching_rules.md |
| 改输出 JSON schema | SKILL.md 的 A.3 节，同步改 tester-app 的接收端 |
| 改交互流程 | SKILL.md 路径 A/B 章节 |
