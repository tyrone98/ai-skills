# Conventions
## Trigger format
- 你可以用一句话触发 Agent，例如：Tab GC / Tab Guard。
- 可选参数用 key=value（空格分隔），例如：
```text
Tab GC keep_ratio=0.1 require_next_action=true
```

## Safety rules (global)
- Never destructive by default.
- 任何关闭/删除类动作，必须：
  1. 先给出 dry-run 预览
  2. 明确列出将关闭的目标
  3. 得到用户确认后再执行

## Git commit constraints
- 仅在用户明确要求提交时执行 commit；push 需要单独明确指令。
- 仅提交与当前任务相关的文件；如存在无关改动，必须先说明并请求确认是否包含。
- 提交前必须检查 `git status --short` 和 `git diff`，确认无意外变更。
- 未经明确要求，不得使用 `git commit --amend`、`git rebase`、`git push --force`。
- 提交信息需简洁明确；必要时先给出建议并等待确认。

# Agent: tab-guard
- Type: Guard / Conditional Agent (non-destructive)
- Purpose: 当浏览器 Tab 数量过多时，提醒用户运行 tab-gc，并可在用户确认后委托执行清理。

## When to use
- “启用 Tab Guard”
- “Tab > 20 自动提示 GC”
- “自动 tab GC（提醒型）”
- “tab-count-based cleanup（先提示）”

## Inputs
- threshold（默认 20）：Tab 数量阈值
- mode（默认 prompt）：守卫行为模式
  - prompt: 只提醒 + 给出建议
  - prompt_then_delegate: 用户确认后自动调用 tab-gc 执行

## Workflow
1. 确认阈值 threshold
   - 用户提供则使用；否则默认 20
2. 获取当前 tab 数量
   - 若系统无法提供 tab count，则请求 tab inventory 后计算
3. 若 tab_count > threshold：
   - 输出提醒（含当前数量与阈值）
   - 推荐运行 tab-gc（默认 dry-run）
   - 不自动执行任何 destructive 动作
4. 若用户明确回复 “执行/继续/1”等确认指令：
   - 委托 tab-gc 执行清理（关闭动作仍由 tab-gc 的确认机制保障）
5. 报告结果
   - before_count, after_count（若执行了 GC）
   - 本次建议/执行摘要

## Trigger examples
```text
Tab Guard
Tab Guard threshold=20
Tab Guard threshold=30 mode=prompt_then_delegate
```

## Output (recommended)
- 提醒消息
- 建议下一步：Run Tab GC (dry-run) / Run Tab GC (strict)
- 若用户确认并执行：附 tab-gc 的关闭列表摘要

# Agent: tab-gc
- Type: Executor / Cleanup Agent (destructive after confirmation)
- Purpose: 通过识别陈旧、重复、低价值 tabs，减少 Tab 数量，同时保护正在进行的工作。

## When to use
- “Tab GC”
- “清理当前 tabs”
- “只保留我正在做的”
- “关闭不常用/重复 tabs”
- “tab cleanup”

## Core rules (defaults)
- keep_ratio=0.3（最多保留 30%）
- require_next_action=true（保留项必须给出下一步行动）
- cutoff_days=7（默认“不常用”的时间阈值：7 天；若可获得 last accessed）

## Protection (always keep)
- pinned tabs
- tabs with unsaved input (if detectable)
- actively playing media (if detectable)

## Modes
- mode=dry-run（默认）：只输出 keep/bookmark/close 列表，不执行关闭
- mode=execute：在用户确认后执行关闭
- profile=strict：更激进策略（常用：keep_ratio=0.1）
- profile=safe：更保守策略（常用：keep_ratio=0.5）

## Workflow
1. Tab inventory
   - 收集：title, url, window, pinned, last_accessed(若可用)
2. 确认“不常用”阈值
   - 用户提供 cutoff 优先；否则用 cutoff_days
3. 保护规则应用
4. 候选项识别（按优先级）
   - 过期（超过 cutoff）
   - 重复/近重复（同域/同路径/同标题强相似）
   - 低价值（搜索结果页、错误页、登录落地页、空白页）
5. 形成决策清单（dry-run 输出）
6. 若用户确认执行：
   - 分批关闭（避免误伤）
7. 报告
   - before/after tab count
   - closed tabs list（title + url）
   - 关键保留项 next actions

## Trigger examples
```text
Tab GC
Tab GC mode=dry-run
Tab GC mode=execute（仅在你已确认关闭时使用）
Tab GC profile=strict keep_ratio=0.1
Tab GC cutoff_days=3 keep_ratio=0.2 require_next_action=true
```

## Output schema
- 必须输出四段（顺序固定）：
  1. keep
     - 每个保留 tab：title, url, reason, task, next_action
  2. bookmark_and_close
     - title, url, reason, suggested_folder
  3. close
     - title, url, reason
  4. top_next_actions
     - 汇总 3–7 条最关键下一步（可执行、短句）

## Confirmation protocol (for execute)
- 在关闭前必须出现一句明确确认提示，例如：
```text
回复 1 执行关闭 / 回复 2 只做书签不关闭 / 回复 3 取消
```
- 仅当用户明确选择执行，才允许进入 mode=execute

# Optional Agent: focus-guard (recommended)
- Type: Meta Guard (non-destructive)
- Purpose: 在工作上下文发散时提供“聚焦提示”，可联动 tab-guard 与 tab-gc。

## Minimal behavior
- 读取当前 tabs → 输出：
  - 你正在进行的 1–3 个任务
  - 每个任务的下一步
  - 若 tab_count 超阈值：提示运行 Tab Guard / Tab GC (dry-run)

## Trigger examples
```text
Focus Check
Summarize my current work context
```
