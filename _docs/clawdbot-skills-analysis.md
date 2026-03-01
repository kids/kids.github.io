# Clawdbot Skills 选择和使用的代码逻辑分析

## 1. Skills 加载机制

### 1.1 加载位置和优先级

Skills 从多个位置加载，按以下优先级合并（高优先级覆盖低优先级）：

```typescript
// 文件: src/agents/skills/workspace.ts
// 优先级顺序（从低到高）：
// 1. extra dirs (skills.load.extraDirs)
// 2. bundled skills (clawdbot-bundled)
// 3. managed skills (~/.clawdbot/skills)
// 4. workspace skills (<workspace>/skills) - 最高优先级
// 5. plugin skills (从插件加载)
```

**关键代码位置：**
- `src/agents/skills/workspace.ts` - `loadSkillEntries()` 函数（第95-175行）

### 1.2 Skills 结构

每个 Skill 包含：
- `Skill` 对象（来自 `@mariozechner/pi-coding-agent`）：
  - `name`: skill名称
  - `description`: 描述
  - `filePath`: SKILL.md文件路径
  - `baseDir`: skill目录路径
- `ParsedSkillFrontmatter`: YAML frontmatter解析结果
- `ClawdbotSkillMetadata`: clawdbot特定元数据
- `SkillInvocationPolicy`: 调用策略配置

**关键代码位置：**
- `src/agents/skills/types.ts` - `SkillEntry` 类型定义（第66-71行）

## 2. Skills 过滤机制

### 2.1 过滤函数：`shouldIncludeSkill`

**文件位置：** `src/agents/skills/config.ts` (第90-151行)

过滤条件（按顺序检查）：

1. **enabled状态检查**
   ```typescript
   if (skillConfig?.enabled === false) return false;
   ```

2. **bundled skills白名单**
   ```typescript
   if (!isBundledSkillAllowed(entry, allowBundled)) return false;
   ```

3. **操作系统要求**
   ```typescript
   if (osList.length > 0 && 
       !osList.includes(resolveRuntimePlatform()) &&
       !remotePlatforms.some((platform) => osList.includes(platform))) {
     return false;
   }
   ```

4. **always标志**（如果为true，直接通过）
   ```typescript
   if (entry.clawdbot?.always === true) return true;
   ```

5. **必需的二进制文件** (`requires.bins`)
   ```typescript
   const requiredBins = entry.clawdbot?.requires?.bins ?? [];
   // 检查PATH中是否存在这些二进制文件
   ```

6. **任一必需的二进制文件** (`requires.anyBins`)
   ```typescript
   const requiredAnyBins = entry.clawdbot?.requires?.anyBins ?? [];
   // 至少有一个存在即可
   ```

7. **必需的环境变量** (`requires.env`)
   ```typescript
   const requiredEnv = entry.clawdbot?.requires?.env ?? [];
   // 检查环境变量或config中的env配置
   ```

8. **必需的配置项** (`requires.config`)
   ```typescript
   const requiredConfig = entry.clawdbot?.requires?.config ?? [];
   // 检查config路径是否为truthy
   ```

### 2.2 额外过滤：skillFilter

如果提供了 `skillFilter`，只保留在过滤列表中的skills：

```typescript
// 文件: src/agents/skills/workspace.ts (第44-63行)
if (skillFilter !== undefined) {
  const normalized = skillFilter.map((entry) => String(entry).trim()).filter(Boolean);
  filtered = normalized.length > 0
    ? filtered.filter((entry) => normalized.includes(entry.skill.name))
    : [];
}
```

### 2.3 Model Invocation 过滤

过滤掉 `disableModelInvocation === true` 的skills（这些只能通过命令调用，不会出现在prompt中）：

```typescript
// 文件: src/agents/skills/workspace.ts (第197-199行)
const promptEntries = eligible.filter(
  (entry) => entry.invocation?.disableModelInvocation !== true,
);
```

## 3. Skills Prompt 生成

### 3.1 Prompt 生成流程

**主要函数：**
- `buildWorkspaceSkillsPrompt()` - 生成prompt字符串
- `buildWorkspaceSkillSnapshot()` - 生成包含版本信息的snapshot
- `resolveSkillsPromptForRun()` - 解析用于运行的prompt

**文件位置：** `src/agents/skills/workspace.ts`

### 3.2 Prompt 格式

**XML格式化实现：**
- **不是**使用anthropic提供的包
- 使用 `formatSkillsForPrompt()` 函数，来自 `@mariozechner/pi-coding-agent` 包
- 该函数将skills格式化为XML格式

**XML格式示例：**
```xml
<available_skills>
  <skill>
    <name>skill-name</name>
    <description>skill description</description>
    <location>path/to/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

**关键代码：**
```typescript
// 文件: src/agents/skills/workspace.ts (第5行)
import {
  formatSkillsForPrompt,
  loadSkillsFromDir,
  type Skill,
} from "@mariozechner/pi-coding-agent";

// 文件: src/agents/skills/workspace.ts (第202行)
const prompt = [remoteNote, formatSkillsForPrompt(resolvedSkills)]
  .filter(Boolean)
  .join("\n");
```

## 4. System Prompt 集成

### 4.1 Skills Section 构建

**文件位置：** `src/agents/system-prompt.ts` (第15-33行)

**过滤和选择的Prompt模板：**
```typescript
function buildSkillsSection(params: {
  skillsPrompt?: string;
  isMinimal: boolean;
  readToolName: string;
}) {
  if (params.isMinimal) return [];
  const trimmed = params.skillsPrompt?.trim();
  if (!trimmed) return [];
  return [
    "## Skills (mandatory)",
    "Before replying: scan <available_skills> <description> entries.",
    `- If exactly one skill clearly applies: read its SKILL.md at <location> with \`${params.readToolName}\`, then follow it.`,
    "- If multiple could apply: choose the most specific one, then read/follow it.",
    "- If none clearly apply: do not read any SKILL.md.",
    "Constraints: never read more than one skill up front; only read after selecting.",
    trimmed,  // 这里插入formatSkillsForPrompt生成的XML
    "",
  ];
}
```

**完整的Skills Section Prompt文本：**
```
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.

<available_skills>
  <skill>
    <name>github</name>
    <description>Interact with GitHub using the `gh` CLI...</description>
    <location>skills/github/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

### 4.2 System Prompt 完整结构

**集成到System Prompt的完整模板：**

Skills section 被插入到 system prompt 的 "## Tooling" 部分之后：

```typescript
// 文件: src/agents/system-prompt.ts (第330-372行)
const lines = [
  "You are a personal assistant running inside Clawdbot.",
  "",
  "## Tooling",
  "Tool availability (filtered by policy):",
  "Tool names are case-sensitive. Call tools exactly as listed.",
  // ... tool descriptions ...
  "TOOLS.md does not control tool availability; it is user guidance for how to use external tools.",
  "If a task is more complex or takes longer, spawn a sub-agent. It will do the work for you and ping you when it's done. You can always check up on it.",
  "",
  "## Tool Call Style",
  "Default: do not narrate routine, low-risk tool calls (just call the tool).",
  "Narrate only when it helps: multi-step work, complex/challenging problems, sensitive actions (e.g., deletions), or when the user explicitly asks.",
  "Keep narration brief and value-dense; avoid repeating obvious steps.",
  "Use plain human language for narration unless in a technical context.",
  "",
  "## Clawdbot CLI Quick Reference",
  // ... CLI commands ...
  "",
  ...skillsSection,  // Skills section 插入这里
  ...memorySection,
  // ... 其他sections ...
];
```

**完整的System Prompt结构（包含Skills部分）：**
```
You are a personal assistant running inside Clawdbot.

## Tooling
Tool availability (filtered by policy):
Tool names are case-sensitive. Call tools exactly as listed.
- read: Read file contents
- write: Create or overwrite files
- edit: Make precise edits to files
- apply_patch: Apply multi-file patches
- grep: Search file contents for patterns
- find: Find files by glob pattern
- ls: List directory contents
- exec: Run shell commands (pty available for TTY-required CLIs)
- process: Manage background exec sessions
- web_search: Search the web (Brave API)
- web_fetch: Fetch and extract readable content from a URL
- browser: Control web browser
- canvas: Present/eval/snapshot the Canvas
- nodes: List/describe/notify/camera/screen on paired nodes
- cron: Manage cron jobs and wake events
- message: Send messages and channel actions
- gateway: Restart, apply config, or run updates
- sessions_list: List other sessions
- sessions_history: Fetch history for another session
- sessions_send: Send a message to another session
- sessions_spawn: Spawn a sub-agent session
- session_status: Show a /status-equivalent status card
- image: Analyze an image with the configured image model

TOOLS.md does not control tool availability; it is user guidance for how to use external tools.

## Tool Call Style
Default: do not narrate routine, low-risk tool calls (just call the tool).
...

## Clawdbot CLI Quick Reference
...

## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.

<available_skills>
  <skill>
    <name>github</name>
    <description>Interact with GitHub using the `gh` CLI...</description>
    <location>skills/github/SKILL.md</location>
  </skill>
</available_skills>

## Memory Recall
...

## Workspace
...
```

**关键点：**
- ✅ **Tools列表在Skills之前**：Agent在读取skill之前就已经知道所有可用工具
- ✅ **无需额外查询**：Agent不需要再去查tools列表，因为已经在system prompt中
- ✅ **直接映射**：Skill文档中提到的工具名称可以直接对应到Tools列表中的工具

## 5. Agent 使用 Skills 的流程

### 5.1 选择阶段

Agent 根据 system prompt 的指导：
1. **扫描** `<available_skills>` 中的 `<description>` 条目
2. **判断**：
   - 如果**恰好一个**skill适用 → 读取其SKILL.md
   - 如果**多个**适用 → 选择最specific的一个，然后读取
   - 如果**没有**适用的 → 不读取任何SKILL.md

### 5.2 读取阶段

Agent 使用 `read` 工具读取选中的skill的SKILL.md文件：

```typescript
// System prompt 指导：
// "read its SKILL.md at <location> with `read`, then follow it."
```

### 5.3 执行阶段

Agent 按照SKILL.md中的指导执行相应的操作（通常是调用工具或执行命令）。

**重要：Agent不需要再去查tools列表**

**原因：**
- **Tools列表已经在System Prompt中**：在"## Tooling"部分（Skills section之前）已经列出了所有可用工具及其简短描述
- Agent在读取skill之前就已经知道有哪些工具可用
- Skill文档中提到的工具名称（如`exec`、`read`等）可以直接对应到system prompt中的工具列表

**System Prompt中的Tools部分示例：**
```
## Tooling
Tool availability (filtered by policy):
Tool names are case-sensitive. Call tools exactly as listed.
- read: Read file contents
- write: Create or overwrite files
- edit: Make precise edits to files
- exec: Run shell commands (pty available for TTY-required CLIs)
- process: Manage background exec sessions
- web_search: Search the web (Brave API)
- browser: Control web browser
...
```

**完整流程示例：**
```
1. Agent看到system prompt中的Tools列表（已包含所有可用工具）
2. Agent看到Skills section，扫描available_skills
3. Agent选择github skill，读取SKILL.md
4. SKILL.md说："Use the `gh` CLI..." 和 "run `gh pr checks`"
5. Agent知道：
   - `exec`工具在Tools列表中（已从system prompt知道）
   - 可以直接调用 `exec("gh pr checks 55 --repo owner/repo")`
   - 不需要再去查询tools列表
```

**设计优势：**
- ✅ **一次性加载**：所有工具信息在system prompt中一次性提供
- ✅ **减少查询**：Agent不需要额外的工具查询步骤
- ✅ **上下文一致**：Skill文档和Tools列表在同一上下文中
- ✅ **高效执行**：Agent可以直接从skill指导映射到已知的工具

## 6. Skill Commands（用户命令调用）

### 6.1 命令生成

**文件位置：** `src/agents/skills/workspace.ts` - `buildWorkspaceSkillCommandSpecs()` (第316-418行)

为每个 `userInvocable !== false` 的skill生成命令规范：
- 命令名：从skill名称sanitize而来（小写、下划线替换特殊字符）
- 描述：skill描述（截断到100字符）
- Dispatch：可选的工具分发配置（`command-dispatch: tool`）

### 6.2 命令解析

**文件位置：** `src/auto-reply/skill-commands.ts` - `resolveSkillCommandInvocation()` (第83-107行)

解析用户输入的命令：
- `/skill-name [args]` - 直接调用skill命令
- `/skill skill-name [args]` - 通过skill命令调用

### 6.3 命令执行

**文件位置：** `src/auto-reply/reply/get-reply-inline-actions.ts` (第160-212行)

如果命令有 `dispatch.kind === "tool"`，直接调用对应工具：
```typescript
if (dispatch?.kind === "tool") {
  const tool = tools.find((candidate) => candidate.name === dispatch.toolName);
  const result = await tool.execute(toolCallId, {
    command: rawArgs,
    commandName: skillInvocation.command.name,
    skillName: skillInvocation.command.skillName,
  });
}
```

否则，重写用户消息，让agent处理：
```typescript
const promptParts = [
  `Use the "${skillInvocation.command.skillName}" skill for this request.`,
  skillInvocation.args ? `User input:\n${skillInvocation.args}` : null,
];
const rewrittenBody = promptParts.join("\n\n");
```

## 7. Skill 元数据配置

### 7.1 Frontmatter 结构

每个SKILL.md文件包含YAML frontmatter：

```yaml
---
name: skill-name
description: "skill description"
metadata: {"clawdbot": {
  "emoji": "🎯",
  "requires": {
    "bins": ["required-binary"],
    "env": ["REQUIRED_ENV"],
    "config": ["config.path"]
  },
  "os": ["darwin", "linux"],
  "always": true,
  "primaryEnv": "API_KEY"
}}
user-invocable: true
disable-model-invocation: false
command-dispatch: tool
command-tool: tool-name
---
```

### 7.2 元数据解析

**文件位置：** `src/agents/skills/frontmatter.ts`

- `resolveClawdbotMetadata()` - 解析clawdbot元数据
- `resolveSkillInvocationPolicy()` - 解析调用策略

## 8. 环境变量注入

### 8.1 Skill Env Overrides

**文件位置：** `src/agents/skills/env-overrides.ts`

在agent运行前，根据skill配置注入环境变量：
- 从 `config.skills.entries[skillKey].env` 读取
- 从 `config.skills.entries[skillKey].apiKey` 读取（如果设置了primaryEnv）

**关键代码：**
```typescript
// 文件: src/agents/pi-embedded-runner/run/attempt.ts (第166-174行)
restoreSkillEnv = params.skillsSnapshot
  ? applySkillEnvOverridesFromSnapshot({
      snapshot: params.skillsSnapshot,
      config: params.config,
    })
  : applySkillEnvOverrides({
      skills: skillEntries ?? [],
      config: params.config,
    });
```

## 9. 关键文件总结

| 文件路径 | 功能 |
|---------|------|
| `src/agents/skills/workspace.ts` | Skills加载、过滤、prompt生成 |
| `src/agents/skills/config.ts` | Skills过滤逻辑（shouldIncludeSkill） |
| `src/agents/skills/frontmatter.ts` | Frontmatter解析 |
| `src/agents/skills/types.ts` | 类型定义 |
| `src/agents/system-prompt.ts` | System prompt构建，包含Skills section |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Agent运行时集成skills |
| `src/auto-reply/skill-commands.ts` | Skill命令解析 |
| `src/auto-reply/reply/get-reply-inline-actions.ts` | Skill命令执行 |

## 10. Skill 中是否包含 Tool 调用和参数？

### 10.1 Skill 内容结构

**Skill文件（SKILL.md）本身不直接包含tool调用和参数**，而是：
- 包含**指导性文本**，告诉agent如何使用工具
- 包含**示例命令**和**工作流程**
- 不包含结构化的tool schema或参数定义

**示例：GitHub Skill**
```markdown
---
name: github
description: "Interact with GitHub using the `gh` CLI..."
---
# GitHub Skill

Use the `gh` CLI to interact with GitHub...

## Pull Requests
Check CI status on a PR:
```bash
gh pr checks 55 --repo owner/repo
```
```

这个skill**不包含**：
- ❌ Tool名称定义
- ❌ Tool参数schema
- ❌ 结构化的API调用规范

这个skill**包含**：
- ✅ 自然语言指导
- ✅ 示例命令
- ✅ 使用场景说明

### 10.2 Command Dispatch（命令分发到工具）

虽然skill文件本身不包含tool调用，但可以通过**frontmatter配置**将skill命令直接分发到工具：

**配置方式：**
```yaml
---
name: my-skill
command-dispatch: tool
command-tool: tool-name
command-arg-mode: raw
---
```

**实现逻辑：**
```typescript
// 文件: src/agents/skills/workspace.ts (第366-408行)
const dispatch = (() => {
  const kindRaw = (
    entry.frontmatter?.["command-dispatch"] ??
    entry.frontmatter?.["command_dispatch"] ??
    ""
  ).trim().toLowerCase();
  
  if (kindRaw !== "tool") return undefined;
  
  const toolName = (
    entry.frontmatter?.["command-tool"] ??
    entry.frontmatter?.["command_tool"] ??
    ""
  ).trim();
  
  return { kind: "tool", toolName, argMode: "raw" } as const;
})();
```

**执行流程：**
1. 用户输入：`/skill-name args`
2. 系统检测到 `command-dispatch: tool`
3. 直接调用指定的tool：`tool.execute(toolCallId, { command: args })`
4. **不经过agent处理**

### 10.3 Agent 使用 Skill 的流程

当agent通过prompt选择skill时：
1. **读取SKILL.md**：使用`read`工具读取文件内容
2. **解析指导**：从markdown文本中理解如何使用工具
3. **执行操作**：根据指导调用相应的工具（如`exec`工具运行命令）

**示例流程：**
```
User: "Check the CI status for PR #55"
Agent: 
  1. 扫描skills，发现github skill适用
  2. read("skills/github/SKILL.md")
  3. 从SKILL.md中学习到使用 `gh pr checks` 命令
  4. exec("gh pr checks 55 --repo owner/repo")
```

### 10.4 总结

| 方面 | 是否包含 |
|------|---------|
| **Skill文件中的tool调用** | ❌ 不包含结构化的tool调用定义 |
| **Skill文件中的参数定义** | ❌ 不包含参数schema |
| **Skill文件中的指导文本** | ✅ 包含自然语言指导和示例 |
| **Command Dispatch配置** | ✅ 可通过frontmatter配置直接分发到工具 |
| **Agent执行时的tool调用** | ✅ Agent读取skill后，根据指导调用工具 |

**设计理念：**
- Skills是**指导性文档**，不是**结构化配置**
- Agent通过**阅读和理解**skill文档来学习如何使用工具
- 这种方式更灵活，允许skill包含复杂的逻辑和条件判断
- 同时支持**快速路径**：通过command-dispatch直接分发到工具

## 11. 总结

Clawdbot的skills机制是一个**延迟加载、按需选择**的系统：

1. **加载时**：从多个位置加载skills，按优先级合并
2. **过滤时**：根据环境、配置、依赖条件过滤eligible skills
3. **Prompt时**：将eligible skills格式化为XML列表注入system prompt
4. **选择时**：Agent根据任务描述扫描skills列表，选择最匹配的
5. **读取时**：Agent使用read工具读取选中skill的SKILL.md
6. **执行时**：Agent按照SKILL.md的指导执行操作

这种设计允许：
- **模块化**：每个skill是独立的SKILL.md文件
- **条件加载**：根据环境自动过滤不适用的skills
- **灵活选择**：Agent可以根据任务动态选择最合适的skill
- **用户调用**：支持通过命令直接调用skills
