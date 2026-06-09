# review-reply

为 Google Play 评论生成回复的 Claude Code skill，给 tester-app 批量调用 + 在 Claude Code 对话里手动调用两种用法。

## 怎么改模板

1. 改 `data/source/XFolder用户回复模板.xlsx`（原始 xlsx，编辑源）
2. 运行：
   ```bash
   python scripts/build_templates.py
   ```
3. 检查输出 `data/templates.json`，把改动连同源文件一起提交。

## 怎么在本地直接当 skill 用

把这个目录软链/拷贝到 `~/.claude/skills/review-reply/`：

```powershell
# Windows
New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\review-reply" -Target "C:\Users\chenj\Documents\trae_projects\review-reply"
```

然后在 Claude Code 会话里：
```
/review-reply
```

## 怎么加新的 app

1. 在 `data/source/*.xlsx` 里加一个新 sheet（结构同现有 sheet：A=类别，B=英文，C/D=别的语言名+译文…）
2. 在 `scripts/build_templates.py` 的 `SHEET_TO_PRODUCT` 和 `PRODUCT_SLUGS` 里加映射
3. 跑 `build_templates.py`
4. 在 `data/package_map.json` 里加 `packageName → product` 映射

## 怎么改 ringwall / xplayer 从"纯生成"变成"有模板"

1. 在 xlsx 里加对应 sheet（命名约定见 `scripts/build_templates.py:SHEET_TO_PRODUCT`）
2. 在 `data/package_map.json` 把 `"product": null` 改成新 sheet 名
3. 重跑 build

## 当前数据量

| 产品 | 模板数 |
|------|--------|
| XFolder | 184 |
| MP3 Cutter | 49 |
| Video to MP3 | 53 |
| ringwall | 0（暂无 sheet）|
| xplayer | 0（暂无 sheet）|

模板**只存英文**；其他目标语言由 skill 在运行时翻译。源 xlsx 里的非英文译文列被 build 脚本忽略。

## 后续接入 tester-app

设计方案见会话；要点：
1. 在 [tester-app `BatchReplyPage.vue:250`](../tester/tester-app/src/pages/BatchReplyPage.vue) 替换 TODO，写一个 Tauri command 调本 skill 并传 JSON 路径
2. 在 `skill_sync.rs` 加 `SkillSource` 一行 + 发 GitHub release
3. UI 加候选下拉
