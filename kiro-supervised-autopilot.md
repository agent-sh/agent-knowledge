# Learning Guide: Kiro Supervised Mode, Autopilot Mode, and Approval Workflows

**Generated**: 2026-03-02
**Sources**: 6 resources analyzed
**Depth**: brief

---

## Prerequisites

- Familiarity with Kiro IDE or CLI (see kiro-cli-agentic-files-settings.md for overview)
- Understanding of AI-assisted code generation concepts
- Knowledge of version control and code diffs
- Basic understanding of file editing workflows

## TL;DR

- **Autopilot Mode (default)**: Kiro executes tasks autonomously without approval; you can view diffs, revert changes, or interrupt, but edits proceed immediately
- **Supervised Mode**: Kiro pauses after each turn with file edits, presenting changes as reviewable hunks; you accept/reject/discuss individual sections before proceeding
- **File Edit Approval**: Presented as granular hunks per file; you can accept all, reject all, or approve specific sections
- **Command Execution**: Treated separately from file edits; commands run without approval in both modes but can be viewed in execution logs
- **Mode Switching**: Can switch between modes mid-session via IDE settings or CLI session context
- **Agent Behavior**: Mode choice affects how agents approach task decomposition and decision-making; supervised mode encourages smaller, reviewable steps

---

## Core Concepts

### Autopilot Mode (Default Behavior)

Autopilot is Kiro's default operating mode, designed for speed and autonomy.

**Characteristics:**
- Agent executes tasks end-to-end without requesting approval
- Creates and modifies files across multiple locations in the codebase
- Runs shell commands as needed
- Makes architectural decisions and trade-offs independently
- Completes the workflow in a single or minimal number of turns

**User Control Points:**
- View diffs before changes are written to disk (read-only inspection)
- Revert all changes with one command if needed
- Interrupt execution mid-task
- Modify the task prompt and restart

**When Code is Modified:**
Files are written to disk immediately upon agent completion. There is no intermediate approval gate; the agent's turn completes and changes persist.

**Agent Reasoning:**
In autopilot mode, agents typically plan comprehensive solutions and execute them with confidence, knowing they can iterate if issues arise. This leads to faster task completion but less granular control over intermediate decisions.

*Source: kiro.dev/docs/chat/autopilot*

### Supervised Mode (Approval-Required Behavior)

Supervised mode requires human review and approval for any file edits before they become permanent.

**Characteristics:**
- Agent completes a turn (thinking, planning, proposing edits)
- Yields control to the user with proposed file changes
- Changes are presented as individual hunks (contiguous blocks of edits within a file)
- User reviews and approves/rejects changes before persistence
- Agent does not proceed until explicit approval or rejection

**User Actions:**
- Accept all changes for the turn
- Reject all changes for the turn
- Accept individual hunks within a file
- Reject individual hunks within a file
- Request discussion or clarification on specific changes
- Modify a rejection reason and ask agent to revise

**File Edit Flow:**
```
Agent proposes edits → Kiro pauses → User reviews hunks →
  [Accept/Reject per hunk] → Changes persist (if approved) →
  Agent continues or cycle repeats
```

**Command Execution in Supervised Mode:**
Commands (shell invocations) are still executed in supervised mode; they are not subject to approval. However, command output is logged and visible in the execution history, allowing the user to understand what the agent did.

**Agent Reasoning:**
In supervised mode, agents tend to approach problems more cautiously, proposing smaller, reviewable chunks of work. They may break a task into more steps, making their reasoning transparent and easier to verify.

*Source: kiro.dev/docs/chat*

---

## File Edit Approval Details

### Hunk-Based Review

File edits in supervised mode are grouped into **hunks** - contiguous blocks of changes within a single file.

**Hunk Structure:**
- File path and filename
- Line range affected (e.g., lines 45-62)
- Context lines (surrounding unchanged code for orientation)
- Proposed changes (insertions, deletions, modifications)
- Diff format (similar to `git diff` or unified diff format)

**Hunk Approval Options:**
1. **Accept this hunk** - Approve the specific change block
2. **Reject this hunk** - Decline the change; propose alternative in next turn
3. **Modify and discuss** - Ask agent to adjust the hunk before approval
4. **Accept all remaining** - Approve all hunks in this turn
5. **Reject all remaining** - Decline all hunks in this turn

### Per-File Review Workflow

Kiro presents changes organized by file:

```
File: src/api/routes.ts
  ├─ Hunk 1 (lines 12-28): Add new route handler
  │   [Accept] [Reject] [Discuss]
  ├─ Hunk 2 (lines 45-51): Update imports
  │   [Accept] [Reject] [Discuss]
  └─ Hunk 3 (lines 89-94): Fix type annotations
      [Accept] [Reject] [Discuss]

File: src/api/types.ts
  └─ Hunk 1 (lines 3-15): Define new interface
      [Accept] [Reject] [Discuss]
```

### Partial Approval

You are not required to approve all hunks in a turn. Mixed approval is supported:

- Accept hunks 1 and 3 of file A
- Reject hunk 2 of file A
- Accept all hunks of file B

Rejected hunks are not written to disk. The agent can then be asked to revise them in the next turn.

### Acceptance Persistence

Once a hunk is accepted, it is immediately written to the target file. Rejected hunks do not affect the file. The agent sees the rejection and can propose a different approach in its next response.

*Source: kiro.dev/docs/chat, synthesized from supervised mode documentation*

---

## Command Execution Approval

### Shell Command Execution

Commands executed via the `execute_bash` or `execute_shell` tool are not subject to approval in either mode (autopilot or supervised).

**Execution Behavior:**
- Agent decides when to run commands
- Commands execute immediately when invoked
- Output is captured and shown to the user
- No pre-execution approval is required

**Rationale:**
Commands are typically read-only operations (file listing, dependency checks, test runs) or setup steps (installing dependencies, creating directories). Blocking on every command would create excessive friction. Dangerous operations (deletes, force pushes) are typically not attempted by agents without explicit user direction.

### Logging and Observability

Command execution details are logged in the session history:

- **Command text** - Exact shell command executed
- **Exit code** - Return status (0 = success, non-zero = failure)
- **STDOUT** - Standard output (typically long)
- **STDERR** - Standard error output (if any)
- **Duration** - How long the command took

Users can review these logs to understand what the agent did and catch unintended side effects (e.g., a command that created files outside the intended directory).

### Command Hooks (Advanced)

If you need to intercept or block certain commands, you can use Kiro Hooks with `PreToolUse` event type to validate commands before execution. A hook can return exit code 2 to block the tool invocation.

*Source: kiro.dev/docs/chat, kiro.dev/docs/hooks*

---

## Switching Modes Mid-Session

### IDE Mode Switching

In the Kiro IDE, you can switch between autopilot and supervised modes during a session.

**Steps:**
1. Open Settings (`Cmd+,` on Mac / `Ctrl+,` on Windows/Linux)
2. Search for "Autopilot" or "Approval"
3. Toggle the **Autopilot Toggle** setting
4. The change takes effect immediately for the next agent response

**Persistence:**
The mode selection is stored in IDE settings and persists across sessions.

### CLI Mode Switching

The Kiro CLI does not have built-in supervised mode toggle in the same way as the IDE. However, you can:

1. **Start a new session** in the opposite mode (manual restart)
2. **Use hooks** to add approval-like behavior for specific tools
3. **Use custom agents** with different configurations

For CLI users wanting approval workflows, the recommended approach is to use hooks that validate tool invocations before execution.

### Mid-Session Behavior

If you switch from **autopilot to supervised** mid-session:
- The next agent turn will pause for approval instead of auto-executing
- Previous turns' changes remain in place (already persisted)
- You start seeing approval prompts for hunks going forward

If you switch from **supervised to autopilot** mid-session:
- The next agent turn will auto-execute without pausing for approval
- No retroactive approval is requested for prior edits
- You lose granular control for subsequent turns

### Recommendation

Most users do not switch modes mid-session because they have different workflows:
- **Autopilot**: For prototyping, familiar codebases, rapid iteration
- **Supervised**: For learning, critical systems, code review workflows

Switching reflects a change in your comfort level with the agent's decisions.

*Source: kiro.dev/docs/chat, kiro.dev/docs/getting-started*

---

## How Modes Affect Agent Behavior

### Autopilot Agent Strategy

In autopilot mode, agents:

- **Plan comprehensively** - Think through multi-step solutions before acting
- **Execute decisively** - Make design choices without hesitation
- **Batch operations** - Group related changes to minimize turns
- **Assume trust** - Proceed with confidence that you'll catch issues in review
- **Iterate externally** - If a turn doesn't work, iterate based on user feedback in the next turn

**Example Flow (Autopilot):**
```
User: "Add user authentication to the API"
  ↓
Agent [Turn 1]:
  - Analyzes codebase architecture
  - Designs auth strategy
  - Implements middleware, routes, types
  - Updates package.json
  - Writes all changes to disk
  - Reports completion
  ↓
User: Views diffs, runs tests, decides if satisfied
```

### Supervised Agent Strategy

In supervised mode, agents:

- **Break down tasks** - Decompose into smaller, reviewable steps
- **Explain reasoning** - Provide context for each change
- **Request micro-approvals** - After each turn, pause for review
- **Adapt to feedback** - Respond to rejected hunks with revisions
- **Build incrementally** - Let user guide direction through approvals

**Example Flow (Supervised):**
```
User: "Add user authentication to the API"
  ↓
Agent [Turn 1]:
  - Proposes auth middleware structure
  - Shows hunk for new middleware file
  ↓
User: Reviews, accepts hunk
  ↓
Agent [Turn 2]:
  - Proposes route definitions
  - Shows hunks for new routes
  ↓
User: Reviews, accepts some hunks, rejects others
  ↓
Agent [Turn 3]:
  - Revises rejected hunks based on feedback
  - Shows updated route definitions
  ↓
[Cycle continues until complete...]
```

### Tool Usage Differences

**Autopilot Mode:**
- Uses more tools per turn
- Chains operations (read, analyze, write, test)
- Makes independent decisions about which files to modify

**Supervised Mode:**
- Typically uses fewer tools per turn
- Focuses on single concerns (e.g., "update this file type")
- Waits for user approval before deciding next steps

### Architectural Decision-Making

**Autopilot:**
- Agent might choose between multiple architectural patterns and implement the one it judges best
- Trade-offs are made implicitly (e.g., choosing Zod over Joi for validation)

**Supervised:**
- Agent might ask for direction on architectural choices
- Presents alternatives and waits for user input
- Reduces risk of committing to the wrong approach

*Source: synthesized from kiro.dev/docs/chat, mode documentation, and user workflow guides*

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Not realizing changes auto-persist in autopilot | Expectation of approval before changes are written | Use supervised mode if you want review before persistence; always check diffs before agent finishes |
| Command execution surprises | Commands run without approval; unexpected side effects occur | Review command output carefully; use hooks to validate dangerous commands; keep commands simple |
| Mode confusion between IDE and CLI | IDE has clear toggle; CLI doesn't expose mode selection | For CLI, understand that supervised mode requires hooks or external tooling; use IDE for approval workflows |
| Partial approval deadlock | User rejects all hunks, agent re-proposes identical changes | Provide clear feedback on why hunks were rejected; ask agent to revise approach, not just the code |
| Missing mid-session mode switch awareness | User switches to supervised mode expecting retrospective approval | Understand that retroactive approval isn't possible; switching only affects future turns |
| Approval workflow slowing down development | Over-reliance on hunk-by-hunk approval for simple changes | Use autopilot for trusted tasks; reserve supervised mode for high-risk changes |

---

## Best Practices

### Choosing Modes

1. **Start in supervised mode if:**
   - You're learning Kiro
   - Working on unfamiliar or critical code
   - Building features with undefined requirements
   - Team code review is a requirement

2. **Use autopilot mode if:**
   - You have high confidence in the agent
   - Working on a familiar codebase
   - Speed is the priority
   - Changes are low-risk (new feature, not core logic)

3. **Hybrid approach:**
   - Use supervised mode for architectural decisions
   - Switch to autopilot for implementation details once direction is set
   - Use supervised mode again for integration and testing phases

### File Approval Workflow

4. **Always review hunks in context:**
   - Don't accept hunks without reading the surrounding code
   - Use the diff view to understand the change
   - Spot-check for subtle bugs (off-by-one errors, missing nullchecks)

5. **Provide clear rejection feedback:**
   - Don't just reject; explain why (e.g., "This violates our naming convention" vs just "No")
   - Suggest the alternative you want
   - Reference your project's steering files if available

6. **Use Accept All selectively:**
   - Only for low-risk changes or when you've already approved similar hunks
   - Prefer granular approval on first exposure to agent's code

### Agent Behavior Optimization

7. **Prime the agent for supervised mode:**
   - When starting supervised session, mention it explicitly: "Review carefully before proceeding"
   - Break down complex requirements into phases

8. **Understand command context:**
   - Commands are not subject to approval; log them for your review
   - If a command fails, the agent will attempt recovery
   - Dangerous operations should be scoped in steering files

9. **Combine modes with steering:**
   - Use `.kiro/steering/` files to define approval criteria (e.g., "all API changes require explicit approval")
   - Supervised mode respects steering; agent will pause at critical areas

*Source: synthesized from kiro.dev documentation and best practices guides*

---

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [kiro.dev/docs/chat](https://kiro.dev/docs/chat/) | Official Docs | Comprehensive chat modes documentation |
| [kiro.dev/docs/chat/autopilot](https://kiro.dev/docs/chat/autopilot/) | Official Docs | Autopilot mode details and controls |
| [kiro.dev/docs/chat/vibe](https://kiro.dev/docs/chat/vibe/) | Official Docs | Vibe mode (creative mode) paired with autopilot/supervised |
| [kiro.dev/docs/steering](https://kiro.dev/docs/steering/) | Official Docs | Steering files for guiding agent behavior |
| [kiro.dev/docs/hooks](https://kiro.dev/docs/hooks/) | Official Docs | Hooks for approval-like workflows in CLI |
| [kiro.dev/docs/specs](https://kiro.dev/docs/specs/) | Official Docs | Specs as complement to supervised mode workflows |
| [Kiro IDE Getting Started](https://kiro.dev/docs/getting-started/first-project/) | Tutorial | Hands-on walkthrough using IDE modes |

---

*This guide was synthesized from 6 primary sources including official Kiro documentation and architectural references. See `resources/kiro-supervised-autopilot-sources.json` for the full source list with quality scores.*
