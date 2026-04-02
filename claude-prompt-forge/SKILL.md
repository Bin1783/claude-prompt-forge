---
name: claude-prompt-forge
description: Generate production-grade prompts following patterns extracted from Claude Code source. Use when writing system prompts, tool descriptions, agent instructions, compression prompts, or any structured LLM prompt. Triggers on "write a prompt", "generate prompt", "design prompt", "create instructions for".
---

# Claude Prompt Forge

Generate production-grade prompts by applying design rules extracted from Claude Code's internal prompt architecture.

## When to Use

- Writing system prompts for agents or tools
- Designing compression/summarization prompts
- Creating tool descriptions with usage instructions
- Writing agent delegation instructions
- Any scenario where you need a structured, high-quality prompt

## Design Rules (extracted from Claude Code source)

The following rules are distilled from analyzing 40+ prompt files in Claude Code's codebase (`services/compact/prompt.ts`, `constants/prompts.ts`, `tools/*/prompt.ts`, `services/extractMemories/prompts.ts`, etc.).

### Rule 1: Layered Structure — Static Shell + Dynamic Injection

Never hardcode dynamic content in prompts. Separate into:
- **Static layer**: cacheable, shared across sessions (role definition, rules, format)
- **Dynamic layer**: injected at runtime (user config, environment state, tool names)

Pattern from Claude Code:
```
const BASE_PROMPT = `Your task is to...`  // static, cacheable

export function getPrompt(customInstructions?: string): string {
  let prompt = BASE_PROMPT
  if (customInstructions) {
    prompt += `\n\nAdditional Instructions:\n${customInstructions}`
  }
  return prompt
}
```

**Apply this rule**: When building prompts, clearly mark which parts are fixed vs. which accept runtime parameters. Use `${variable}` placeholders for dynamic content.

### Rule 2: Behavioral Guardrails — Bookend Pattern

Critical constraints appear BOTH at the start AND end of the prompt. Claude Code calls this the "preamble + trailer" pattern.

Pattern from `compact/prompt.ts`:
```
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Tool calls will be REJECTED and will waste your only turn.`

const NO_TOOLS_TRAILER = `REMINDER: Do NOT call any tools. Respond with plain text only.
Tool calls will be rejected and you will fail the task.`

// Usage: PREAMBLE + body + TRAILER
```

**Apply this rule**: For any hard constraint, state it once at the beginning (so the model reads it first) and once at the end (so it's fresh in attention when generating). Use different wording each time — not copy-paste.

### Rule 3: Structured Output via Drafting Scratchpad

For complex outputs, require a two-phase response:
1. `<analysis>` block — model organizes thoughts (stripped before use)
2. `<summary>` or `<output>` block — clean final output (kept)

Pattern from Claude Code:
```
Before providing your final summary, wrap your analysis in <analysis> tags.
In your analysis process:
1. Chronologically analyze each message...
2. Double-check for technical accuracy...

[then the output format in <summary> tags]
```

Post-processing strips `<analysis>`:
```
formattedSummary = summary.replace(/<analysis>[\s\S]*?<\/analysis>/, '')
```

**Apply this rule**: For any task requiring accuracy (summarization, extraction, classification), add a scratchpad phase. Name the tags to match the task: `<reasoning>`, `<planning>`, `<analysis>`.

### Rule 4: Section-Numbered Output Format with Examples

Define output format as numbered sections with clear labels, then provide a complete `<example>` showing the expected structure.

Pattern from Claude Code (9-section compression):
```
Your summary should include the following sections:

1. Primary Request and Intent: [description of what goes here]
2. Key Technical Concepts: [description]
...
9. Optional Next Step: [description with detailed rules]

Here's an example of how your output should be structured:

<example>
<summary>
1. Primary Request and Intent:
   [Detailed description]
2. Key Technical Concepts:
   - [Concept 1]
   ...
</summary>
</example>
```

**Apply this rule**: Always include at least one complete example. The example should show realistic content, not just `[placeholder]`. Number sections for unambiguous reference.

### Rule 5: Negative Instructions with Consequence Explanation

Don't just say "don't do X" — explain what happens if the model does X.

Pattern from Claude Code:
```
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
```

```
- NEVER skip hooks (--no-verify) or bypass signing (--no-gpg-sign)
  unless the user has explicitly asked for it.
  If a hook fails, investigate and fix the underlying issue.
```

**Apply this rule**: Every "don't" gets a "because" or "instead". Format: `Don't X — [consequence]. Instead, Y.`

### Rule 6: Hierarchical List Construction

Complex instructions use nested bullet lists: top-level items are categories, sub-items are specific rules.

Pattern from Claude Code's `prependBullets()`:
```
const items = [
  'When issuing multiple commands:',       // category header
  [                                         // sub-items (indented)
    'If independent, make parallel calls.',
    'If dependent, chain with &&.',
    'Use ; only when order matters but failure is OK.',
  ],
  'For git commands:',                      // next category
  [
    'Prefer new commit over amending.',
    'Never skip hooks.',
  ],
]
```

**Apply this rule**: Group related instructions under category headers. Use consistent indentation. Never mix abstraction levels in the same list.

### Rule 7: Tool Description = What + Anti-Patterns + When-Instead

Every tool/capability description follows this structure:
1. One-line summary of what it does
2. When to use it (positive examples)
3. When NOT to use it (anti-patterns with alternatives)
4. Specific instructions with edge cases

Pattern from Claude Code BashTool:
```
Executes a given bash command and returns its output.

IMPORTANT: Avoid using this tool to run find, grep, cat...
Instead, use the appropriate dedicated tool:
 - File search: Use Glob (NOT find or ls)
 - Content search: Use Grep (NOT grep or rg)
 - Read files: Use Read (NOT cat/head/tail)

# Instructions
 - If your command will create new directories...
 - Always quote file paths that contain spaces...
```

**Apply this rule**: For any capability description, include the "NOT this, use THAT instead" section. Models need explicit routing guidance.

### Rule 8: Persona + Scope Boundary for Sub-Agents

When prompting a sub-agent or specialized mode, always:
1. Declare the new role explicitly
2. Define what tools/resources are available
3. Set a turn budget
4. Define scope boundaries (what to include/exclude)

Pattern from Claude Code memory extraction:
```
You are now acting as the memory extraction subagent.
Analyze the most recent ~${N} messages above.

Available tools: Read, Grep, Glob, read-only Bash, and Edit/Write
for paths inside the memory directory only.

You have a limited turn budget. The efficient strategy is:
turn 1 — all reads in parallel; turn 2 — all writes in parallel.

You MUST only use content from the last ~${N} messages.
Do not waste turns attempting to investigate further.
```

**Apply this rule**: Sub-agent prompts must answer: "Who am I?", "What can I touch?", "How many turns do I get?", "What's in scope?"

### Rule 9: Conditional Sections — Feature Flags in Prompts

Prompts should conditionally include/exclude sections based on context, not dump everything always.

Pattern from Claude Code:
```
function getPrompt(): string {
  const items = [
    'Base instruction...',
    ...(isGitRepo ? ['Git-specific instructions...'] : []),
    ...(hasSandbox ? [getSandboxSection()] : []),
  ]
  return items.join('\n')
}
```

**Apply this rule**: Design prompts with optional sections. Mark them with comments like `// include if: X`. Shorter prompts = fewer distractions = better compliance.

### Rule 10: Partial vs Full — Multiple Prompt Variants for the Same Task

Create variants of the same prompt for different contexts rather than one mega-prompt with conditionals everywhere.

Pattern from Claude Code compression:
```
BASE_COMPACT_PROMPT          — compress the entire conversation
PARTIAL_COMPACT_PROMPT       — compress only recent messages (earlier retained)
PARTIAL_COMPACT_UP_TO_PROMPT — compress prefix (newer messages follow)
```

Each variant shares the 9-section structure but differs in scope instructions.

**Apply this rule**: If a prompt has 2+ substantially different modes, split into named variants. Share structure via constants/templates.

## Workflow

When asked to generate a prompt, follow these steps:

### Step 1: Classify the Prompt Type

Determine which category this prompt falls into:
- **System Prompt**: persistent identity/behavior (Rules 1, 2, 6, 9)
- **Tool Description**: capability + usage guide (Rules 5, 6, 7)
- **Agent Instruction**: sub-agent delegation (Rules 2, 4, 8)
- **Extraction/Compression**: structured output from input (Rules 3, 4, 5)
- **Task Prompt**: one-shot task instruction (Rules 2, 4, 5)

### Step 2: Ask Clarifying Questions

Before generating, ask:
1. What model will run this? (affects tag style, context window)
2. Is this a one-shot prompt or a persistent system prompt?
3. What are the hard constraints? (things that MUST or MUST NOT happen)
4. What's the expected output format?
5. Will this prompt have dynamic/runtime content injected?

### Step 3: Draft Using the Template

```
[PREAMBLE — hard constraints, role declaration]

[CONTEXT — what the model needs to know, scope boundaries]

[TASK — what to do, numbered sections if complex output]

[FORMAT — output structure with <example> blocks]

[ANTI-PATTERNS — what not to do, with consequences]

[TRAILER — repeat critical constraints]
```

### Step 4: Self-Review Checklist

Before presenting the final prompt, verify:
- [ ] Hard constraints appear at start AND end (Rule 2)
- [ ] Every "don't" has a "because" or "instead" (Rule 5)
- [ ] Output format has at least one complete example (Rule 4)
- [ ] Dynamic content uses clear placeholders (Rule 1)
- [ ] No mixed abstraction levels in lists (Rule 6)
- [ ] Sub-agents have role + tools + budget + scope (Rule 8)
- [ ] Prompt doesn't include irrelevant sections (Rule 9)

### Step 5: Present with Rationale

Show the prompt and annotate which rules were applied and why. Offer variants if the use case has multiple modes (Rule 10).
