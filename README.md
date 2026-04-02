# Claude Prompt Forge

  基于 Claude Code 源码逆向提炼的提示词生成 Skill，用于生成生产级别的高质量提示词。

  ## 这是什么？

  Claude Prompt Forge 是一个 [Claude Code](https://claude.ai/claude-code) 的 Skill（插件），帮助你为 LLM 应用编写高质量、结构化的提示词。它不是基于通用提示词技巧，而是从 Claude Code 实际源码中的 40+ 个提示词文件中，提炼出 **10 条具体的设计规则**。

  ## 10 条设计规则

  | # | 规则 | 来源 |
  |---|------|------|
  | 1 | **静态壳 + 动态注入** — 可缓存的结构与运行时参数分离 | `compact/prompt.ts` getCompactPrompt() |
  | 2 | **首尾夹击** — 关键约束在提示词开头和结尾各出现一次 | `compact/prompt.ts` PREAMBLE + TRAILER |
  | 3 | **草稿 + 正式输出** — `<analysis>` 用于思考，`<output>` 用于干净结果 | `compact/prompt.ts` analysis + summary |
  | 4 | **编号分段 + 完整示例** — 结构化输出格式配合完整示例 | `compact/prompt.ts` 9 段结构 |
  | 5 | **否定指令 + 后果说明** — 每个"不要"都带"因为" | `BashTool/prompt.ts` |
  | 6 | **分层嵌套列表** — 类别标题下嵌套子规则 | `BashTool/prompt.ts` prependBullets() |
  | 7 | **工具描述 = 是什么 + 反模式 + 替代方案** | `BashTool/prompt.ts`、`FileEditTool/prompt.ts` |
  | 8 | **子代理 = 角色 + 工具 + 预算 + 范围** | `extractMemories/prompts.ts` |
  | 9 | **条件段落** — 按上下文按需组装，不堆砌所有内容 | `constants/prompts.ts` feature flags |
  | 10 | **多变体** — 同一任务拆分命名变体，共享结构 | `compact/prompt.ts` BASE/PARTIAL/UP_TO |

  ## 安装

  ### Claude Code CLI / Desktop

  将 `claude-prompt-forge` 文件夹复制到 Claude Code 的 skills 目录：

  ```bash
  # 克隆仓库
  git clone https://github.com/Bin1783/claude-prompt-forge.git

  # 复制到 Claude Code skills 目录
  cp -r claude-prompt-forge/claude-prompt-forge ~/.claude/skills/

  验证安装

  在 Claude Code 中，skill 应该出现在可用列表中。可以通过以下方式调用：

  /claude-prompt-forge

  使用方式

  安装后，通过以下方式触发：

  - 在 Claude Code 中输入 /claude-prompt-forge
  - 或直接描述需求："写一个提示词..."、"生成提示词"、"设计一个指令..."

  Skill 的工作流程：

  1. 分类 — 判断提示词类型（系统提示词、工具描述、代理指令、提取/压缩、任务提示词）
  2. 澄清 — 逐个提问：目标模型、硬约束、输出格式、是否需要动态注入
  3. 生成 — 按模板构建提示词：
  [PREAMBLE — 硬约束、角色声明]
  [CONTEXT — 范围边界]
  [TASK — 复杂任务用编号分段]
  [FORMAT — 输出结构 + 完整示例]
  [ANTI-PATTERNS — 不要做什么 + 后果]
  [TRAILER — 重复关键约束]
  4. 自检 — 对照 10 条规则逐项检查
  5. 呈现 — 标注应用了哪些规则，需要时提供多个变体

  支持的提示词类型

  ┌────────────┬──────────────────────────┬─────────────────┐
  │    类型    │         适用场景         │    主要规则     │
  ├────────────┼──────────────────────────┼─────────────────┤
  │ 系统提示词 │ Agent 身份定义、持久行为 │ 规则 1、2、6、9 │
  ├────────────┼──────────────────────────┼─────────────────┤
  │ 工具描述   │ 能力说明 + 使用指南      │ 规则 5、6、7    │
  ├────────────┼──────────────────────────┼─────────────────┤
  │ 代理指令   │ 子代理任务委托           │ 规则 2、4、8    │
  ├────────────┼──────────────────────────┼─────────────────┤
  │ 提取/压缩  │ 从输入中提取结构化输出   │ 规则 3、4、5    │
  ├────────────┼──────────────────────────┼─────────────────┤
  │ 任务提示词 │ 一次性任务指令           │ 规则 2、4、5    │
  └────────────┴──────────────────────────┴─────────────────┘

  使用示例

  上下文压缩提示词

  为 LLM 对话应用生成自动压缩提示词：
  - 3 个变体：完整压缩 / 部分压缩 / 再次压缩
  - 9 段结构化摘要，配合 <analysis> 草稿机制
  - 支持多次压缩的超长对话

  图片审核提示词

  生产环境内容安全审核：
  - 9 大违规类别，每个类别有明确的子类型
  - 结构化 JSON 输出，含置信度评分
  - 2 个变体：单图审核 / 批量审核
  - 边界图片处理：三级决策（通过 / 拒绝 / 人工复审）

  构建过程

  通过阅读 Claude Code 还原后的源码（services/compact/prompt.ts、constants/prompts.ts、tools/*/prompt.ts、services/extractMemories/prompts.ts 等 40+ 个提示词文件），识别反复出现的结构模式，提炼为可复用的设计规则。



致谢
Linux.Do 社区
