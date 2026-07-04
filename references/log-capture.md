# 日志捕获(补上系统的地基)

整套深度培育都建立在"真实调用日志"上。但 Claude Code / Codex **默认不会**把 skill 调用记成结构化日志。旧版让用户"重要会话后手动追加 JSONL"——没人会做,于是所有人都掉进冷启动/回溯的低置信度模式。本文件提供真正可落地的捕获方式,让"真实日志"从空想变可得。

三条路径,按可靠性从高到低:**自动 hook > 会话粘贴回溯 > 冷启动**。

---

## 路径一 · 自动 hook 捕获(推荐,Claude Code)

Claude Code 的 **PostToolUse hook** 会在每次工具调用后触发。skill 通过 Skill 工具被调用,所以可以用一个 hook 自动把每次 skill 调用记进 JSONL。

### 配置(写入 `~/.claude/settings.json` 或项目 `.claude/settings.json`)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Skill",
        "hooks": [
          {
            "type": "command",
            "command": "jq -c '{ts: now, session: .session_id, cwd: .cwd, skill: .tool_input}' >> ~/.claude/skill-gardener/usage-raw.jsonl",
            "async": true
          }
        ]
      }
    ]
  }
}
```

- `matcher: "Skill"` 匹配 Skill 工具;`async: true` 不阻塞主循环(一次会话可能有很多次调用)
- hook 从 stdin 收到事件 JSON(含 `session_id`、`cwd`、`tool_input` 即被调用的 skill 名与参数/prompt),用 jq 抽取关键字段追加到文件

### ⚠️ 两个必须诚实告知用户的限制

1. **matcher 名称需按你的版本确认**。matcher 做精确工具名匹配,内置 Skill 工具的确切名称可能随版本不同。最稳的验证法:先挂一个**无 matcher 的全捕获 hook** 打印 `.tool_name`,调用一次任意 skill,回看确切名字再填进 matcher。较新版本(v2.1.85+)可用更精确的 `if` 字段替代 matcher。

2. **hook 只能可靠捕获"调用"(输入侧),难以捕获 skill 最终产出(输出侧)**。PostToolUse 拿到的是 skill 的启动输入,不是模型随后生成的完整输出。要配出输入→输出对,两种补法:
   - **配一个 Stop hook**:在 assistant 回合结束时追加该回合的最终输出(实现更重,适合愿意折腾的用户)
   - **轻量法**:hook 只负责记"哪个 skill、什么 prompt、何时",输出侧用下面的"会话粘贴回溯"随手补——这对评估已经够用(很多失败模式的检测信号在输入侧和用户后续反馈里就能看到)

### 让 Gardener 消化原始日志

原始 `usage-raw.jsonl` 按 skill 名分流到 `_runtime/logs/<skill-name>/usage.jsonl`。Gardener 首次培育某 skill 时,若发现原始日志,自动做一次分流整理,并告知用户已积累多少条。

---

## 路径二 · 会话粘贴回溯(低摩擦,任何工具通用)

不想配 hook,或用 Codex(hook 机制不同)时的现实主义方案。把"手动记 JSONL"的高摩擦,降为"把最近几次的输出粘给我,我来结构化"。

流程:
1. 用户说"我把最近几次 X skill 的结果贴给你" / 直接粘贴
2. Gardener 引导补齐每条的三要素:**触发的 prompt(哪怕一句话)、skill 的输出、你对这次结果满不满意(满意/改了哪里/哪里不对)**——"满不满意"这条最值钱,是失败模式检测的直接信号
3. Gardener 结构化写入 `_runtime/logs/<skill-name>/retrospective.jsonl`,标 `source: retrospective`

回溯日志的置信度低于自动日志(它是用户挑出来的、有选择偏差),但**显著高于冷启动**,是实践中最现实的起点。凑够 5-10 条就能开始有意义的深度培育。

---

## 路径三 · 冷启动(兜底,最低置信度)

完全没有真实数据时,才用 Claude 凭 skill 的声明意图和典型场景生成测试 prompt。所有结果标 `confidence: cold_start`,不从冷启动轮提炼原则,不据此覆盖主 skill 而不经用户确认。

**重要**:冷启动主要用于跑通流程、暴露最粗的问题,不要对它的结论过度信任。冷启动能做的,`~tune` 静态审查往往做得更好、更便宜——所以**没数据时优先 `~tune`,而不是冷启动 `~cultivate`**。

---

## 日志质量 > 日志数量

- 10 条带真实用户反馈("这次不对,我想要的是…")的日志,价值高于 100 条只有输出、不知道好坏的日志。
- 引导用户在日志里留下"满不满意"信号,是本系统最该做的一件小事。
- 日志覆盖的场景越多样越好;全是同一种 prompt 的日志会让评估过拟合到那一种。

---

## 给用户的一句话建议

**现在就配上路径一的 hook**(五分钟,一劳永逸),让日志被动积累;同时用路径二把手头已有的几次结果补进来。两周后你会有一批真实数据,那时 `~cultivate` 才真正跑得出高置信度的结论。在那之前,用 `~tune` 已经能把 skill 改得明显更好。
