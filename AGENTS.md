# Agent Knowledge Base

> Learning guides created by /learn. Reference these when answering questions about listed topics.

## Available Topics

| Topic | File | Sources | Depth | Created |
|-------|------|---------|-------|---------|
| AI CLI Non-Interactive & Programmatic Usage | ai-cli-non-interactive-programmatic-usage.md | 52 | deep | 2026-02-10 |
| AI CLI Advanced Integration (MCP, SDKs, Cross-Tool) | ai-cli-advanced-integration-patterns.md | 38 | deep | 2026-02-10 |
| Skill/Plugin Distribution Patterns | skill-plugin-distribution-patterns.md | 40 | deep | 2026-02-21 |
| All-in-One Plus Modular Packages | all-in-one-plus-modular-packages.md | 40 | deep | 2026-02-21 |
| GitHub Org Project Management (Multi-Repo OSS) | github-org-project-management.md | 15 | deep | 2026-02-21 |
| GitHub Organization Structure Patterns | github-org-structure-patterns.md | 18 | deep | 2026-02-21 |
| Browser Agent Auth Flows (Slack, Notion, AWS) | browser-agent-auth-flows.md | training | deep | 2026-02-22 |
| Social Platform Auth Flows (X, Reddit, Discord, LinkedIn) | social-platform-auth-flows.md | 35 | deep | 2026-02-22 |
| Google & Microsoft Auth Flows for Browser Agents | google-microsoft-auth-flows.md | 28 | deep | 2026-02-22 |
| Auth Flows: GitHub, GitLab, Atlassian (Browser Agent) | auth-flows-github-gitlab-atlassian.md | 28 | deep | 2026-02-22 |
| Generic Auth Patterns & CAPTCHA Systems | generic-auth-patterns-captcha-systems.md | 40 | deep | 2026-02-22 |
| Cursor IDE Memory Files & Project Context | cursor-ide-memory-context.md | 20 | medium | 2026-02-22 |
| Multi-Product Org Docs & Website Architecture | multi-product-org-docs.md | 42 | deep | 2026-02-21 |
| ACP with Codex, Gemini, Copilot, Claude | acp-with-codex-gemini-copilot-claude.md | 24 | medium | 2026-03-02 |
| Kiro Supervised Autopilot | kiro-supervised-autopilot.md | - | deep | 2026-03-02 |
| Git History Analysis in Developer Tools | git-history-analysis-developer-tools.md | 24 | medium | 2026-03-14 |
| AI Agent Commits & Git Analysis Impact | ai-agent-commits-git-analysis-impact.md | 25 | medium | 2026-03-14 |
| AI Commit Detection Forensics | ai-commit-detection-forensics.md | 42 | deep | 2026-03-14 |
| glide-mq (Message Queue on Valkey/Redis) | glide-mq.md | 12 | medium | 2026-03-19 |
| Skills for Library/SDK - Best Practices | skills-for-library-and-sdk-best-practices.md | 16 | medium | 2026-03-19 |
| RAG Skill Indexing for AI Agent Knowledge | rag-skill-indexing.md | 18 | deep | 2026-03-31 |
| Codex CLI Plugin Manifest System | codex-plugin-manifest.md | 42 | deep | 2026-04-01 |
| RESP MQ Server in Rust (Wire Protocol, Lua, Queue DS) | resp-mq-server-rust.md | 42 | deep | 2026-04-03 |

## Trigger Phrases

Use this knowledge when user asks about:
- "RESP protocol Rust" -> resp-mq-server-rust.md
- "Redis server Rust" -> resp-mq-server-rust.md
- "RESP wire protocol" -> resp-mq-server-rust.md
- "Redis compatible server" -> resp-mq-server-rust.md
- "message queue server Rust" -> resp-mq-server-rust.md
- "BullMQ Lua scripts" -> resp-mq-server-rust.md
- "EVAL EVALSHA implementation" -> resp-mq-server-rust.md
- "Lua embedding Rust mlua" -> resp-mq-server-rust.md
- "redis.call Lua Rust" -> resp-mq-server-rust.md
- "mini-redis architecture" -> resp-mq-server-rust.md
- "DragonflyDB architecture" -> resp-mq-server-rust.md
- "Garnet Redis replacement" -> resp-mq-server-rust.md
- "Sidekiq Redis commands" -> resp-mq-server-rust.md
- "Celery Redis transport" -> resp-mq-server-rust.md
- "queue data structures Rust" -> resp-mq-server-rust.md
- "skip list sorted set" -> resp-mq-server-rust.md
- "time wheel delayed jobs" -> resp-mq-server-rust.md
- "BRPOP LPUSH implementation" -> resp-mq-server-rust.md
- "Redis AOF WAL Rust" -> resp-mq-server-rust.md
- "Redis command subset MQ" -> resp-mq-server-rust.md
- "Tokio TCP server RESP" -> resp-mq-server-rust.md
- "shared-nothing Redis Rust" -> resp-mq-server-rust.md
- "Redis replacement single binary" -> resp-mq-server-rust.md
- "BullMQ compatibility server" -> resp-mq-server-rust.md
- "Sidekiq compatibility server" -> resp-mq-server-rust.md
- "Celery compatibility server" -> resp-mq-server-rust.md
- "glide-mq server backend" -> resp-mq-server-rust.md
- "redis-protocol crate" -> resp-mq-server-rust.md
- "RESP2 RESP3 parsing" -> resp-mq-server-rust.md
- "Redis Lua scripting Rust" -> resp-mq-server-rust.md
- "queue persistence AOF" -> resp-mq-server-rust.md
- "Redis benchmark methodology" -> resp-mq-server-rust.md
- "io_uring Rust server" -> resp-mq-server-rust.md
- "Kvrocks architecture" -> resp-mq-server-rust.md

## Quick Lookup

| Keyword | Guide |
|---------|-------|
| RESP, RESP2, RESP3, wire protocol | resp-mq-server-rust.md |
| Redis server Rust, Redis replacement Rust | resp-mq-server-rust.md |
| mini-redis, tokio redis server | resp-mq-server-rust.md |
| BullMQ Lua scripts, EVAL EVALSHA | resp-mq-server-rust.md |
| mlua, rlua, Lua embedding Rust | resp-mq-server-rust.md |
| redis.call, redis.pcall, Lua API | resp-mq-server-rust.md |
| DragonflyDB, Garnet, KeyDB, Kvrocks | resp-mq-server-rust.md |
| Sidekiq Redis, Celery Redis, Kombu | resp-mq-server-rust.md |
| queue data structures, skip list, time wheel | resp-mq-server-rust.md |
| BRPOP, RPOPLPUSH, blocking operations | resp-mq-server-rust.md |
| AOF, WAL, persistence queue | resp-mq-server-rust.md |
| redis-benchmark, memtier_benchmark | resp-mq-server-rust.md |
| shared-nothing, thread-per-core, sharding | resp-mq-server-rust.md |
| io_uring, tokio-uring, glommio | resp-mq-server-rust.md |
| redis-protocol crate, RESP parsing | resp-mq-server-rust.md |
| TLS RESP, tokio-rustls server | resp-mq-server-rust.md |
| DashMap, crossbeam-skiplist, parking_lot | resp-mq-server-rust.md |
| bumpalo arena, slab allocator, bytes crate | resp-mq-server-rust.md |
| MULTI EXEC, Redis transactions | resp-mq-server-rust.md |
| Redis Pub/Sub, SUBSCRIBE PUBLISH | resp-mq-server-rust.md |
| Redis streams, XADD XREADGROUP | resp-mq-server-rust.md |

## How to Use

1. Check if user question matches a topic
2. Read the relevant guide file
3. Answer based on synthesized knowledge
4. Cite the guide if user asks for sources
