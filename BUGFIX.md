# BUGFIX（2.0.2 → 2.0.3）

## 标题

「看它刷过什么」恒为 0 条：`parseSeenTable()` 只认 Markdown 表格行，读不到值班轮实际写的 bullet 行（上游 issue #77 / #78）。

## 证据链

- 上游 issue [#77](https://github.com/c1overzz/dsh-bili-scroll/issues/77)、[#78](https://github.com/c1overzz/dsh-bili-scroll/issues/78)（均 open，报于 2.0.1）：`seen.md` 里明明有几十上百行记录，设置页「看它刷过什么」与 `/api/bili-duty/watched` 恒返回 `total: 0`。
- 代码定位：`index.js` 的 `parseSeenTable()` 第二遍扫描里 `if (!line.startsWith('|')) continue`，非表格行全部跳过；而 `prompts/duty.md` 没有把判重表格式写死，值班轮实际按 `- 日期 · BV · 标题 · UP · 状态` 的 bullet 一行一条写入，两边对不上。
- 端到端复现（假 ctx 真跑 `apply()` + 真实 HTTP 路由 `/api/bili-duty/watched`，夹具 seen.md = 1 行表格 + 4 行 bullet，bullet 样本取自 #77/#78 原文）：
  - 修复前：`total = 1`，只有表格行被解析，4 行 bullet 全部丢失，复现脚本 exit 1。
  - 修复后：`total = 5`，5 行全部解析且 picked/skipped 判定正确，复现脚本 exit 0。

## 根因

解析器与写入方（值班轮的 AI）对判重表格式的约定不一致：解析器只支持 `| 日期 | BV | 标题 | UP | 状态 |` 表格行，写入方按自然习惯写 `·` 分隔的 bullet 行，且状态常挂在最后一段的尾括号里（`标题（挑中）`、`标题（跳过：58 秒…）`）。2.0.1 的「watched 解析容错」只补了日期带时间与 BV 列扫描，未覆盖 bullet 格式。

## 改动

仅 `index.js` 的 `parseSeenTable()`（+34 / −9）：

- 把原表格分支的记录组装尾巴（去重 key、skipped/picked/commented/delivered 判定、`out.push`）原样抽成内部 `pushRow(day, bvid, title, up, status)`，表格分支行为不变。
- 非表格行新增 bullet 分支：正则 `^[-*]\s*(日期)(?:时间)?\s*·\s*(BV…)\s*·\s*(.+)$` 匹配后按 `·` 切分剩余段——≥3 段取 `[标题, UP, 状态…]`，2 段取 `[标题, UP]`；没有独立状态段时，若标题以尾括号结尾且括号内容命中状态关键词，则把括号内容拆为 status 并从标题中去掉。解析结果走同一个 `pushRow`，去重与 delivered 判定与表格行完全一致。

## 复验结果

- 复现脚本（端到端走真实路由）：修复前 exit 1（total=1）→ 修复后 exit 0（total=5，逐行 day/bvid/title/up/status/picked 断言全过）。
- `node --check index.js` 通过。
- 上游自带预检 `node selfcheck.mjs`：修复前 exit 0、修复后 exit 0（`git stash` 对照），无回归。上游 package.json 无 test/typecheck/build 脚本，故以 selfcheck + 本复现脚本为验证口径。

## 边界

- 只改解析器，不改 `prompts/duty.md` 的写入约定，也不改表格行格式；新旧两种行混写在同一个 seen.md 里都能读。
- bullet 行的日期必须在行首、BV 必须为第二段（与 #77/#78 报告的实际写法一致）；AI 自起小标题段落（如 `## 2026-10-22 这一轮`）下的 bullet 行只要行首带日期即可解析，不带日期的自由文本行仍读不到——不在本版范围。
- `rollSeen` 的归档仍只搬运表格行，bullet 行不会被滚进 `seen/` 归档——不影响面板读取与判重，不在本版范围。
- 未做真实 B 站账号/网络验证：本修复是纯离线解析逻辑，复现与复验均走本地假 ctx + 夹具文件。
