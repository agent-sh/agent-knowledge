# Learning Guide: Detecting AI-Generated Commits and Code Contributions in Git History

**Generated**: 2026-03-14
**Sources**: 42 resources analyzed
**Depth**: deep

## Prerequisites

- Familiarity with git internals (commits, trailers, author vs committer, git notes, git blame)
- Understanding of conventional commits format
- Basic knowledge of AI coding tools (Copilot, Claude Code, Cursor, etc.)
- Some exposure to statistical/stylometric concepts is helpful but not required

## TL;DR

- Explicit attribution (Co-Authored-By trailers, bot author emails, branch prefixes) is the most reliable detection signal, but many tools add none by default or allow disabling it.
- The academic study "Fingerprinting AI Coding Agents on GitHub" (arXiv:2601.17406) achieved 97.2% F1-score classifying commits across 5 agents using 41 behavioral features - primarily commit message patterns, PR structure, and code characteristics.
- Commit message structure (multiline ratio, length, conventional commit usage) is the single strongest fingerprint, with 44.7% global feature importance.
- Code-level stylometric signals (comment density, conditional density, naming conventions, function size uniformity) can distinguish AI from human code with 90-98% accuracy in controlled settings, but drop significantly in real-world conditions.
- A composite scoring approach combining explicit attribution, commit metadata, message heuristics, code stylometrics, and temporal patterns provides the most reliable detection, but no single signal is sufficient alone.

## 1. Explicit Attribution Signals (High Reliability)

### 1.1 Co-Authored-By Trailers

The most direct signal. Format: `Co-Authored-By: Name <email>`

| Tool | Default Trailer | Configurable | Default State |
|------|----------------|--------------|---------------|
| Claude Code | `Co-Authored-By: Claude <noreply@anthropic.com>` | Yes (can disable) | Enabled |
| Aider | `Co-authored-by: aider (model) <noreply@aider.chat>` | Yes (can disable) | Enabled |
| Cursor | `Co-authored-by: Cursor <cursoragent@cursor.com>` | Partial | Enabled (auto-added) |
| OpenAI Codex CLI | Configurable via `command_attribution` in config.toml | Yes (can disable) | Recently added |
| Copilot Coding Agent | Co-authored-by with requesting user | No | Enabled |
| Windsurf | None | N/A | No trailer |
| Cline | None | N/A | No trailer |
| Continue | None | N/A | No trailer |
| JetBrains AI | None | N/A | No trailer |
| Tabnine | None | N/A | No trailer |
| Sourcegraph Cody | None | N/A | No trailer |

**Detection pattern**: Parse commit message trailers for `Co-Authored-By` containing known AI tool names or emails.

```
# Known AI co-author emails
noreply@anthropic.com          # Claude Code
noreply@aider.chat             # Aider
cursoragent@cursor.com         # Cursor
agent@cursor.com               # Cursor (variant)
```

**Reliability**: HIGH when present, but easily disabled or never added by many tools.

### 1.2 Commit Message Attribution Text

Some tools append plain-text attribution beyond trailers.

| Tool | Attribution Text | Position |
|------|-----------------|----------|
| Claude Code | `Generated with Claude Code` | Above trailer, after blank line |
| Aider | `aider: ` prefix (optional) | Start of commit message |
| Copilot (web) | Copilot-generated commit messages (no attribution text) | N/A |

**Detection pattern**: Search commit messages for `Generated with Claude Code`, `aider:` prefix.

### 1.3 Author/Email Identity

When the AI tool is the commit author (not just co-author).

| Tool | Author Name | Author Email | Context |
|------|------------|--------------|---------|
| Copilot Coding Agent | `GitHub Copilot` | Bot-format noreply | Autonomous agent PRs |
| Devin | `devin-ai-integration[bot]` | `bot@devin.ai` | All Devin commits |
| Google Jules | `Jules` | Jules bot account | Configurable (sole, co-author, or user) |
| Replit Agent | User's Replit name | `no-reply@replit.com` pattern | Auto-commits |
| Lovable | Lovable bot | Platform-specific | Auto-sync commits |

**Detection pattern**: Check commit author email domain against known AI tool domains.

```
# Known AI author email domains/patterns
bot@devin.ai
*@users.noreply.github.com with bot suffix
no-reply@replit.com
```

### 1.4 Branch Naming Prefixes

| Tool | Branch Prefix | Pattern | Hardcoded |
|------|--------------|---------|-----------|
| Copilot Coding Agent | `copilot/` | `copilot/<descriptive-name>` | Yes (security restriction) |
| Cursor Background Agent | `cursor/` | `cursor/<task-description>` | Yes |
| Devin | Not documented publicly | Agent-created branches | Unknown |

**Detection pattern**: Branch names starting with `copilot/` or `cursor/` strongly indicate AI agent authorship.

**Reliability**: HIGH - these prefixes are hardcoded for security and cannot be changed.

## 2. Commit Message Heuristics (Medium-High Reliability)

### 2.1 Per-Tool Message Patterns

**Claude Code** (when not customized by user rules):
- Tends toward conventional commits with emoji prefixes
- Format: `<emoji> <type>(scope): description`
- Examples: `feat(auth): add OAuth2 support`, `fix(api): handle null values`
- Often includes detailed body with bullet points
- Higher proportion of multiline messages

**Aider**:
- Conventional commits format by default
- Can be prefixed with `aider:` when configured
- Uses a weak model to generate messages from diffs
- Messages tend to be descriptive but shorter

**OpenAI Codex** (autonomous agent):
- Distinctive multiline commit messages (67.5% feature importance in fingerprinting study)
- Extensive descriptions of changes
- Most distinguishable via message structure

**Copilot Coding Agent**:
- Creates conventional-style commit messages
- PR descriptions are notably long and detailed (38.4% feature importance)
- Higher change concentration (focused modifications)

**Cursor**:
- Generates commit messages from staged changes + repo history
- Adapts to existing project conventions
- Background Agent may ignore commit message rules set in .cursor/rules

**Devin**:
- Multiline commit messages (48.9% importance)
- Conventional commit format
- More granular commits with dispersed file modifications

### 2.2 Generic AI-Generated Commit Message Signals

| Signal | AI Indicator | Human Indicator |
|--------|-------------|-----------------|
| Message length | Longer, more detailed | Shorter, terser |
| Multiline ratio | Higher | Lower |
| Bullet points | More frequent | Less frequent |
| File lists in body | Common | Rare |
| Conventional commit | More consistent adherence | Often inconsistent |
| Emoji usage | Systematic (mapped to types) | Sporadic or absent |
| Capitalization | Consistent | Varies per developer |
| Imperative mood | Nearly always | Mixed |
| "Refactor" frequency | Higher | Lower |

### 2.3 Phrases That Correlate with AI Generation

While no single phrase is definitive, these appear disproportionately in AI-generated commits:
- "Implement", "Add support for", "Update to handle"
- "Ensure", "Properly handle", "Fix edge case"
- "Add comprehensive", "Improve error handling"
- Precise technical descriptions of changes
- Grammatically perfect English with no abbreviations

**Reliability**: MEDIUM - easily influenced by user instructions, project rules, and evolving tool behavior. Good humans write similar messages.

## 3. Code-Level Forensic Patterns (Medium Reliability)

### 3.1 Stylometric Signatures

Research on LLM-generated code stylometry reveals distinguishable patterns.

**Comment Density and Style** (strongest code-level signal):
- AI code has higher comment density with more formal, descriptive comments
- Claude Code: 19.8% feature importance for comment density in fingerprinting study
- AI comments describe "what" more than "why"
- Docstrings are more thorough and template-like
- Removing comments only drops detection accuracy by 2-3 percentage points, suggesting signals persist in code structure

**Naming Conventions**:
- AI uses longer, more descriptive identifiers
- Claude favors longer identifiers; ChatGPT uses shorter variable names
- AI naming is more consistent within a session (uniform style)
- AI rarely uses abbreviations or project-specific shorthand

**Function Size and Structure**:
- AI generates functions with eerily similar length and structure
- Human code shows high variance in function sizes
- AI code exhibits template-like repetitive patterns

**Conditional Density**:
- Claude Code shows 27.2% feature importance for conditional statements
- AI tends toward more defensive programming with explicit condition checks
- Generic try/catch blocks are more common

**Import Patterns**:
- AI may include redundant or unused imports
- Import ordering is more consistent
- AI sometimes imports packages that don't exist (hallucination - a strong signal)

### 3.2 Diff-Level Patterns

| Pattern | AI Tendency | Human Tendency |
|---------|------------|----------------|
| Diff size | Larger, more uniform | Variable |
| Files per commit | More files changed at once | Fewer, more focused |
| Change concentration (Gini) | Lower (distributed) or very high (agent tasks) | Moderate |
| Whitespace consistency | Very consistent | Varies |
| Trailing whitespace | Rarely adds | Occasionally adds |
| Line length | More uniform | Variable |

### 3.3 Hallucination Artifacts

A strong signal unique to AI:
- References to non-existent functions, methods, or packages
- API calls using incorrect parameter names from training data
- Import of packages that don't exist in the target ecosystem
- These follow predictable patterns based on naming conventions learned from training data

### 3.4 Perplexity and Entropy Analysis

Statistical approaches from NLP research applied to code:

- **Perplexity**: AI-generated code shows more uniform (lower variance) perplexity across tokens. Human code has more "surprising" token sequences.
- **Burstiness**: AI code has lower burstiness (less variation in complexity across lines)
- **Effectiveness**: AUC of 87.81% for code >50 lines, but drops to 69.75% for code <20 lines
- **Line-level granularity**: Perplexity can detect specific lines as AI-generated

**Reliability**: MEDIUM - works well for large code blocks, poorly for small changes. Newer LLMs produce higher-entropy text, making detection harder over time.

## 4. Repository and Workflow Patterns (Medium Reliability)

### 4.1 Temporal Patterns

| Pattern | AI Indicator |
|---------|-------------|
| Commits at unusual hours for the developer | May indicate autonomous agent |
| Very regular commit intervals | Suggests automated workflow |
| Burst of many commits in short time | Agent completing a task |
| Weekend/off-hours activity | Agent running asynchronously |
| Timestamp clustering | Multiple commits within seconds |

### 4.2 Worktree Usage

AI agents increasingly use git worktrees for parallel work:
- Multiple branches active simultaneously
- Branch creation/deletion patterns
- `copilot/` or `cursor/` prefixed branches in worktrees
- Clean separation between parallel tasks

### 4.3 PR Patterns

| Signal | AI Pattern | Human Pattern |
|--------|-----------|---------------|
| PR body length | Longer, more detailed | Variable |
| Checklists | More common | Less common |
| Code blocks in description | More common | Less common |
| Bullet points | More structured | More narrative |
| Hyperlinks | More in Cursor PRs | Variable |
| PR-to-commit ratio | Often 1 PR per task | Variable |
| Draft PR usage | Common for agent workflows | Less common |

### 4.4 Merge Strategy Patterns

- AI agents often produce large PRs (1000+ lines)
- Squash-merge can obscure AI authorship (replaces agent author with merger)
- Recommendation: use merge commits to preserve AI attribution
- AI agents generally don't handle complex merge conflicts well

## 5. Detection Tools and Approaches

### 5.1 Explicit Tracking Tools

**Git AI** (usegitai.com):
- Vendor-agnostic, zero-configuration CLI
- Stores authorship logs as git notes (`git notes --ref=ai`)
- Pre/post-edit checkpoints capture AI vs human changes
- Preserves attribution through rebases, cherry-picks, squashes
- Requires agent integration (not retroactive)

**Agent Trace** (agent-trace.dev):
- Open specification by Cursor (RFC)
- JSON format recording file/line-level attribution
- Contributor types: `human`, `ai`, `mixed`, `unknown`
- Content hashing for tracking code across moves
- Partners: Amp, Cline, Cognition (Devin), Jules, Vercel, Cloudflare, OpenCode
- Schema: `https://agent-trace.dev/schemas/v1/trace-record.json`

### 5.2 Heuristic Detection Tools

**IsVibeCoded** (isvibecoded.com):
- Analyzes GitHub repos for AI-generated code patterns
- Examines: naming, repetition, comments, boilerplate, creativity
- Provides confidence score (0-100)
- Entertainment/insight only - not for hiring or integrity checks

**VibeDetect** (vibedetect.io):
- Weighted scoring system combining multiple analyzers
- Signals: template similarity, dependency selection, UX depth, naming patterns
- Score 0-100 mapped to human-readable verdicts
- Deterministic heuristics + optional LLM analysis

**AI Usage Measurement Framework**:
- Analyzes commit messages for tool signatures (simple string matching)
- Confidence levels: High (0.9) for explicit mentions, Medium (0.7-0.85), Low (0.3-0.6)
- Supports: Copilot, Windsurf, Cursor, ChatGPT, Claude, Devin, Amazon Q, Codeium, Tabnine

**SonarQube AI Code Assurance** (deprecated in 2026.1):
- Detects Copilot usage via GitHub App API
- Evaluates user Copilot activity + code contribution patterns
- Does NOT analyze code patterns - only user activity
- Limited to GitHub Copilot

### 5.3 Academic/Research Approaches

**Fingerprinting AI Coding Agents** (arXiv:2601.17406):
- 33,580 PRs from 5 agents (Codex, Copilot, Devin, Cursor, Claude Code)
- 41 features across 5 categories
- XGBoost model: 97.2% F1-score for multi-class agent identification
- Best features: multiline commit ratio (44.7%), change concentration Gini (10.1%)
- Per-agent accuracy: Codex 99%, Copilot 99%, Devin 94%, Cursor 85%, Claude Code 67%

**LLM Code Stylometry** (arXiv:2506.17323):
- Attributes code to specific LLMs (GPT-4, Claude, DeepSeek, etc.)
- CodeT5-Authorship achieves 97.56% binary, 95.40% multi-class
- Comment style is dominant signal (2-3pp impact)
- Tested only on C language; generalization unknown

**Perplexity-Based Detection** (AAAI 2024):
- Uses targeted masking perturbation on high-perplexity code areas
- AUC 87.81% for >50 LOC, 69.75% for <20 LOC
- GPTZero code detection: AUC only 0.64 (poor)

**Binoculars** (ICML 2024):
- Contrasts two related LLMs to detect AI text
- 90%+ detection at 0.01% false positive rate for text
- Not specifically validated for code
- Zero-shot, domain-agnostic

## 6. Building a Composite Detection Score

### 6.1 Signal Categories and Weights

```
COMPOSITE_SCORE =
    (explicit_signals * 0.40) +      # Trailers, author emails, branch prefixes
    (commit_message_signals * 0.25) + # Message patterns, phrases, structure
    (code_signals * 0.20) +          # Stylometrics, comment density, naming
    (workflow_signals * 0.15)         # Timing, PR structure, diff patterns
```

### 6.2 Explicit Signal Scoring (Weight: 40%)

| Signal | Score | Confidence |
|--------|-------|------------|
| Known AI Co-Authored-By trailer | 1.0 | Very High |
| Known AI author email | 1.0 | Very High |
| Known AI branch prefix (copilot/, cursor/) | 0.9 | High |
| "Generated with Claude Code" text | 1.0 | Very High |
| "aider: " message prefix | 0.9 | High |
| AI tool name in commit message | 0.7 | Medium |

### 6.3 Commit Message Signal Scoring (Weight: 25%)

| Signal | Score Contribution | Notes |
|--------|-------------------|-------|
| Multiline commit with extensive description | +0.2 | Codex/Devin pattern |
| Perfect conventional commit adherence | +0.15 | Most AI tools default to this |
| Emoji prefix pattern | +0.1 | Claude Code pattern |
| Unusually long message for project norm | +0.15 | Compare to repo baseline |
| Bullet-point file lists in body | +0.1 | AI documentation style |
| Perfect grammar, no abbreviations | +0.1 | AI tendency |
| Generic phrases ("ensure", "properly handle") | +0.05 | Weak signal |

### 6.4 Code Signal Scoring (Weight: 20%)

| Signal | Score Contribution | Notes |
|--------|-------------------|-------|
| Hallucinated imports (nonexistent) | +0.5 | Strong AI indicator |
| Uniform function sizes | +0.15 | Template-like structure |
| High comment density (vs repo norm) | +0.1 | AI tendency |
| Verbose variable names (vs repo norm) | +0.1 | AI tendency |
| Generic try/catch blocks | +0.1 | AI error handling |
| Low perplexity variance (if measured) | +0.15 | Statistical signal |

### 6.5 Workflow Signal Scoring (Weight: 15%)

| Signal | Score Contribution | Notes |
|--------|-------------------|-------|
| Commit at unusual hour for author | +0.1 | Asynchronous agent |
| Burst of commits in seconds | +0.15 | Agent task completion |
| Large diff size (>500 lines) | +0.1 | Agent style |
| Many files changed atomically | +0.1 | Agent style |
| PR from copilot/ or cursor/ branch | +0.3 | Direct indicator |

### 6.6 Interpretation Scale

| Score Range | Classification |
|-------------|---------------|
| 0.0-0.2 | Very likely human-authored |
| 0.2-0.4 | Probably human, possibly AI-assisted |
| 0.4-0.6 | Uncertain - AI assistance likely |
| 0.6-0.8 | Probably AI-generated |
| 0.8-1.0 | Very likely AI-generated |

### 6.7 The Spectrum Problem

AI contribution is not binary. The spectrum includes:
1. **Fully human** - No AI involvement
2. **AI-suggested completions** - Copilot/Tabnine inline suggestions (undetectable)
3. **AI-assisted** - Human writes code with AI chat help (partially detectable)
4. **AI pair-programmed** - Human guides AI agent, reviews output (detectable with attribution)
5. **AI-generated, human-reviewed** - Agent writes, human approves (often detectable)
6. **Fully autonomous** - Agent works independently (most detectable)

Current detection approaches work best at levels 4-6.

## 7. Per-Tool Forensic Fingerprint Reference

### 7.1 GitHub Copilot

**Inline Suggestions** (completion mode):
- No metadata added whatsoever
- Completely invisible in git history
- Indistinguishable from human code

**Copilot Coding Agent** (autonomous):
- Author: `GitHub Copilot` (bot account)
- Branch: `copilot/<descriptive-name>` (hardcoded prefix)
- Co-authored-by: requesting user
- PR: One PR per assigned issue
- Messages: Conventional, moderately detailed
- Behavioral fingerprint: Long PR descriptions (38.4%), high change concentration (24.9%)

### 7.2 Claude Code

- Trailer: `Co-Authored-By: Claude <noreply@anthropic.com>` (default, sometimes includes model name)
- Text: `Generated with Claude Code` in commit body
- Both can be disabled via settings
- Messages: Conventional commits, often with emoji
- Code fingerprint: High conditional density (27.2%), elevated comment density (19.8%)
- Behavioral: Well-documented, control-flow-intensive code

### 7.3 Cursor

**Editor mode**: No attribution (like Copilot inline)

**Agent mode** (including Background Agent):
- Trailer: `Co-authored-by: Cursor <cursoragent@cursor.com>` (auto-added, hard to disable)
- Branch: `cursor/<task-description>` prefix for Background Agent
- Messages: Adapts to project conventions
- Behavioral fingerprint: Bullet points in PR bodies (17.2%), hyperlinks (12.8%)

### 7.4 Aider

- Author name: `(aider)` appended to git author name
- Committer name: `(aider)` appended to committer name
- Trailer: `Co-authored-by: aider (model) <noreply@aider.chat>`
- Message prefix: `aider: ` (optional)
- Messages: Conventional commits via weak model
- All attribution is configurable and can be disabled

### 7.5 OpenAI Codex CLI

- Trailer: Configurable via `command_attribution` in `~/.codex/config.toml`
- Default: Recently added co-author trailer (default text configurable)
- No author/committer modification
- Behavioral fingerprint: Extensive multiline commits (67.5%), most distinctive overall
- Messages: Very detailed, multi-paragraph descriptions

### 7.6 Devin

- Author: `devin-ai-integration[bot]` / `bot@devin.ai`
- GitHub integration: GitHub App bot account
- PR template: `.github/PULL_REQUEST_TEMPLATE/devin_pr_template.md`
- Behavioral: Multiline commits (48.9%), distributed changes across files
- Highly autonomous - creates full PRs independently

### 7.7 Google Jules

- Author: `Jules` (bot account)
- Authorship modes: Jules sole author, co-authored, or user sole author
- Creates branches, opens PRs
- Supports Agent Trace specification

### 7.8 Windsurf / Codeium

- No commit attribution by default
- AI commit message generation feature (single-click)
- No Co-Authored-By trailer
- No branch prefix
- One of the hardest tools to detect

### 7.9 Amazon Q Developer

- No commit-level attribution by default
- Can generate commit messages via `@git` context modifier
- No trailers or author modification
- Configurable via rules for commit message format

### 7.10 JetBrains AI Assistant

- No commit attribution
- Generates commit messages from diffs
- No Co-Authored-By trailer
- No author modification

### 7.11 Bolt.new / Lovable / Replit Agent / v0

These web-based platforms have distinct patterns:
- **Lovable**: Auto-commits to GitHub, creates "lovable" branches, platform-specific author
- **Replit Agent**: Auto-commits at each step, uses Replit username, `no-reply@replit.com`
- **Bolt.new**: Auto-commits on non-breaking changes, StackBlitz integration
- **v0**: Frontend-only component generation, Vercel deployment context

### 7.12 Cline / Continue

- No default commit attribution
- Generate commit messages via git diff analysis
- Use VSCode Git extension API
- No trailers, no author modification
- Undetectable via metadata alone

## 8. Reliability Assessment

### 8.1 Reliable Signals (Low False Positive Rate)

| Signal | False Positive Risk | Notes |
|--------|-------------------|-------|
| Known AI email domain | Very Low | Definitive when present |
| `copilot/` or `cursor/` branch | Very Low | Hardcoded prefixes |
| Co-Authored-By with AI email | Very Low | Definitive when present |
| "Generated with Claude Code" | Very Low | Definitive when present |
| Hallucinated imports | Low | Almost never human error |
| `(aider)` in author name | Very Low | Definitive when present |

### 8.2 Unreliable Signals (High False Positive Rate)

| Signal | False Positive Risk | Notes |
|--------|-------------------|-------|
| "Clean" code style | Very High | Good developers write clean code |
| Conventional commit format | High | Many teams enforce this |
| Long commit messages | High | Some developers are verbose |
| Large diffs | Medium-High | Refactoring produces large diffs |
| Consistent formatting | High | Formatters (Prettier, Black) do this |
| Perfect grammar | Medium | Non-native speakers use grammar tools |
| Generic error handling | Medium | Common in enterprise code |
| Perplexity analysis (<20 LOC) | High | AUC drops to 0.64-0.70 |

### 8.3 Evasion and Degradation

**Easy to evade**:
- Disable Co-Authored-By trailers (all tools support this)
- Rewrite commit messages manually
- Amend author information
- Use interactive rebase to clean history

**Hard to evade**:
- Code stylometric patterns (require significant manual editing)
- Hallucination artifacts (require careful review)
- Branch prefixes on autonomous agents (hardcoded)
- Statistical patterns across many commits (can't fake burstiness)

**Degradation over time**:
- As AI tools improve, code becomes more human-like
- Newer LLMs produce higher-entropy, more varied output
- Tools learn from user conventions, reducing distinctiveness
- Model retraining needed as agents evolve

## 9. Emerging Standards

### 9.1 Agent Trace Specification

The most promising standard for forward-looking attribution:

```json
{
  "version": "0.1.0",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-01-25T10:00:00Z",
  "files": [{
    "path": "src/app.ts",
    "conversations": [{
      "contributor": { "type": "ai", "model_id": "anthropic/claude-opus-4-5" },
      "ranges": [{ "start_line": 1, "end_line": 50 }]
    }]
  }]
}
```

Contributor types: `human`, `ai`, `mixed`, `unknown`
Storage: flexible (files, git notes, database)
Content hashing: `murmur3:<hash>` for tracking code across moves

### 9.2 Alternative Trailers

Emerging proposals beyond Co-Authored-By:

```
Coding-Agent: Claude Code
Model: claude-opus-4-6

# Or consolidated:
AI-assistant: OpenCode v1.0.203 (Claude Opus 4.5)
```

### 9.3 Git AI (usegitai.com)

Vendor-agnostic tracking via git notes:
- Pre/post-edit checkpoints
- Authorship logs linking line ranges to agent sessions
- Survives rebases, cherry-picks, squashes
- Requires proactive agent integration

## 10. Practical Implementation for git-map

### 10.1 Detection Pipeline

```
1. EXPLICIT CHECK (fast, definitive)
   - Parse trailers for known AI emails
   - Check author email against known AI domains
   - Check branch name for known prefixes
   - Search message body for attribution text

2. HEURISTIC ANALYSIS (medium speed, probabilistic)
   - Score commit message patterns vs repo baseline
   - Analyze diff characteristics (size, file count, concentration)
   - Check temporal patterns (unusual hours, burst frequency)

3. STYLOMETRIC ANALYSIS (slow, probabilistic, optional)
   - Comment density vs repo baseline
   - Naming convention consistency
   - Function size uniformity
   - Import validity checking
   - Perplexity analysis (for large changes)

4. COMPOSITE SCORING
   - Weight and combine all signals
   - Classify on spectrum: human -> assisted -> generated
   - Output confidence level
```

### 10.2 Minimum Viable Detection

For a static cached artifact from git history, focus on:

1. **Trailer parsing** - regex for Co-Authored-By with AI tool names/emails
2. **Author email matching** - against known AI domain list
3. **Branch prefix matching** - `copilot/`, `cursor/`
4. **Attribution text matching** - "Generated with Claude Code", "aider:" prefix
5. **Message pattern scoring** - multiline ratio, length vs baseline, conventional adherence

This covers the highest-confidence signals with lowest computational cost.

### 10.3 Known AI Email Registry

```
# Definitive AI tool emails (update as tools change)
noreply@anthropic.com                    # Claude Code
noreply@aider.chat                       # Aider
cursoragent@cursor.com                   # Cursor
agent@cursor.com                         # Cursor (variant)
bot@devin.ai                             # Devin
no-reply@replit.com                      # Replit Agent
*copilot*@users.noreply.github.com       # Copilot (pattern)
*devin*@users.noreply.github.com         # Devin (pattern)
```

### 10.4 Known AI Tool String Patterns

```
# In Co-Authored-By trailer name field
"Claude"
"aider"
"Cursor"
"Copilot"
"GitHub Copilot"
"Codex"
"Jules"
"Devin"

# In commit message body
"Generated with Claude Code"
"Generated by Claude"
"Generated by Copilot"
"claude.com/claude-code"

# In commit message prefix
"aider: "

# In author name suffix
"(aider)"
"(Cursor)"
```

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Treating all clean code as AI | Good developers write clean code too | Use composite scoring, not single signals |
| Ignoring disabled attribution | Most tools allow disabling trailers | Don't rely solely on explicit signals |
| Static detection rules | AI tools evolve, patterns change | Maintain updateable pattern registry |
| Binary classification | AI assistance is a spectrum | Use confidence scores, not yes/no |
| Ignoring project conventions | Teams may enforce AI-like patterns | Compare against per-repo baselines |
| Trusting perplexity for short code | AUC drops to 0.64-0.70 for <20 LOC | Only apply to substantial changes |
| Assuming Co-Authored-By is definitive | Users can add fake trailers | Cross-reference with other signals |

## Best Practices

1. **Start with explicit signals** - Trailer parsing and email matching catch the majority of attributed AI commits with zero false positives (Source: tool documentation analysis)
2. **Maintain an updateable registry** - AI tool attribution formats change frequently; design for easy updates (Source: Claude Code issues #617, #5458, #4224)
3. **Compare against per-repo baselines** - "Unusual" commit message length means nothing without knowing the project's normal patterns (Source: fingerprinting study methodology)
4. **Use composite scoring, not thresholds** - No single heuristic is reliable alone; combine signals with appropriate weights (Source: VibeDetect methodology)
5. **Classify on a spectrum** - Distinguish between AI-generated, AI-assisted, and human-authored; avoid binary classification (Source: Agent Trace contributor types)
6. **Accept uncertainty** - Inline completion tools (Copilot suggestions, Tabnine) leave zero trace; some AI contributions will always be invisible (Source: tool documentation)
7. **Plan for evolution** - Retrain/update detection as AI tools change; what works today may not work in 6 months (Source: arXiv:2601.17406 limitations)

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [Fingerprinting AI Coding Agents on GitHub](https://arxiv.org/abs/2601.17406) | Academic Paper | Most comprehensive empirical study on agent detection |
| [AIDev Dataset](https://arxiv.org/abs/2602.09185) | Dataset/Paper | 932K agent-authored PRs for research |
| [Agent Trace Specification](https://agent-trace.dev/) | Open Spec | Emerging standard for AI code attribution |
| [Git AI](https://usegitai.com/) | Tool | Vendor-agnostic AI code tracking via git notes |
| [LLM Code Stylometry](https://arxiv.org/abs/2506.17323) | Academic Paper | Model-level attribution via code style |
| [Aider Git Integration](https://aider.chat/docs/git.html) | Docs | Detailed attribution configuration reference |
| [Claude Code Git Settings](https://docs.anthropic.com/en/docs/claude-code/settings) | Docs | Attribution configuration |
| [Copilot Coding Agent Docs](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent) | Docs | Official branch/commit behavior |
| [Cursor Git Integration](https://cursor.com/help/integrations/git) | Docs | Agent attribution and blame features |
| [AI Coding Agent Commits Deserve Better](https://fabiorehm.com/blog/2026/03/02/our-coding-agent-commits-deserve-better-than-co-authored-by/) | Blog Post | Analysis of Co-Authored-By problems, alternative proposals |
| [Binoculars: Zero-Shot Detection](https://arxiv.org/abs/2401.12070) | Academic Paper | State-of-the-art LLM text detection method |
| [Perplexity-Based Code Detection](https://ojs.aaai.org/index.php/AAAI/article/view/30361/32410) | Academic Paper | Code-specific perplexity detection approach |
| [Code Fingerprints: Disentangled Attribution](https://arxiv.org/html/2603.04212v1) | Academic Paper | Recent work on LLM code attribution |

---

*This guide was synthesized from 42 sources. See `resources/ai-commit-detection-forensics-sources.json` for full source list.*
