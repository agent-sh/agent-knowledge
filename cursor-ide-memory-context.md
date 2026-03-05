# Learning Guide: Cursor IDE Memory Files and Project Context

**Generated**: 2026-02-22
**Sources**: 20 resources analyzed
**Depth**: medium

## Prerequisites

- Basic familiarity with an AI-assisted code editor (VS Code, Cursor, or similar)
- Understanding of what a system prompt is and how LLMs use context windows
- Awareness that Cursor is built on VS Code and uses model APIs (Anthropic, OpenAI, Google)

## TL;DR

- Cursor does not retain memory between sessions natively - rules files (`.cursor/rules/`, `AGENTS.md`) are the primary mechanism for injecting persistent project context into every conversation.
- The legacy `.cursorrules` file (project root, plain text) still works but is deprecated; migrate to `.cursor/rules/` MDC files or `AGENTS.md`.
- `AGENTS.md` is the cross-tool compatible format: plain markdown, supports nested directories, no frontmatter required.
- Cursor 2.0 removed the Notepads feature and several manual `@`-mention context types; the agent now gathers context automatically via semantic search.
- Codebase context is provided through vector-indexed semantic search (updated every 5 minutes), not in-memory storage.

---

## Core Concepts

### 1. How Cursor Reads Project Context

LLMs do not retain state between completions. Cursor compensates for this through prompt-level context injection: rules files, codebase indexing, and explicit `@`-mentions. Every chat session starts fresh; the "memory" is always re-injected from files on disk.

The agent has three sources of context:

1. **Instructions** - System prompt + rules (always-apply and auto-detected)
2. **Codebase index** - Semantic vector search over your repository
3. **User messages** - Explicit `@`-mentions and conversation history within the session

### 2. Rules Files (Primary Memory Mechanism)

Rules are the closest Cursor equivalent to Claude Code's `CLAUDE.md`. Rule contents are injected at the start of the model context window on each relevant invocation.

**Four rule types:**

| Type | Storage | Scope |
|------|---------|-------|
| Project Rules | `.cursor/rules/` (version-controlled) | Per-repository |
| User Rules | Cursor settings UI | All projects on this machine |
| Team Rules | Cursor dashboard | All projects in the org (Team/Enterprise) |
| AGENTS.md | Project root or subdirectories | Per-directory |

**Precedence order**: Team Rules > Project Rules > User Rules

### 3. .cursor/rules/ Directory (MDC Format)

Project rules live in `.cursor/rules/` as `.md` or `.mdc` files. The `.mdc` extension unlocks YAML frontmatter for fine-grained control:

```markdown
---
description: "TypeScript coding standards for this repo"
alwaysApply: false
globs: ["**/*.ts", "**/*.tsx"]
---

# TypeScript Standards

- Use strict mode
- Prefer `type` over `interface` for unions
- No `any` - use `unknown` and narrow it
```

**Application modes controlled by frontmatter:**

- **Always Apply** (`alwaysApply: true`) - Injected in every session regardless of context
- **Apply Intelligently** (description only, no globs) - Agent decides relevance using the description
- **Apply to Specific Files** (`globs` array) - Triggered when matched files are in context
- **Apply Manually** (no frontmatter) - User must `@mention` the rule explicitly

Rules can be triggered manually via `@rule-name` in chat.

**Best practices for rule content:**
- Keep each rule file under 500 lines
- One concern per file (e.g., `typescript-style.mdc`, `testing-conventions.mdc`, `api-design.mdc`)
- Reference external files rather than copying their contents inline
- Do not duplicate what a linter already enforces
- Do not document basic commands the AI already knows
- Add rules reactively: "add a rule when you notice the agent making the same mistake repeatedly"

### 4. AGENTS.md (Cross-Tool Compatible Format)

`AGENTS.md` is plain markdown without frontmatter. It is simpler than `.cursor/rules/` and intentionally compatible with other AI coding tools (OpenCode, Codex, etc.).

**Placement and nesting:**

```
project/
  AGENTS.md              # Global project instructions - applies everywhere
  src/
    frontend/
      AGENTS.md          # Frontend-specific - applies in frontend/ subtree
    backend/
      AGENTS.md          # Backend-specific - applies in backend/ subtree
```

When the agent is working in a subdirectory, instructions combine hierarchically. More specific (deeper) files take precedence over parent-level files.

`AGENTS.md` is read by the agent automatically. No configuration required.

**When to use AGENTS.md vs .cursor/rules/:**

Use `AGENTS.md` when you want:
- Simple, readable instructions that work across multiple AI tools
- No metadata or conditional application logic
- Team members who aren't Cursor users to also benefit from the file

Use `.cursor/rules/` MDC files when you want:
- Rules scoped to specific file patterns (globs)
- Manual-only rules invoked via `@mention`
- Agent-evaluated relevance based on description
- Multiple composable rule pieces

### 5. Legacy .cursorrules File

The `.cursorrules` file (placed in the project root, plain text, no frontmatter) predates the `.cursor/rules/` system. It is still supported as of early 2026 but officially deprecated. Cursor documentation recommends migrating to `.cursor/rules/` or `AGENTS.md`.

Community repositories like `awesome-cursorrules` (38k+ GitHub stars) maintain a large library of `.cursorrules` templates for popular stacks. These patterns remain valid content even when migrated to the new format.

**Migration path:**
1. Move `.cursorrules` content to `.cursor/rules/project-conventions.mdc`
2. Add `alwaysApply: true` frontmatter if you want equivalent always-on behavior
3. Or split the content into focused rule files with appropriate glob patterns

### 6. Codebase Indexing and Semantic Search

Cursor indexes your repository into a vector database stored on Cursor's servers. This provides the "codebase understanding" that lets the agent navigate your project without you manually attaching files.

**Indexing pipeline:**
1. Files are synchronized to Cursor's servers
2. Code is chunked into logical units (functions, classes, blocks)
3. Each chunk is embedded into a vector representation
4. Vectors are stored in a similarity-search database
5. Index refreshes every 5 minutes; deleted files are removed promptly

**What gets indexed:** All files not excluded by `.cursorignore` or `.gitignore`

**What is excluded automatically:** `node_modules/`, build artifacts, binaries, media, lock files, and files matching `.gitignore`

**Two ignore files:**
- `.cursorignore` - Excludes files from all AI features (semantic search, Tab, Agent, @mentions)
- `.cursorindexingignore` - Excludes from the index only; files remain accessible via explicit @mention

The agent retrieves relevant context on demand using semantic search - you rarely need to manually attach files. This replaces many of the manual `@`-mention context types that existed in pre-2.0 Cursor.

### 7. Conversation Context and Session Memory

Cursor does not persist memory across sessions. Each new chat window starts without knowledge of previous conversations.

**Within a session:**
- Cursor automatically summarizes conversation history as it grows to fit within the context window
- Users can manually trigger `/summarize` to compress the conversation and continue efficiently
- The agent can use `@Past Chats` to reference a previous conversation explicitly (adds the full prior conversation as context)

**Context window options:**
- Default: ~200k tokens
- Max Mode: Extends to the model's maximum supported context window (20% cost premium); beneficial for very large codebases or complex multi-file tasks

**Checkpoints:** Cursor creates automatic snapshots before agent modifications, enabling undo.

**Export:** Chats can be exported as markdown or shared as read-only links (paid plans).

### 8. @Mentions for Explicit Context

When the agent does not automatically find the right context, you can attach it explicitly:

| Mention | What it includes |
|---------|-----------------|
| `@Files` | Entire file contents |
| `@Folders` | Folder path and contents overview |
| `@Code` | Specific selected code snippet |
| `@Docs` | Built-in or custom URL documentation |
| `@Past Chats` | Full previous conversation history |
| `@rule-name` | Manually invoke a specific rule |

**Removed in Cursor 2.0:** `@Definitions`, `@Web`, `@Link`, `@Recent Changes`, `@Linter Errors` - these were replaced by automatic agent context gathering.

**Custom documentation:** `@Docs` allows you to add any URL. Cursor reads the page and all subpages, making the documentation available to the agent.

### 9. Skills System (SKILL.md Files)

Skills are a newer addition (January 2026) extending the rules concept with executable components. A skill is a version-controlled package that teaches the agent domain-specific tasks.

**Structure:**

```
.cursor/skills/
  deploy/
    SKILL.md          # Frontmatter + instructions
    scripts/
      deploy.sh       # Referenced scripts
```

**SKILL.md frontmatter:**

```yaml
---
name: deploy
description: "Deploy the application to staging or production"
---
```

Skills are discovered from `.cursor/skills/`, `~/.cursor/skills/`, and marketplace locations. The agent loads them contextually based on relevance, or users invoke them with `/skill-name`.

Setting `disable-model-invocation: true` converts a skill to an explicit-only command (like a traditional slash command).

### 10. Agent Hooks for Context Injection

Hooks (`hooks.json`) allow injecting additional context at specific lifecycle points:

```json
{
  "sessionStart": {
    "command": "./scripts/inject-context.sh",
    "additional_context": "..."
  }
}
```

The `sessionStart` hook is particularly relevant for memory: it runs before the first agent turn and can inject dynamic context (e.g., current git branch, environment state, team-specific runtime information) that rules files cannot provide because it changes.

Hook configuration inherits hierarchically: enterprise → team → project → user.

### 11. MCP for External Memory and Context

The Model Context Protocol (MCP) extends Cursor's context beyond what fits in the token window or the local codebase index. MCP servers connect to external systems:

- **Documentation systems** (AWS docs, framework docs, internal wikis)
- **Project management** (Linear, GitHub Issues, Jira)
- **Databases** (query live data during code generation)
- **Version control** (Git history, PR discussions)
- **Communication** (Slack threads, Notion pages)

This is the recommended approach for "memory" that spans multiple sessions or multiple developers - store it externally in a system Cursor can query via MCP, rather than trying to embed it all in rules files.

### 12. Notepads (Removed in Cursor 2.0)

Cursor previously had a Notepads feature for storing reusable text snippets that could be referenced in chat. This feature was removed in Cursor 2.0. The replacement approaches are:

- Use `AGENTS.md` or `.cursor/rules/` for persistent project instructions
- Use `@Docs` with a URL for reference documentation
- Use `@Past Chats` to reference previous discussions
- Store project decisions in `.cursor/plans/` (saved plan files)

---

## File Structure Reference

```
project/
├── .cursor/
│   ├── rules/                    # Project rules (MDC format)
│   │   ├── always-on.mdc         # alwaysApply: true
│   │   ├── typescript.mdc        # globs: ["**/*.ts"]
│   │   ├── testing.mdc           # description-based auto-apply
│   │   └── manual-rule.md        # No frontmatter = manual @mention only
│   ├── skills/                   # Agent skills
│   │   └── deploy/
│   │       └── SKILL.md
│   └── plans/                    # Saved implementation plans
├── .cursorrules                  # DEPRECATED - legacy project rules
├── .cursorignore                 # Exclude from all AI features
├── .cursorindexingignore         # Exclude from index only
├── AGENTS.md                     # Cross-tool compatible project instructions
└── src/
    └── frontend/
        └── AGENTS.md             # Subdirectory-specific instructions
```

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Using `.cursorrules` for new projects | Legacy format familiar from tutorials | Use `.cursor/rules/` MDC or `AGENTS.md` instead |
| One giant rules file | Easy to start with one file | Split into focused files under 500 lines each; use glob scoping |
| Rules duplicating linter config | Feels thorough | Trust the linter; rules are for conventions and judgment, not enforcement |
| Expecting memory across sessions | Assumes chat-like persistence | Use rules files for persistent context; use `@Past Chats` for session replay |
| Not using `.cursorignore` for secrets | Secrets are just files | Add `.env`, credential files, and sensitive configs to `.cursorignore` |
| Overly broad `alwaysApply: true` | Wants AI to always know everything | Only `alwaysApply` for truly universal conventions; use descriptions for others |
| Ignoring nested AGENTS.md | Unaware of the feature | Place subdirectory-specific instructions close to the code they govern |

---

## Best Practices

1. **Start with AGENTS.md** for simple projects or when the team uses multiple AI tools. Add `.cursor/rules/` MDC files as needs grow.

2. **Scope rules to files they govern.** A TypeScript style rule with `globs: ["**/*.ts"]` is better than an always-on rule - it only loads when relevant, saving context tokens.

3. **Write rules reactively, not speculatively.** Add a rule after you see the agent make the same mistake twice. Do not try to predict all edge cases upfront.

4. **Use `description` for intelligent application.** A well-written description (1-2 sentences) allows the agent to decide when a rule is relevant without you manually invoking it.

5. **Combine `.cursorignore` with good hygiene.** Keep secrets, large generated files, and vendored code out of the AI's context. This makes semantic search more accurate and protects sensitive data.

6. **Use MCP for team-shared knowledge.** Rules files are individual developer or repository configuration. For shared organizational context (runbooks, architecture decisions, API docs), serve it via MCP so it is always current.

7. **Save plans to `.cursor/plans/`.** After planning a complex feature, use "Save to workspace." This creates a reference document that future agent sessions can read as a `@File`.

8. **Use Max Mode selectively.** Enable it for large codebases or when the agent is missing relevant context it should be finding. The 20% cost premium adds up for routine tasks.

---

## Summary: Comparison with Claude Code's CLAUDE.md

| Claude Code | Cursor Equivalent | Notes |
|-------------|-------------------|-------|
| `CLAUDE.md` (project root) | `AGENTS.md` (project root) | Both are plain markdown; identical cross-tool format |
| `CLAUDE.md` (subdirectory) | `AGENTS.md` (subdirectory) | Both support hierarchical nesting |
| `.claude/` directory | `.cursor/` directory | Tool-specific configs live here |
| No equivalent | `.cursor/rules/*.mdc` | Cursor adds metadata, glob scoping, apply modes |
| No equivalent | `.cursor/skills/` | Cursor adds executable skill packages |
| `hooks.json` equivalent | `hooks.json` in `.cursor/` | Similar lifecycle hook system |
| MCP servers | MCP servers | Both support MCP for external context |

---

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [Cursor Rules Documentation](https://cursor.com/docs/context/rules) | Official Docs | Complete reference for all rule types and MDC format |
| [Cursor Agent Overview](https://cursor.com/docs/agent/overview) | Official Docs | How the agent uses rules, AGENTS.md, and SKILL.md |
| [Cursor Skills Documentation](https://cursor.com/docs/context/skills) | Official Docs | SKILL.md format and skill discovery |
| [Cursor Hooks Documentation](https://cursor.com/docs/agent/hooks) | Official Docs | Session lifecycle hooks for dynamic context injection |
| [Cursor Semantic Search](https://cursor.com/docs/context/semantic-search) | Official Docs | How codebase indexing works |
| [Cursor Ignore Files](https://cursor.com/docs/context/ignore-files) | Official Docs | .cursorignore vs .cursorindexingignore |
| [Cursor MCP Documentation](https://cursor.com/docs/context/mcp) | Official Docs | Extending context via Model Context Protocol |
| [awesome-cursorrules (GitHub)](https://github.com/PatrickJS/awesome-cursorrules) | Community | 38k+ star collection of community rule templates by stack |
| [cursorrules.org](https://www.cursorrules.org) | Community | AI-generated rule configs for popular frameworks |
| [Cursor Quickstart](https://cursor.com/docs/get-started/quickstart) | Official Docs | Includes note on saving plans to `.cursor/plans/` |

---

*Generated by /learn from 20 sources.*
*See `resources/cursor-ide-memory-context-sources.json` for full source metadata.*
