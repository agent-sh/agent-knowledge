# Learning Guide: Libraries Shipping Agent Skills - Adoption, Integration & Migration

**Generated**: 2026-03-19
**Sources**: 20 resources analyzed
**Depth**: medium

## Prerequisites

- Basic understanding of agent skills (SKILL.md format)
- npm ecosystem familiarity (publishing, package.json)
- Experience maintaining or using open-source libraries

## TL;DR

- Libraries are now shipping agent skills inside their npm packages so AI agents can correctly use their APIs
- This solves the **version fragmentation problem** - training data contains multiple library versions with no way to disambiguate
- **TanStack Intent** is the leading CLI for generating, validating, and distributing library skills via npm
- Real adopters: ElectricSQL, TanStack, Vercel (React/Next.js), OpenAI (Agents SDK)
- Skills function as an **adoption funnel** - developers who install skills are already building, already at the point of adoption
- The pattern shifts discovery from "Google -> landing page -> signup" to "agent discovers skill -> developer installs -> product adopted"

## The Problem: Why Libraries Need Skills

### Version Fragmentation

When a library ships a breaking change, AI models don't "catch up." Training data contains both old and new API versions forever, with no way for the agent to know which version the developer has installed. The agent might suggest `v2` patterns to someone on `v3`, or mix APIs from different major versions in the same file.

**Key insight**: This is the core motivation. Community-maintained rules files, copy-pasted docs, and README snippets all lack versioning. When `npm update` ships a breaking change, none of these knowledge sources update.

### The Manual Discovery Problem

Before library-shipped skills, the developer workflow was:
1. Google "how to use X with AI agent"
2. Find a community rules file (maybe outdated)
3. Copy-paste into `.cursorrules` or `CLAUDE.md`
4. Hope it matches their installed version
5. Manually update when the library changes

This doesn't scale and creates a maintenance burden on the community rather than the library maintainers who know their APIs best.

## The Solution: Skills as Part of the Package

### How It Works

```
1. Library maintainer creates skills in their repo
2. Skills ship inside the npm package
3. Developer runs: npx @tanstack/intent install
4. CLI traverses node_modules, finds intent-enabled packages
5. Skills auto-wire into agent config (CLAUDE.md, .cursorrules, etc.)
6. npm update -> skills update -> agent stays current
```

**Key insight**: Skills version-lock to the installed library version. The agent gets documentation for exactly the version the developer is using, not whatever was in training data.

## Real-World Examples

### ElectricSQL - Database Sync Library

ElectricSQL collaborated with TanStack to ship skills for `@tanstack/db`, `@electric-sql/client`, and `@durable-streams/client`. Their motivation: *"once a breaking change ships, models don't 'catch up.' They develop a permanent split-brain."*

After installing:
```bash
npx @tanstack/intent install
```
The agent understands the correct APIs for the installed version. Bug reports on skill quality ship fixes to everyone on the next release.

### Vercel - React & Next.js Skills

Vercel ships official agent skills covering:

| Skill | What It Does |
|-------|-------------|
| `react-best-practices` | 40+ rules across 8 categories (memoization, bundle optimization, server components) |
| `web-design-guidelines` | 100+ rules for accessibility, focus, forms, animation, typography, dark mode, i18n |
| `vercel-deploy-claimable` | Creates claimable deployments on Vercel |
| `next-skills` | Next.js version migration guides, cache components, PPR patterns |

Installation:
```bash
npx skills add vercel-labs/agent-skills
```

These encode **10 years of React and Next.js optimization knowledge** into agent-consumable format. When an agent reviews React code, it loads `react-best-practices` to ensure components follow performance patterns.

### OpenAI - Agents SDK Maintenance

OpenAI uses skills to maintain their own Python and TypeScript Agents SDK repos. Between Dec 2025 and Feb 2026, this approach helped increase PR merges from 316 to 457.

Their skills cover the full maintenance lifecycle:

| Skill | Purpose |
|-------|---------|
| `code-change-verification` | Formatting, linting, type-checking, tests |
| `docs-sync` | Audit docs against codebase for drift |
| `examples-auto-run` | Execute examples in automated mode |
| `final-release-review` | Compare release tags, check readiness |
| `implementation-strategy` | Determine compatibility boundaries before API changes |
| `changeset-validation` | Ensure Changesets match package diffs |
| `integration-tests` | Publish to local Verdaccio, test across Node/Bun/Deno/browsers |

**Key insight**: OpenAI's approach shows skills aren't just for external developers - they also help **maintainers** automate their own recurring engineering work. The `AGENTS.md` file acts as a policy layer: "Run `$code-change-verification` when runtime code changes."

### TanStack - The Tooling Provider

TanStack built the tooling (Intent CLI) and is also a user. Their libraries ship skills for TanStack Router, TanStack Query, and TanStack DB, with skills generated from their own documentation via `npx @tanstack/intent scaffold`.

### Remotion - Programmatic Video

Remotion's skill announcement tweet pulled 18,000 likes and 14.8M views. They ship 37+ skills covering animations, 3D, subtitles, audio, charts, and transitions.

```bash
npx skills add remotion-dev/skills
```

Skills are modular - 28+ rule files load dynamically based on what you're building (e.g., only animation rules when working on animations). This is a textbook example of progressive disclosure at the library level. 150K+ installs on skills.sh.

### Prisma - ORM with Version Migration

Prisma ships official skills targeting v7.x:

| Skill | Purpose |
|-------|---------|
| Prisma CLI reference | All CLI commands: init, generate, migrate dev/deploy, db push/pull/seed, studio |
| Prisma Client API | CRUD, query options, filter operators, transactions, raw queries |
| `prisma-upgrade-v7` | **Migration skill** - step-by-step from v6 to v7: ESM config, driver adapters, prisma.config.ts, removed features |

The migration skill is the most interesting pattern - it specifically helps agents guide developers through breaking changes between major versions.

### Supabase - Database Platform

Supabase ships `supabase-postgres-best-practices` with references across 8 categories prioritized by impact:

| Category | Priority |
|----------|----------|
| Query Performance | Critical |
| Connection Management | Critical |
| Schema Design | High |
| Concurrency & Locking | Medium-High |
| Security & RLS | Medium-High |
| Data Access Patterns | Medium |
| Monitoring & Diagnostics | Low-Medium |
| Advanced Features | Low |

```bash
npx skills add supabase/agent-skills
```

They also ship MCP server integration (`@supabase/mcp`) complementing the skills - MCP for tool access (run queries, manage projects), skills for knowledge (best practices, patterns).

### Callstack - React Native Ecosystem

Callstack ships 3 focused skills that map to the React Native adoption lifecycle:

| Skill | Lifecycle Stage |
|-------|----------------|
| `react-native-best-practices` | **Day-to-day development** - derived from "The Ultimate Guide to RN Optimization" |
| `react-native-brownfield-migration` | **Adoption** - incremental migration into existing native apps |
| `upgrading-react-native` | **Maintenance** - version upgrade challenges (dependency alignment, React 19, patches) |

**Key insight**: Callstack codified learnings from upgrading their biggest clients' apps. The upgrade skill captures real-world pain points (abandoned dependencies, testing breakage) that aren't in official docs.

### Stripe - Payment Integration

Stripe takes a multi-layer approach:

| Layer | Tool | What It Does |
|-------|------|-------------|
| MCP Server | `@stripe/mcp` | Direct API access via function calling (remote at mcp.stripe.com or local) |
| Agent Toolkit | `@stripe/agent-toolkit` | Framework integration (OpenAI SDK, LangChain, CrewAI, Vercel AI SDK) |
| Skills | Integration best practices | Guidance on which products/features to use, setup patterns |
| Token Meter | `@stripe/token-meter` | Billing integration with OpenAI/Anthropic/Google SDKs |

Stripe shows how skills and MCP complement each other: MCP provides tool access (create charges, manage subscriptions), skills provide knowledge (which API to use when, integration patterns).

### Microsoft - Azure & .NET SDKs

Microsoft's `microsoft/skills` repo ships 126 skills - 5 core + 121 language-specific (Python, .NET, TypeScript, Java):

| Category | Skills |
|----------|--------|
| Azure Cosmos DB | SDK patterns, query optimization |
| AI Foundry | Project client, agents, evals, connections |
| AI Search | Vector search, hybrid search, semantic ranking |
| Azure Deployment (AZD) | Infrastructure as code, deployment workflows |
| .NET Skills | `dotnet/skills` - platform team's own skills for .NET developers |

Additionally, `microsoft/azure-skills` ships 20 curated Azure-specific skills.

### Coinbase - Blockchain/Web3

Coinbase shipped Base blockchain skills for developers building on their L2 chain. Part of the early Q1 2026 wave of major companies adopting the format.

## Summary: Who's Shipping What

| Company/Library | Install | Skills Count | Focus |
|-----------------|---------|-------------|-------|
| **Vercel** | `npx skills add vercel-labs/agent-skills` | 3+ | React/Next.js best practices, deployment |
| **Remotion** | `npx skills add remotion-dev/skills` | 37+ | Video rendering, animations, 3D |
| **Prisma** | `npx skills add prisma/skills` | 3+ | ORM usage, CLI, v6->v7 migration |
| **Supabase** | `npx skills add supabase/agent-skills` | 1 (8 categories) | Postgres best practices |
| **Callstack** | Clone `callstackincubator/agent-skills` | 3 | React Native best practices, brownfield, upgrades |
| **Stripe** | `npm i @stripe/agent-toolkit` | MCP + skills | Payment integration patterns |
| **Microsoft** | `npx skills add microsoft/skills` | 126 | Azure SDKs, .NET, AI services |
| **ElectricSQL** | `npx @tanstack/intent install` | 3 | Database sync, streams |
| **TanStack** | `npx @tanstack/intent install` | Multiple | Router, Query, DB |
| **OpenAI** | Repo-local | 8+ (Python), 11+ (JS) | SDK maintenance, releases |
| **Coinbase** | Via skills.sh | Multiple | Base blockchain |

As of March 2026, skill adoption topped 351,000 installs across the ecosystem.

## TanStack Intent - The Distribution Pipeline

### CLI Commands for Library Maintainers

```bash
npx @tanstack/intent scaffold    # Generate skill drafts from your docs
npx @tanstack/intent validate    # Check well-formedness
npx @tanstack/intent stale       # Flag skills drifting from source docs
npx @tanstack/intent setup-github-actions  # CI validation workflows
```

### CLI Commands for Developers

```bash
npx @tanstack/intent install     # Discover + wire skills from node_modules
npx @tanstack/intent list        # Show intent-enabled packages
npx @tanstack/intent feedback    # Report skill issues
```

### Staleness Detection

Each skill declares its source documentation:

```yaml
name: tanstack-router-search-params
description: Type-safe search param patterns for TanStack Router
metadata:
  sources:
    - docs/framework/react/guide/search-params.md
```

When those docs change, `npx @tanstack/intent stale` flags the skill. Run it in CI and you get a failing check when sources drift - skills become part of the release checklist.

### Alternative: npm-agentskills

Anthony Fu's `skills-npm` and the `npm-agentskills` package offer a lighter approach:

```json
{
  "name": "my-library",
  "agents": {
    "skills": ["./skills"]
  }
}
```

The `agents` field in `package.json` points to skill directories. Tools like `npx skills add` discover and install them.

## Skill Categories for Libraries

Based on patterns from real adopters, library skills typically fall into these categories:

### 1. Integration Skills (Getting Started)

Teach the agent how to set up and use the library correctly:
- Correct import patterns and initialization
- Connection/configuration for the installed version
- Common "hello world" patterns
- Environment-specific setup (Node, Edge, Deno, browsers)

### 2. Migration Skills (Version Upgrades)

Help developers upgrade between library versions:
- API mapping tables (old -> new)
- Breaking change checklists
- Codemods or automated migration steps
- Deprecated pattern detection

**Example**: Vercel's `next-skills` includes migration guides for upgrading between Next.js versions.

### 3. Best Practices Skills (Correct Usage)

Encode expert knowledge about optimal patterns:
- Performance patterns (memoization, lazy loading, batching)
- Security patterns (input validation, auth flows)
- Architecture patterns (component structure, data flow)

**Example**: Vercel's `react-best-practices` with 40+ rules across 8 categories.

### 4. Maintenance Skills (For Library Maintainers)

Help maintainers manage their own repos:
- PR review workflows
- Release readiness checks
- Doc-code sync validation
- Test coverage analysis

**Example**: OpenAI's entire skill suite for Agents SDK maintenance.

### 5. Ecosystem Skills (Interop)

Teach correct patterns for using the library with common companions:
- Framework integration (React + your library, Fastify + your library)
- Testing patterns (Jest, Vitest with your library)
- Deployment patterns (Vercel, AWS, Docker)

## The Adoption Funnel Shift

Traditional library adoption:
```
Google search -> Landing page -> Docs -> Signup -> Integration
```

Skills-based adoption:
```
Agent discovers skill -> Developer installs -> Agent builds correctly -> Product adopted
```

Early data suggests skill installs correlate with activation. Developers who install skills are already in their IDE, already building, already at the point of adoption. As of March 2026, agent skill adoption topped 351,000 installs.

**Key insight**: Skills shift the competitive advantage from "best docs" to "best agent experience." Libraries that ship skills get used correctly by agents; libraries that don't get hallucinated APIs and frustrated developers.

## Common Pitfalls for Library Maintainers

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Skills drift from library | Docs updated but skills not | Use `@tanstack/intent stale` in CI |
| Over-explaining basics | Adding context agents already know | Focus on YOUR API specifics, not generic concepts |
| One giant skill | Dumping all docs into one SKILL.md | Split by use case: getting-started, migration, advanced |
| Not testing with agents | Assuming correctness | Use the Two-Claude method: one creates, one tests |
| Shipping without version alignment | Skills reference wrong API version | Ship inside the npm package, not as a separate repo |
| Ignoring feedback loop | No mechanism for users to report skill bugs | Use `@tanstack/intent feedback` or GitHub issues |

## Best Practices for Library Maintainers

1. **Ship skills INSIDE your npm package** - version-lock to your library
2. **Generate from your docs** - `npx @tanstack/intent scaffold` drafts from existing documentation
3. **Staleness detection in CI** - fail builds when skills drift from source docs
4. **Split by use case** - separate getting-started, migration, best-practices, and advanced skills
5. **Focus on what agents get wrong** - test your library with agents first, then create skills for the failure modes
6. **Include migration skills** - agents frequently help with version upgrades, give them correct migration paths
7. **Encode gotchas prominently** - the things that trip up developers are the highest-value skill content
8. **Use the feedback loop** - skill bug reports -> fix -> npm publish -> everyone benefits
9. **Keep skills concise** - under 500 lines per SKILL.md, push details to reference files
10. **Test across agent tools** - Claude Code, Cursor, Copilot, Codex all have different strengths

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [TanStack Intent Blog Post](https://tanstack.com/blog/from-docs-to-agents) | Blog | The manifesto for library-shipped skills |
| [TanStack Intent Docs](https://tanstack.com/intent/latest/docs/overview) | Official Docs | CLI reference and configuration |
| [ElectricSQL Skills Announcement](https://electric-sql.com/blog/2026/03/06/agent-skills-now-shipping) | Blog | First major adopter case study |
| [OpenAI: Skills for SDK Maintenance](https://developers.openai.com/blog/skills-agents-sdk) | Blog | How OpenAI uses skills for their own repos |
| [Vercel Agent Skills](https://vercel.com/docs/agent-resources/skills) | Official Docs | React/Next.js skills directory |
| [Vercel Agent Skills Repo](https://github.com/vercel-labs/agent-skills) | GitHub | Source code for Vercel's skills |
| [Anthropic: Agent Skills Vision](https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills) | Blog | Anthropic's strategy for the skills ecosystem |
| [Agent Skills Specification](https://agentskills.io/what-are-skills) | Spec | Open format specification |
| [npm-agentskills](https://github.com/onmax/npm-agentskills) | GitHub | Alternative: bundle skills via package.json |
| [skills.sh Directory](https://skills.sh) | Directory | Browse available skills from libraries |

---

*Generated by /learn from 20 sources.*
*See `resources/skills-for-library-and-sdk-best-practices-sources.json` for full source metadata.*
