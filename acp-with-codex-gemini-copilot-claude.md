# Learning Guide: ACP (Agent Communication Protocol) with Codex, Gemini, Copilot, and Claude

**Generated**: 2026-03-02
**Sources**: 24 resources analyzed
**Depth**: medium

---

## Prerequisites

- Familiarity with at least one AI coding tool (Claude Code, Codex CLI, Gemini CLI, or GitHub Copilot)
- Basic understanding of HTTP, REST APIs, and JSON-RPC
- General awareness of the Model Context Protocol (MCP)
- Understanding of client-server architecture and inter-process communication

## TL;DR

- **ACP** (Agent Communication Protocol) is an open protocol under the Linux Foundation for agent interoperability, using REST endpoints and MIME types. It originated from IBM's BeeAI project and is now merging with Google's **A2A** protocol.
- **A2A** (Agent-to-Agent) is Google's open protocol using JSON-RPC 2.0 over HTTP(S) with Agent Cards for discovery. It focuses on opaque agent-to-agent collaboration. Over 50 technology partners support it.
- **MCP** (Model Context Protocol) is Anthropic's open standard for connecting AI to tools and data -- it is complementary to A2A/ACP, not a competitor. MCP handles agent-to-tool communication; A2A handles agent-to-agent communication.
- **None of the major AI coding CLIs** (Claude Code, Codex CLI, Gemini CLI, Copilot) natively implement ACP or A2A as a built-in protocol. All four support MCP to varying degrees.
- **Cross-tool AI communication** is possible today via MCP bridges, the Claude Agent SDK, and Claude Code's `mcp serve` feature, but formal agent-to-agent protocols between different vendor tools remain nascent.

---

## Core Concepts

### 1. What is ACP (Agent Communication Protocol)?

ACP is an open protocol for agent interoperability that addresses the fragmentation challenge in AI development. It enables standardized communication across different agent frameworks and implementations.

**Key characteristics:**
- Uses simple, well-defined **REST endpoints** aligned with standard HTTP patterns
- Employs **MIME types** for content identification, enabling extensibility for any data format
- Supports synchronous, asynchronous, and streaming interactions
- Supports both stateful and stateless patterns
- Provides online and offline agent discovery
- Framework-agnostic: works across BeeAI, LangChain, CrewAI, and custom implementations

**Origin and governance:**
- Originally developed by IBM as part of the **BeeAI** project (now called "Agent Stack")
- Contributed to the **Linux Foundation AI & Data** program
- SDKs available in **Python** (`pip install acp-sdk`) and **TypeScript**
- Reference implementation: Agent Stack (formerly BeeAI), available at github.com/i-am-bee/beeai

**ACP SDK example (Python):**

```python
from acp_sdk import Server, Client, Message, MessagePart

# Server side
server = Server()

@server.agent()
async def echo(input: list[Message]):
    """Echoes everything"""
    for message in input:
        yield message

server.run(port=8000)

# Client side
async with Client(base_url="http://localhost:8000") as client:
    run = await client.run_sync(
        agent="echo",
        input=[Message(parts=[MessagePart(content="Hello!")])]
    )
    print(run)
```

### 2. What is Google's A2A (Agent-to-Agent) Protocol?

A2A is an open protocol contributed by Google to the Linux Foundation, enabling communication and interoperability between opaque agentic applications.

**Key characteristics:**
- Uses **JSON-RPC 2.0 over HTTP(S)** as its transport layer
- Supports synchronous request/response, Server-Sent Events (SSE) streaming, and asynchronous push notifications
- Data formats: text, files, and structured JSON
- Designed for **opaque** agents -- no requirement to share internal state, memory, or tool implementations

**Core concepts:**
- **Agent Cards**: JSON documents that describe an agent's capabilities, connection info, and supported modalities. Used for discovery.
- **Tasks**: Units of work with a defined lifecycle, producing artifacts as outputs.
- **Messages**: Structured communication between client and remote agents.
- **Opacity principle**: Agents collaborate without exposing internal memory, proprietary logic, or specific tool implementations.

**Announced**: April 9, 2025 by Google. Over 50 technology partners including Atlassian, Salesforce, SAP, ServiceNow, Langchain, MongoDB, and many major consulting firms.

**SDKs**: Python, Go, JavaScript, Java, .NET (available via `pip install a2a-sdk` for Python).

**Licensed**: Apache 2.0 under the Linux Foundation.

### 3. ACP vs A2A: Convergence

ACP and A2A are converging under the Linux Foundation. As of early 2026:
- ACP's website (agentcommunicationprotocol.dev) describes ACP as "now part of A2A"
- Agent Stack (BeeAI) has adopted A2A as its interoperability protocol
- Both protocols share similar goals but had different transport choices (REST for ACP, JSON-RPC for A2A)
- The merged effort means new implementations should target the A2A specification

### 4. MCP (Model Context Protocol) -- The Complement

MCP is Anthropic's open standard for connecting AI applications to external systems (tools, data sources, workflows). It is explicitly **not** an agent-to-agent protocol.

**Key distinction:**
- **MCP**: Agent-to-tool communication (connecting AI to databases, APIs, file systems)
- **A2A/ACP**: Agent-to-agent communication (connecting AI agents to each other)

As Google's A2A announcement stated: "A2A complements Anthropic's Model Context Protocol (MCP), which provides helpful tools and context to agents."

**MCP architecture:**
- Client-server model with stdio, SSE, and HTTP transports
- Servers expose tools, resources, and prompts
- Clients (AI applications) consume these capabilities
- Supported by Claude Code, Codex CLI, Gemini CLI, Copilot, Kiro, Cursor, and many others

### 5. Protocol Comparison

| Feature | ACP | A2A | MCP |
|---------|-----|-----|-----|
| **Purpose** | Agent-to-agent interop | Agent-to-agent interop | Agent-to-tool integration |
| **Transport** | REST/HTTP | JSON-RPC 2.0 over HTTP(S) | stdio, SSE, HTTP |
| **Discovery** | Agent registry | Agent Cards (JSON) | Server configuration |
| **Streaming** | Yes | Yes (SSE) | Yes (SSE, streamable HTTP) |
| **Async** | Yes | Yes (push notifications) | Limited |
| **Creator** | IBM / Linux Foundation | Google / Linux Foundation | Anthropic |
| **Status** | Merging into A2A | Active, growing ecosystem | Mature, widely adopted |
| **Opacity** | Framework-agnostic | Fully opaque agents | N/A (tool exposure) |
| **SDKs** | Python, TypeScript | Python, Go, JS, Java, .NET | Python, TypeScript, + more |
| **When to use** | Legacy ACP deployments | Agent orchestration across vendors | Connecting AI to tools/data |

---

## How Each AI Coding Tool Relates to ACP/A2A

### Claude Code (Anthropic)

**ACP/A2A support**: No native ACP or A2A protocol implementation.

**What it does support:**
- **MCP** (full support): Claude Code can both consume MCP servers and act as one (`claude mcp serve`)
- **Subagents**: Specialized sub-agents within a single Claude Code session (Explore, Plan, custom agents)
- **Agent Teams** (experimental): Multiple Claude Code instances coordinating via shared task lists, mailbox messaging, and direct inter-agent communication
- **Agent SDK**: Python and TypeScript SDKs for building production agents with Claude Code's tools programmatically

**Cross-tool bridging:**
- Claude Code can serve as an MCP server, letting other tools invoke Claude's capabilities:
  ```bash
  claude mcp serve
  ```
- The Agent SDK enables programmatic spawning of Claude agents that can interact with MCP servers from any provider
- MCP bridge servers (community-built) can connect Claude to other AI model APIs

**Agent Teams architecture:**
- Team lead coordinates work via shared task list
- Teammates communicate via a mailbox messaging system
- Each teammate has its own context window
- Supports plan approval workflows
- Experimental; requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

### OpenAI Codex CLI

**ACP/A2A support**: No native ACP or A2A protocol implementation.

**What it does support:**
- **MCP**: Basic MCP support via `shell-tool-mcp` component and MCP Registry integration
- **Multi-provider**: Configurable to work with various AI providers (OpenAI, Azure, Gemini, Ollama, etc.)
- **AGENTS.md**: Supports instruction files that merge from global to directory-level context

**Architecture:**
- Primarily Rust-based (96.1%), operates as a single-agent tool
- Three approval modes: Suggest, Auto Edit, Full Auto
- Sandboxed execution using Apple Seatbelt (macOS) or Docker (Linux)
- No multi-agent coordination or agent-to-agent communication mechanisms documented

**Cross-tool bridging:**
- Can be configured to use different LLM providers, but this is provider-switching, not agent-to-agent communication
- MCP server support enables external tool integration

### Google Gemini CLI

**ACP/A2A support**: No native A2A protocol implementation despite Google creating A2A.

**What it does support:**
- **MCP**: Explicit MCP support for custom integrations
- **Google Search grounding**: Built-in real-time information access
- **GitHub integration**: Dedicated GitHub Action for automated workflows
- **Non-interactive mode**: JSON and streaming output for automation and scripting

**Notable absence:**
Gemini CLI documentation contains no references to A2A or ACP support. This is notable because Google created the A2A protocol. The A2A protocol appears designed for enterprise agent orchestration scenarios rather than developer CLI tools.

### GitHub Copilot

**ACP/A2A support**: No native ACP or A2A protocol implementation.

**What it does support:**
- **MCP**: Full MCP support through Copilot Extensions in VS Code, JetBrains, and other IDEs
- **GitHub MCP Server**: Dedicated server for code tasks, coding agent invocation, and code scanning
- **Extensions ecosystem**: Third-party tool integrations via MCP
- **Push protection**: Security features for AI-generated code

**Organization controls:**
- MCP server policy management for Copilot Business/Enterprise
- Allowlist/denylist controls for MCP servers

### Kiro (AWS)

**ACP/A2A support**: No documented ACP or A2A support.

**What it does support:**
- **MCP**: Full MCP server support with configuration UI
- **Agentic features**: Specs, hooks, steering, and agentic chat
- Built on Claude (AWS product)
- MCP server management with connection status indicators

---

## Cross-AI-Tool Communication Patterns

### Pattern 1: MCP Bridge Servers

Community-built MCP servers that bridge to other AI providers:

| Server | Description | Source |
|--------|-------------|--------|
| `mcp-server-gemini-bridge` | Bridge to Google Gemini API (Pro, Flash) | jaspertvdm (GitHub) |
| `mcp-server-openai-bridge` | Bridge to OpenAI API (GPT-4, GPT-4o) | jaspertvdm (GitHub) |
| `mcp-server-ollama-bridge` | Bridge to local Ollama LLM server | jaspertvdm (GitHub) |
| `Grok-MCP` | Access to xAI's Grok API | merterbak (GitHub) |
| `blockrun-mcp` | Access 30+ AI models without API keys | blockrunai (GitHub) |

**Example: Using Claude Code to consult Gemini via MCP bridge:**
```bash
# Add Gemini bridge as MCP server in Claude Code
claude mcp add --transport stdio gemini-bridge -- npx -y mcp-server-gemini-bridge

# Then in Claude Code, the Gemini bridge tools become available
# Claude can call Gemini for a second opinion on code
```

### Pattern 2: Agent-to-Agent MCP Servers

Dedicated MCP servers for agent discovery and communication:

| Server | Description |
|--------|-------------|
| `agentnet` | Agent-to-agent referral network for AI agent discovery and recommendation |
| `prolink` | Agent-to-agent marketplace middleware with MCP-native discovery and negotiation |
| `hashnet-mcp-js` | Registry Broker for discovering, registering, and chatting with AI agents |
| `agenium` | Network protocol enabling agent discovery via `agent://` URIs with mTLS trust |

### Pattern 3: Claude Code as MCP Server

Claude Code can expose its tools as an MCP server for other applications:

```bash
# Start Claude Code as an MCP server
claude mcp serve
```

**Configuration for Claude Desktop:**
```json
{
  "mcpServers": {
    "claude-code": {
      "type": "stdio",
      "command": "claude",
      "args": ["mcp", "serve"],
      "env": {}
    }
  }
}
```

This means any MCP client (Kiro, Cursor, Gemini CLI, etc.) could potentially use Claude Code's tools.

### Pattern 4: Claude Agent SDK for Programmatic Orchestration

The Claude Agent SDK enables building custom multi-agent systems:

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def main():
    async for message in query(
        prompt="Use the researcher agent to analyze this codebase",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Task"],
            agents={
                "researcher": AgentDefinition(
                    description="Research specialist for code analysis",
                    prompt="Analyze code quality and architecture patterns.",
                    tools=["Read", "Glob", "Grep"],
                )
            },
            mcp_servers={
                "gemini": {"command": "npx", "args": ["mcp-server-gemini-bridge"]}
            }
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

### Pattern 5: Sequential Thinking MCP Server

The Sequential Thinking MCP server enables structured reasoning across tool calls:

```json
{
  "mcpServers": {
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
    }
  }
}
```

**How it works:**
- Decomposes problems into sequential reasoning steps
- Supports revision of previous thoughts
- Enables branching into alternative reasoning paths
- Dynamically adjusts the total thought count
- Useful for complex problems where the full scope is not initially clear

**Parameters:**
- `thought` (string): Current reasoning step
- `nextThoughtNeeded` (boolean): Whether continuation is needed
- `thoughtNumber` / `totalThoughts` (integer): Position tracking
- `isRevision` / `revisesThought`: For reconsidering previous thinking
- `branchFromThought` / `branchId`: For alternative reasoning paths

This is not cross-AI communication per se, but enables more sophisticated reasoning within a single AI tool.

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Confusing ACP with A2A | They were separate protocols now merging | Target A2A for new implementations; ACP is being absorbed |
| Expecting AI CLIs to support A2A natively | A2A is designed for enterprise agent orchestration, not developer CLI tools | Use MCP bridges for cross-tool communication today |
| Treating MCP as an agent-to-agent protocol | MCP is for tool integration, not agent collaboration | Use A2A/ACP for agent-to-agent; MCP for agent-to-tool |
| Assuming Gemini CLI supports A2A | Google created A2A but hasn't built it into Gemini CLI | Gemini CLI uses MCP; A2A is separate infrastructure |
| Using MCP bridges in production without security review | Community MCP bridges are educational examples | Implement proper authentication, rate limiting, and input validation |
| Over-engineering with agent teams | Agent teams consume significantly more tokens | Use subagents for focused tasks; teams only when inter-agent coordination is needed |
| Expecting ACP SDK to be stable | ACP is merging into A2A | Monitor Linux Foundation announcements for migration guidance |

---

## Best Practices

1. **Use MCP as the universal adapter layer**: All major AI coding tools support MCP. Build MCP servers to create cross-tool capabilities. (Sources: MCP docs, Claude Code docs, Gemini CLI docs, Codex CLI docs)

2. **Target A2A for new agent-to-agent work**: ACP is merging into A2A. New implementations should use the A2A SDK. (Source: agentcommunicationprotocol.dev, A2A GitHub)

3. **Use Claude Code's `mcp serve` for lightweight bridging**: Expose Claude's tools to any MCP-compatible client without building custom infrastructure. (Source: Claude Code MCP documentation)

4. **Leverage the Claude Agent SDK for complex orchestration**: When you need programmatic control over multi-agent workflows with MCP integration. (Source: Claude Agent SDK overview)

5. **Start with subagents before agent teams**: Subagents have lower token costs and simpler coordination. Use teams only when agents need to communicate with each other. (Source: Claude Code subagents and agent teams docs)

6. **Keep cross-AI bridges stateless**: MCP bridge servers should not maintain conversation state across calls. Each invocation should be self-contained. (Source: MCP server best practices)

7. **Use Agent Stack (BeeAI) for A2A reference implementations**: Agent Stack is the official reference implementation for A2A-compatible agent deployment. (Source: Agent Stack GitHub)

8. **Monitor the A2A specification evolution**: The protocol is actively evolving with new features planned for Agent Card authorization, credential management, and enhanced discovery. (Source: A2A GitHub repository)

---

## Current Maturity Assessment

| Aspect | Maturity Level | Notes |
|--------|---------------|-------|
| **MCP ecosystem** | Mature | Hundreds of servers, supported by all major tools |
| **A2A specification** | Growing | 50+ partners, SDKs in 5 languages, under Linux Foundation |
| **ACP specification** | Transitioning | Merging into A2A; existing ACP deployments should plan migration |
| **Cross-CLI agent communication** | Nascent | No AI CLI natively implements A2A; MCP bridges are community-built |
| **Claude Code agent teams** | Experimental | Functional but with known limitations; disabled by default |
| **MCP bridge servers** | Early | Community-built, not production-hardened |
| **Agent Stack (BeeAI) production readiness** | Developing | 1000+ GitHub stars, modular architecture, but still evolving |
| **Sequential Thinking MCP** | Stable | Part of official MCP reference servers |

---

## Real-World Cross-AI Communication Examples

### Example 1: Claude Code consulting Gemini via MCP bridge
```bash
# Setup
claude mcp add --transport stdio gemini -- npx -y mcp-server-gemini-bridge

# In Claude Code session:
# "Get Gemini's perspective on the best approach for this database migration"
# Claude calls the Gemini MCP bridge tool, gets Gemini's response, synthesizes both perspectives
```

### Example 2: Multi-agent research with Claude Agent SDK
```python
# A lead agent delegates research to subagents, each with different MCP servers
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def multi_source_research():
    async for msg in query(
        prompt="Research best practices for API rate limiting. "
               "Use the researcher agent to analyze our codebase "
               "and the web-researcher to find current best practices.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Task", "WebSearch"],
            agents={
                "researcher": AgentDefinition(
                    description="Codebase analysis specialist",
                    prompt="Analyze the existing codebase for patterns.",
                    tools=["Read", "Glob", "Grep"],
                ),
                "web-researcher": AgentDefinition(
                    description="Web research specialist",
                    prompt="Search the web for current best practices.",
                    tools=["WebSearch", "WebFetch"],
                )
            }
        ),
    ):
        if hasattr(msg, "result"):
            print(msg.result)
```

### Example 3: A2A-compatible agent via Agent Stack
```python
# Deploy an agent that is automatically A2A-compatible
# Using Agent Stack (BeeAI)

# 1. Define your agent in any framework (LangGraph, CrewAI, custom)
# 2. Deploy via Agent Stack CLI
# 3. Agent is automatically exposed as an A2A-compatible service

# Other A2A-compatible agents can discover and communicate with it
# via Agent Cards and JSON-RPC 2.0
```

### Example 4: Claude Code agent team for parallel investigation
```text
# In Claude Code with agent teams enabled:

I need to investigate why our API is slow. Create an agent team:
- One teammate to profile the database queries
- One teammate to analyze the application logs
- One teammate to check the network latency metrics
Have them share findings and build a unified diagnosis.
```

---

## Open Source Implementations

### A2A Protocol
- **A2A SDKs**: Python, Go, JavaScript, Java, .NET (github.com/google/A2A)
- **Agent Stack**: Reference A2A implementation (github.com/i-am-bee/beeai)
- **DeepLearning.AI Course**: Official course on building A2A-compliant agents

### ACP Protocol (Legacy/Transitioning)
- **ACP SDK (Python)**: `pip install acp-sdk` -- v1.0.3, maintained by IBM (pypi.org/project/acp-sdk)
- **ACP SDK (TypeScript)**: Available via npm

### MCP Ecosystem
- **Official MCP servers**: github.com/modelcontextprotocol/servers
- **Sequential Thinking**: `@modelcontextprotocol/server-sequential-thinking`
- **AI Bridge servers**: Community-built bridges to Gemini, OpenAI, Ollama, Grok

### Claude Agent SDK
- **Python**: `pip install claude-agent-sdk` (github.com/anthropics/claude-agent-sdk-python)
- **TypeScript**: `npm install @anthropic-ai/claude-agent-sdk`
- **Demo agents**: github.com/anthropics/claude-agent-sdk-demos

---

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [A2A Protocol Specification](https://a2a-protocol.org/) | Spec | Official A2A specification and documentation |
| [ACP Website](https://agentcommunicationprotocol.dev/) | Docs | ACP protocol overview (now merging into A2A) |
| [Agent Stack (BeeAI)](https://github.com/i-am-bee/beeai) | Code | Reference A2A implementation with full agent infrastructure |
| [MCP Introduction](https://modelcontextprotocol.io/introduction) | Docs | Official MCP documentation and architecture |
| [Claude Code MCP Guide](https://code.claude.com/docs/en/mcp) | Docs | Comprehensive guide to MCP in Claude Code |
| [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents) | Docs | Creating and managing specialized AI subagents |
| [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams) | Docs | Multi-agent coordination with shared tasks and messaging |
| [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) | Docs | Programmatic access to Claude Code's capabilities |
| [Claude Agent SDK Demos](https://github.com/anthropics/claude-agent-sdk-demos) | Code | Example agents including multi-agent research pattern |
| [Codex CLI](https://github.com/openai/codex) | Code | OpenAI's coding agent with MCP support |
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | Code | Google's terminal AI with MCP support |
| [Google A2A Blog Post](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) | Blog | Google's announcement and vision for A2A |
| [ACP SDK (Python)](https://pypi.org/project/acp-sdk/) | Package | Python SDK for building ACP agents |
| [MCP Servers Repository](https://github.com/modelcontextprotocol/servers) | Code | Official MCP reference servers including Sequential Thinking |
| [Awesome MCP Servers](https://github.com/punkpeye/awesome-mcp-servers) | List | Curated list of MCP servers including AI bridges |

---

*This guide was synthesized from 24 sources. See `resources/acp-with-codex-gemini-copilot-claude-sources.json` for full source list with quality scores.*
