# Learning Guide: RAG Indexes and Knowledge Retrieval for AI Agent Skill Files

**Generated**: 2026-03-31
**Sources**: 18 resources analyzed
**Depth**: deep

## Prerequisites

- Familiarity with SKILL.md format and Agent Skills open standard
- Basic understanding of how AI coding agents (Claude Code, Cursor, Copilot) load context
- Experience writing CLAUDE.md or similar project instruction files

## TL;DR

- Agent skill files are **not traditional RAG** - they use **progressive disclosure** (metadata -> instructions -> resources) rather than embedding-based retrieval
- The **description field** is the single most important element - it is the gatekeeper that determines whether a skill activates
- **XML structured references outperform plain markdown tables** for LLM parsing when documents are injected into context, but **markdown tables are better as human-readable routing indexes** that the model reads from disk
- **Semantic routing triggers in descriptions** beat direct file links for document selection - the model decides based on natural language matching, not path-based lookup
- For the actual skill files that agents read: keep SKILL.md **under 500 lines / 5,000 tokens**, use **one-level-deep file references**, and put a **table of contents** at the top of any reference file over 100 lines
- The state of the art for knowledge retrieval in AI coding tools is **not embedding-based RAG** - it is **filesystem-based progressive disclosure** with **description-driven activation**

## Core Insight: Skills Are Not Traditional RAG

Traditional RAG systems chunk documents, embed them into vectors, retrieve relevant chunks via similarity search, and inject them into the LLM context. AI agent skill files work differently.

The Agent Skills architecture uses a **three-tier progressive disclosure** model:

| Tier | What Loads | When | Token Cost |
|------|-----------|------|------------|
| Metadata | name + description only | Session startup | ~50-100 tokens per skill |
| Instructions | Full SKILL.md body | When skill is activated | <5,000 tokens recommended |
| Resources | Referenced files, scripts, data | When instructions reference them | Varies (effectively unlimited) |

This means the retrieval problem is really a **routing problem**: given a user's request, which skill should activate? The "index" is the collection of descriptions, not an embedding database.

**Key implication**: Optimizing skill-based knowledge retrieval means optimizing descriptions, file structure, and reference patterns - not building vector indexes.

## Question 1: XML Structured References vs Plain Markdown Tables

### For Context Injection (API / System Prompt)

When documents are programmatically injected into the LLM context (system prompt, tool results, API messages), **XML tags outperform plain markdown**.

Anthropic's official guidance recommends wrapping documents in structured XML:

```xml
<documents>
  <document index="1">
    <source>api-reference.md</source>
    <document_content>
      {{CONTENT}}
    </document_content>
  </document>
</documents>
```

Why XML wins here:
- Claude parses XML tags **unambiguously** - no edge cases with pipe characters or alignment
- Nested tags create **natural hierarchy** (documents > document > content)
- Tags enable **ground-then-answer** patterns: "Find quotes from the documents first, then answer"
- The model can distinguish injected documents from instructions and conversation reliably

### For Skill Routing Files (SKILL.md Body)

When the model reads a SKILL.md file from the filesystem and needs to decide which reference documents to load next, **markdown tables and structured lists work well**. The model reads these as structured text and follows links.

Example routing pattern that works:

```markdown
## Available References

**API patterns**: See [reference/api-patterns.md](reference/api-patterns.md)
**Data modeling**: See [reference/data-modeling.md](reference/data-modeling.md)
**Performance**: See [reference/performance.md](reference/performance.md)

## Quick Search

Find specific topics using grep:
grep -i "caching" reference/api-patterns.md
grep -i "indexing" reference/data-modeling.md
```

This works because the model is **reading a file and making a decision**, not parsing injected context. The links serve as a table of contents, not as a structured data format.

### Recommendation

Use **XML wrapping when injecting documents programmatically** into context. Use **markdown with clear section headers and file references** in SKILL.md routing files that the model reads from disk. Do not use XML inside SKILL.md files - they are markdown files read by both humans and models.

## Question 2: Direct File Links vs Semantic Routing Triggers

### How Skill Activation Actually Works

Agent tools use **description-based matching**, not file path matching. The activation flow:

1. At startup, the agent loads all skill **descriptions** into context (~50-100 tokens each)
2. When a user message arrives, the model reads all descriptions and decides which skill is relevant
3. The model loads the full SKILL.md of the matched skill
4. SKILL.md references point to deeper resources that the model loads on demand

The description is the **only thing** that determines whether a skill activates. Direct file links play no role in the initial routing decision.

### Semantic Routing Triggers Win

A description like this activates reliably:

```yaml
description: Build applications with Valkey, choose commands and data types,
  implement caching, sessions, queuing, locking, rate-limiting, leaderboard,
  counter, and search patterns. Use for streams, pub-sub, scripting,
  transactions, performance optimization, and migration from Redis.
```

It works because it contains **domain-specific vocabulary** that matches what developers actually say when they need this knowledge. The model performs semantic matching against these terms.

A file-path-based approach would not work:

```yaml
# This does NOT work for activation
description: Reference files in reference/valkey-patterns/
```

### Within SKILL.md: File References as Routing

Once a skill is activated and the model reads SKILL.md, file references serve as **second-level routing**. The model reads the SKILL.md body, understands the user's specific need, and selectively loads only the relevant reference file.

The pattern is:

```markdown
## Caching Patterns
For TTL strategies, eviction policies, and cache-aside patterns,
see [reference/caching.md](reference/caching.md)

## Data Structures
For choosing between Strings, Hashes, Sorted Sets, and Streams,
see [reference/data-structures.md](reference/data-structures.md)
```

The model reads this as a router: "The user asked about caching, so I'll load reference/caching.md." This is **semantic routing with file links as destinations** - the link itself is not the trigger, the surrounding description is.

### Recommendation

Use **semantic keywords in descriptions** for skill activation (tier 1 routing). Use **descriptive text + file links** in SKILL.md for second-level routing (tier 2). Never rely on file paths alone for routing - always pair with natural language context describing when to load each reference.

## Question 3: Embedding-Based vs Keyword vs Structured Metadata Retrieval

### State of the Art for General RAG

For general-purpose RAG (searching over large document collections), the state of the art is **hybrid retrieval**: combining dense embeddings with sparse keyword matching (BM25), followed by a reranking step.

Anthropic's Contextual Retrieval research shows:

| Approach | Retrieval Failure Reduction |
|----------|---------------------------|
| Embeddings alone | Baseline |
| Contextual Embeddings | 35% fewer failures |
| Contextual Embeddings + BM25 | 49% fewer failures |
| + Reranking | 67% fewer failures |

The key insight: **embedding-based and keyword-based retrieval are complementary**. Embeddings capture semantic similarity ("caching" matches "temporary storage"). BM25 captures exact terminology ("XADD" matches "XADD"). Combining both with a reranker gives the best results.

### State of the Art for AI Agent Skills

AI agent skill files do **not use embedding-based retrieval**. The retrieval mechanism is:

1. **Description matching** (tier 1): The LLM reads all descriptions and decides based on natural language understanding. This is more powerful than embedding similarity because the LLM understands intent, not just semantic proximity.

2. **Filesystem navigation** (tier 2): The model reads SKILL.md, sees references, and makes file-read tool calls. This is deterministic navigation, not retrieval.

3. **Grep/search** (tier 3): For large reference files, the model can use grep to find specific content. This is keyword matching executed by the model.

Some tools add embedding-based features on top:
- **Cursor** indexes the codebase with embeddings for `@codebase` semantic search
- **VS Code Copilot** has workspace indexing for `@workspace` queries
- **Claude Code** relies primarily on filesystem tools (Glob, Grep, Read) rather than embeddings

### Structured Metadata

Structured metadata in skill files improves routing precision:

```yaml
---
name: valkey-caching
description: Caching patterns with Valkey...
paths:
  - "src/**/*.ts"
  - "lib/cache/**"
metadata:
  domain: database
  technology: valkey
---
```

The `paths` field enables **conditional activation** - the skill only loads when the model works with matching files. This is a form of metadata-based filtering that reduces false positives.

### Recommendation

For skill files, invest in **description quality** and **structured SKILL.md routing** over embedding-based retrieval. The LLM's own comprehension of descriptions is the primary retrieval mechanism. For very large knowledge bases (100+ reference documents), consider adding grep-based search hints in SKILL.md. Embedding-based retrieval is most useful at the tool level (Cursor codebase indexing, Copilot workspace search) rather than at the skill routing level.

## Question 4: How AI Coding Tools Discover and Load Skill/Knowledge Files

### Claude Code

**Discovery**: Scans directories for SKILL.md files at startup.

| Scope | Path |
|-------|------|
| Personal | `~/.claude/skills/<name>/SKILL.md` |
| Project | `.claude/skills/<name>/SKILL.md` |
| Plugin | `<plugin>/skills/<name>/SKILL.md` |
| Enterprise | Managed settings |
| Nested | Subdirectory `.claude/skills/` auto-discovered |

**Context loading**: 
- Metadata (name + description) injected into system prompt at startup
- Full SKILL.md read via bash tool when triggered
- Referenced files read on demand via bash
- CLAUDE.md files loaded hierarchically from directory tree
- `.claude/rules/*.md` files loaded unconditionally or conditionally (with `paths` frontmatter)

**Description budget**: 1% of context window (default). Descriptions truncated at 250 chars in listings. Configurable via `SLASH_COMMAND_TOOL_CHAR_BUDGET`.

### Cursor

**Discovery**: Scans `.cursor/rules/` for MDC (Markdown Configuration) files and `.agents/skills/` for Agent Skills.

**Context loading**:
- `alwaysApply: true` rules loaded unconditionally
- `globs: "pattern"` rules loaded when working with matching files
- Codebase indexed with embeddings for `@codebase` semantic queries
- AGENTS.md and legacy `.cursorrules` supported for backward compatibility
- Agent Skills loaded via standard progressive disclosure

### GitHub Copilot / VS Code

**Discovery**: Scans `.github/copilot-instructions.md`, `.agents/skills/`, and `.github/skills/`.

**Context loading**:
- Custom instructions always included in context
- Agent Skills follow the standard progressive disclosure pattern
- `@workspace` triggers embedding-based codebase search
- VS Code extensions can provide additional context via language server protocol

### OpenAI Codex

**Discovery**: Scans `.agents/skills/` and `.codex/skills/` directories.

**Context loading**: Standard progressive disclosure. Full SKILL.md loaded on activation.

### Cross-Tool Standard

The `.agents/skills/` directory is the emerging cross-client convention. Skills placed there are discoverable by all compliant tools. Over 30 tools now support the Agent Skills open standard, including Gemini CLI, JetBrains Junie, OpenHands, Roo Code, Goose, Amp, Databricks, and Snowflake.

### Recommendation

For maximum cross-tool compatibility:
1. Place skills in `.agents/skills/<name>/SKILL.md` for cross-client discovery
2. Also place in `.claude/skills/<name>/SKILL.md` for Claude Code native support
3. Keep descriptions under 250 characters (front-load the key use case)
4. Use `paths` frontmatter for conditional activation when appropriate
5. Test with multiple tools to verify description matching works across different activation engines

## Question 5: Optimal File Size and Structure for RAG Chunks

### For Embedding-Based RAG Systems

When building traditional RAG indexes over knowledge documents:

| Parameter | Recommendation | Rationale |
|-----------|---------------|-----------|
| Chunk size | 256-512 tokens | Sweet spot for most embedding models |
| Overlap | 10-20% of chunk size | Preserves boundary context |
| Chunking strategy | Markdown-aware / semantic | Respects document structure |
| Context enrichment | Prepend parent section headers | Addresses the "lost context" problem |

Anthropic's Contextual Retrieval adds chunk-level context by having a model prepend a brief explanation of where each chunk fits in the document. This reduces retrieval failures by 35% over plain chunking.

For hierarchical knowledge (like the RAPTOR approach), multi-level summaries allow retrieval at different granularities - specific chunks for detailed questions, summary nodes for broad questions.

### For Agent Skill Files (Not Embedding-Based)

Skill files are read directly by the model, not chunked and embedded. The optimization targets are different:

| File Type | Size Target | Rationale |
|-----------|------------|-----------|
| SKILL.md body | Under 500 lines / ~5,000 tokens | Fits in context alongside conversation without crowding |
| Reference files | Under 300 lines each | Model can read entire file in one tool call |
| Large references | Include table of contents at top | Model may use `head -100` to preview, TOC ensures it sees full scope |
| Total skill directory | No practical limit | Files only load on demand |

### Structure for Optimal Model Consumption

Models parse structured documents better when they follow these patterns:

**1. Table of contents for files over 100 lines**

```markdown
# API Reference

## Contents
- Authentication (line 15)
- Core Methods: GET, SET, DEL (line 45)
- Advanced: Transactions, Scripting (line 120)
- Error Handling (line 180)
```

**2. Section headers as natural chunk boundaries**

```markdown
## Caching Patterns

### Cache-Aside (Lazy Loading)
[focused content for this pattern]

### Write-Through
[focused content for this pattern]
```

**3. One concept per reference file**

Instead of a 2000-line monolith reference, split into focused files:

```
reference/
  caching-patterns.md      (150 lines)
  data-structure-choice.md  (200 lines)
  performance-tuning.md     (180 lines)
  migration-from-redis.md   (120 lines)
```

The model loads only the file relevant to the user's question. Four 150-line files are better than one 600-line file because the model only pays the context cost for the portion it needs.

### Recommendation

For SKILL.md routing files: stay under 500 lines, use progressive disclosure to deeper references. For reference documents: keep each under 300 lines with clear section headers. For very large knowledge bases: split by domain/topic into focused files, with SKILL.md serving as the router. Always include a table of contents for files over 100 lines.

## Question 6: Designing Trigger Phrases for Maximum Recall Without False Positives

### How Trigger Matching Works

In AI agent skill systems, the model reads all skill descriptions at startup and uses its own language understanding to match user requests to skills. This is not keyword matching - it is semantic comprehension by the LLM.

The description is the **single most important field** for correct activation. Writing it well is the highest-leverage optimization for skill retrieval.

### Principles for High-Recall Descriptions

**1. Include both "what" and "when"**

```yaml
# Good - states both capability and trigger context
description: Build applications with Valkey, choose data types and commands.
  Use when implementing caching, sessions, queuing, locking, rate-limiting,
  leaderboards, or counters with Valkey or Redis.

# Bad - only states capability
description: Valkey application patterns.
```

**2. Front-load the primary use case**

Descriptions may be truncated at 250 characters. Put the most distinctive trigger phrase first:

```yaml
# Good - primary trigger is first
description: Debug and profile Valkey performance bottlenecks. Use for
  slow queries, memory analysis, latency spikes, or connection pool tuning.

# Bad - generic preamble wastes the first 250 chars
description: A comprehensive skill that provides detailed guidance for
  various aspects of working with Valkey performance issues...
```

**3. Use domain-specific vocabulary, not generic terms**

The model matches on semantic meaning, but domain-specific terms reduce ambiguity:

```yaml
# Good - specific vocabulary
description: Implement pub-sub messaging, Valkey Streams consumers,
  XADD/XREAD patterns, consumer groups, and event-driven architectures.

# Bad - too generic
description: Handle messaging and events with a database.
```

**4. Include common synonyms and alternative phrasings**

Users express the same intent in different ways:

```yaml
description: Deploy Valkey clusters, configure replication and failover,
  set up Sentinel for high availability. Use when scaling, deploying,
  or operating Valkey in production. Covers Docker, Kubernetes, and
  bare-metal setups.
```

This catches "deploy valkey", "scale valkey", "valkey kubernetes", "valkey sentinel", "valkey HA", and "valkey production" as triggers.

**5. Exclude explicitly to prevent false positives**

If a skill is frequently confused with another, use the description or SKILL.md instructions to clarify boundaries:

```yaml
description: Valkey client library usage with GLIDE SDK. Use when writing
  application code that connects to Valkey. Does NOT cover server
  administration, deployment, or internals.
```

### Trigger Phrase Design for CLAUDE.md Knowledge Indexes

For CLAUDE.md files that act as routing indexes (like the `agent-knowledge/CLAUDE.md` pattern), trigger phrases serve a different purpose. They are loaded into context and help the model match user questions to knowledge files.

The pattern:

```markdown
## Trigger Phrases

Use this knowledge when user asks about:
- "RAG chunking strategy" -> rag-skill-indexing.md
- "skill description quality" -> rag-skill-indexing.md
- "progressive disclosure" -> rag-skill-indexing.md
- "context window optimization" -> rag-skill-indexing.md
```

For these in-context triggers:
- Use **exact phrases users would type**, not academic descriptions
- Include **both short queries and longer questions**
- Add **tool-specific terms** ("CLAUDE.md", "SKILL.md", "cursor rules")
- Keep the total trigger section **under 30 entries per guide** to avoid context bloat

### Recommendation

Invest heavily in description quality - it has the highest impact on retrieval precision. Front-load the primary use case within 250 characters. Include domain-specific vocabulary and common synonyms. Use "Use when..." phrasing to explicitly state activation triggers. For CLAUDE.md routing indexes, map exact user phrases to knowledge files.

## Synthesis: The Optimal SKILL.md Knowledge Architecture

Combining all findings, here is the recommended architecture for AI agent skill files that route to reference knowledge:

### Directory Structure

```
skill-name/
  SKILL.md              # Router: <500 lines, progressive disclosure
  reference/
    topic-a.md          # Focused reference: <300 lines each
    topic-b.md          # One concept per file
    topic-c.md          # Include TOC if >100 lines
  scripts/
    helper.py           # Executable code (runs, not loaded into context)
  examples/
    pattern-1.md        # Example files for specific patterns
```

### SKILL.md Template

```yaml
---
name: domain-skill
description: <Primary capability in first sentence. Use when <trigger 1>,
  <trigger 2>, <trigger 3>. Covers <specific topic list>.>
---

# Domain Skill

## Quick Start

[Most common operation in <10 lines - gets the user productive immediately]

## Reference Topics

**Topic A**: [2-3 sentence description of what's covered]
See [reference/topic-a.md](reference/topic-a.md)

**Topic B**: [2-3 sentence description of what's covered]
See [reference/topic-b.md](reference/topic-b.md)

**Topic C**: [2-3 sentence description of what's covered]
See [reference/topic-c.md](reference/topic-c.md)

## Quick Search

Find specific items:
grep -i "keyword" reference/topic-a.md
grep -i "keyword" reference/topic-b.md
```

### Reference File Template

```markdown
# Topic Title

## Contents
- Section 1: [brief description]
- Section 2: [brief description]
- Section 3: [brief description]

## Section 1

[Focused content, practical examples, code samples]

## Section 2

[Focused content, practical examples, code samples]
```

### Key Principles

1. **Description is everything** - Invest 80% of optimization effort in the description field
2. **Progressive disclosure** - Only load what's needed, when it's needed
3. **One concept per file** - Reference files should be independently useful
4. **Table of contents** - Every file over 100 lines gets a TOC at the top
5. **Grep-friendly** - Include search hints for large reference sections
6. **One-level deep** - SKILL.md references files; files do not reference other files
7. **Under 500 lines** - SKILL.md body stays concise; deep content goes in reference files
8. **Front-load triggers** - Primary use case in the first 250 characters of description
9. **Test across models** - Haiku needs more guidance, Opus needs less explanation
10. **Iterate from observation** - Watch how the model navigates your skill, adjust based on actual behavior

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Giant monolith SKILL.md (1000+ lines) | Crowds context window, reduces adherence | Split into SKILL.md router + reference files |
| Deeply nested references (3+ levels) | Model may use `head -100` and miss content | Keep references one level deep from SKILL.md |
| Vague descriptions ("helps with data") | Model cannot match to user intent | Include specific vocabulary and trigger phrases |
| Embedding-based retrieval for <50 docs | Over-engineering; filesystem navigation is faster | Use description-based routing with file references |
| XML tags inside SKILL.md body | SKILL.md is markdown read by humans and models | Use XML only for programmatic context injection |
| No table of contents in long files | Model may partially read and miss important sections | Add TOC at top of any file over 100 lines |
| Duplicate content across reference files | Wastes context when multiple files loaded | Single source of truth per concept |
| Description longer than 250 chars without front-loading | Key triggers truncated in skill listings | Put the most distinctive trigger phrase first |

## Further Reading

- Anthropic Skill Authoring Best Practices: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- Agent Skills Open Standard: https://agentskills.io/specification
- Claude Code Skills Documentation: https://code.claude.com/docs/en/skills
- Claude Code Memory System: https://code.claude.com/docs/en/memory
- Anthropic Context Engineering: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic Contextual Retrieval: https://www.anthropic.com/research/contextual-retrieval
- Anthropic Prompting Best Practices: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
