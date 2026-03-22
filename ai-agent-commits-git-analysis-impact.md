# Learning Guide: How AI-Generated Code and AI Agent Commits Affect Git History Analysis and Developer Metrics

**Generated**: 2026-03-14
**Sources**: 25 resources analyzed
**Depth**: medium

## Prerequisites

- Familiarity with git concepts: commits, blame, log, diff, branches, merges
- Understanding of traditional git history analysis metrics (churn, ownership, coupling, bus factor)
- Awareness of AI coding tools (GitHub Copilot, Claude Code, Cursor, Codex, Aider, Windsurf)
- Basic understanding of code quality metrics (complexity, duplication, test coverage)

## TL;DR

- AI tools now generate 41-46% of all code, fundamentally changing what git history data means for analysis tools.
- GitClear's research on 211M lines shows code duplication up 4-8x, refactoring down from 25% to under 10%, and short-term churn increasing - all since AI coding tools became mainstream.
- Traditional ownership/bus-factor metrics break down when AI writes the code but humans accept it - the human may not truly "own" or understand the code they committed.
- Two emerging standards - Git AI (git-notes-based, explicit agent marking) and Agent Trace (Cursor's JSON-based spec) - aim to track AI vs. human authorship at line granularity.
- Git analysis tools must adapt by: detecting AI contributions, applying separate quality gates for AI code, distinguishing prompted code from autonomously generated code, and recalibrating churn/ownership metrics.

## Core Concepts

### 1. Scale of AI Code in Modern Repositories

The proportion of AI-generated code has crossed a critical threshold where it materially affects all git history analysis:

- **41% of all code** written in 2025 was AI-generated (industry aggregate).
- **GitHub Copilot** generates 46% of code for its users, with Java reaching 61%. 88% of Copilot suggestions stay in the final version.
- **50% of companies** now have at least half their code AI-generated, up from 20% at the start of 2025 (Jellyfish data).
- **GitHub Octoverse 2025**: 986 million commits pushed (+25.1% YoY), 43.2 million PRs merged monthly (+23% YoY). 80% of new developers use Copilot within their first week.
- **25% of Google's code** is AI-assisted (per CEO Sundar Pichai).

**Implication for git-map**: Any git history analysis tool that does not account for AI-generated code is analyzing a fundamentally different codebase than what existed before 2023. Metrics calibrated on human-only codebases will produce misleading signals.

### 2. How AI Code Differs From Human Code in Git History

GitClear's research on 211 million changed lines (2020-2024) reveals systematic differences:

**Code Composition Shifts**

| Metric | 2020/2021 | 2024 | Trend |
|--------|-----------|------|-------|
| Copy/pasted (cloned) code | 8.3% | 12.3% | Up 48% |
| Moved/refactored code | 25% | <10% | Down 60%+ |
| Duplicated code blocks | Baseline | 8x increase | Dramatic rise |
| Short-term churn (revised within 2 weeks) | 3.1% | 5.7% | Up 84% |

**Commit Pattern Differences**

AI-generated commits exhibit distinct patterns:

- **Larger commits**: PR size increased 154% on average with AI tools. AI agents often produce big-batch changes rather than incremental human-style edits.
- **Higher issue density**: AI-generated PRs contain 10.83 issues per PR vs. 6.45 for human-only (1.7x more).
- **Uniform style**: AI code shows "a level of perfection in syntax" - verbose function names, consistent formatting, formal documentation, modern syntax. Human code has abbreviated names, inconsistent formatting, brief comments, legacy patterns.
- **Fewer iterative commits**: AI repos often show fewer commits with large code chunks committed at once, sometimes with generic messages like "Updated files."
- **Higher revert/churn rate**: Code that needs revision within two weeks of initial commit has risen significantly.

**Quality Metric Distortion**

Research from AlterSquare on AI-assisted PRs found:
- 82% had inadequate error handling
- 76% lacked network timeouts
- 44% introduced N+1 database query problems
- Duplicate code ratios rose from 3.1% to 14.2% in AI-heavy projects
- Cyclomatic complexity doubled (4.2 to 8.1)
- 68% of AI modules required complete rewrites, not minor fixes

### 3. Impact on Code Churn Metrics

Code churn - the rate at which recently written code is modified or deleted - is one of the most disrupted metrics:

**The "AI Churn Spike"**

AI-generated code is analogous to code written by "a short-term developer that doesn't thoughtfully integrate their work into the broader project" (GitClear). This manifests as:

- **Higher short-term churn**: Code revised within 2 weeks went from 3.1% (2020) to 5.7% (2024). AI suggestions are accepted quickly but often need rapid correction.
- **Inflated "added" lines**: Agent-first repositories show +76.6% lines added at adoption. But this velocity doesn't necessarily correlate with useful work.
- **Reduced "moved" code**: Refactoring (moving code to better locations) dropped from 25% to under 10%. AI generates new code rather than reorganizing existing code.

**DORA Impact**: A 7.2% drop in software delivery stability has been linked to AI usage (DORA report).

**What This Means for git-map**: Raw churn metrics will show inflated instability. A git-map tool needs to distinguish between:
1. Churn from human iteration (normal, healthy)
2. Churn from AI-generated code being corrected (signal of AI quality issues)
3. Churn from AI-assisted refactoring (potentially healthy)

### 4. Ownership and Attribution Breakdown

Traditional git analysis assigns ownership based on who authored commits. AI disrupts this model fundamentally.

**The Attribution Gap**

- **git blame becomes unreliable**: When Copilot completes 46% of a developer's code, `git blame` shows the human as author but the human may not understand the code.
- **Co-Authored-By inconsistency**: Claude Code adds `Co-Authored-By: Claude <noreply@anthropic.com>` by default, but Copilot adds no attribution. Aider adds `(aider)` to the author name. There is no universal standard.
- **Phantom ownership**: A developer who accepts AI suggestions without deep review appears to "own" code they may not be able to debug or explain.

**Four Responsibility Frameworks Emerging**

| Model | Description | Ownership Implication |
|-------|-------------|----------------------|
| AI as tool | Like a compiler - developer fully owns output | Traditional ownership applies |
| AI as junior dev | Developer supervises and reviews | Shared but human responsible |
| AI as independent agent | Works autonomously on tasks | Policy and traceability needed |
| AI as teammate | Part of team workflow | Requires review and metadata |

**Bot Sponsorship Pattern**: Teams are implementing policies where any AI-authored or reviewed PR must have a named human "sponsor" who takes full responsibility. This preserves accountability but doesn't solve the knowledge gap.

**Legal Implications**: In fintech, healthtech, and enterprise software, unclear authorship has legal consequences. Clear authorship trails are essential for debugging, liability, and regulatory compliance.

### 5. Bus Factor and Knowledge Risk

The bus factor measures how many people need to leave before a project stalls. AI changes this calculation:

**Knowledge Dilution**

- Creating a PR signals you understand the code - but with AI, there is a disconnect between implied and actual understanding.
- Teams go "from owners to operators" - they can follow AI instructions to add features but cannot debug complex issues.
- Tribal knowledge about design decisions evaporates when AI makes architectural choices without documenting rationale.
- 90% of developers have committed AI-generated code, but only 3% highly trust it.

**The AI-Amplified Bus Factor**

If one developer uses AI to produce 5x more code, they become a larger single point of failure - not because they wrote more, but because they may be the only person who knows the prompts and intent behind AI-generated code. When they leave, the organization loses not just a developer but the entire context chain.

**Recommendation for git-map**: Bus factor calculations should weight ownership by understanding confidence, not just commit volume. Consider:
- Ratio of AI-to-human code per contributor
- Whether the contributor has made subsequent manual modifications (indicating comprehension)
- Code review depth on AI-generated PRs

### 6. Coupling Analysis Distortion

AI-generated code introduces coupling patterns that differ from human-created coupling:

**Context Window Isolation Problem**

AI agents treat each prompt as a standalone task, producing code that appears logical in isolation but clashes with existing patterns. This creates:

- **Hidden circular dependencies**: Service A depends on B, B depends on A - visible only during refactoring.
- **Tight coupling by default**: AI prioritizes immediacy over separation of concerns. It adds redundant libraries without checking what already exists.
- **Missing abstraction layers**: When given "Build a REST API," agents generate flat structures with inline route handlers, tightly coupled logic, and missing service/repository layers.
- **Over-abstraction**: AI sometimes merges distinct components into bloated "universal" abstractions.

**Architectural Erosion**

- AI doesn't track what's already been written elsewhere and solves each prompt in isolation.
- No separation of concerns between presentation, business logic, and data layers unless specifically instructed.
- Scalability gaps: no pagination, rate limiting, or async processing by default.
- Resilience gaps: missing retry logic, circuit breakers, graceful fallbacks.

**Implication for coupling analysis**: File co-change patterns in AI-heavy codebases may reflect AI's tendency to touch related files simultaneously during a prompt session, rather than genuine architectural coupling. True coupling signals need to be distinguished from AI-session-correlation artifacts.

### 7. AI Commit Detection Methods

Several approaches exist to identify AI-generated commits:

**Explicit Tracking (Recommended)**

| Tool/Standard | Method | Storage | Status |
|--------------|--------|---------|--------|
| Git AI | Agent hooks mark code at creation time | Git Notes (.git/ai/) | Active, open source |
| Agent Trace (Cursor) | JSON trace records link code ranges to conversations | .agent-trace/traces.jsonl | RFC/draft phase |
| Sonar AI Code Assurance | Detects Copilot usage via GitHub API | SonarQube metadata | Production |
| Claude Code Co-Authored-By | Trailer in commit messages | Commit message | Default-on |
| Aider author marking | Adds (aider) to git author name | Commit metadata | Default-on |

**Heuristic Detection (Fallback)**

Pattern-based signals for identifying AI-generated code:

- **Commit patterns**: Fewer commits with larger code chunks, generic messages, uniform formatting.
- **Code style**: Verbose function names, consistent formatting, formal documentation, modern syntax, few bad practices. Human code shows abbreviated names, inconsistent formatting, legacy syntax.
- **Structural patterns**: Template similarity, dependency selection patterns, naming conventions, excessive symmetry.
- **Repository signals**: Analyzed by tools like isvibecoded.com and VibeDetect, which combine deterministic heuristics and optional LLM analysis.

**Git AI Technical Details**

Git AI operates as a zero-config git extension:
- Uses pre-edit and post-edit hooks to create checkpoints (small diffs in .git/ai/)
- Each checkpoint records whether changes are AI or human authored
- Consolidates into Authorship Logs (JSON in Git Notes) at commit time
- Preserves attribution across rebases, merges, squashes, stash/pops, cherry-picks, and amends
- Stores session metadata: agent tool/model, human author, additions/deletions, accepted/overridden lines
- ~10-20ms overhead per command, fully offline

**Agent Trace Technical Details**

Cursor's specification uses JSON-based trace records:
- Tracks contributions at file or line level
- Categories: human, AI, mixed, unknown
- Storage-agnostic (files, git notes, databases)
- Supports multiple VCS (Git, Jujutsu, Mercurial)
- Optional content hashes for tracking refactored code
- Does not define ownership semantics or quality assessment

### 8. How Analysis Tools Handle AI Code

**CodeScene**

- Provides Code Health metric linking code quality to development velocity
- MCP server creates continuous feedback loop for AI agents with real-time quality checks
- Hotspot analysis identifies frequently-changed unhealthy code
- Six best practice patterns for agentic AI: Pull Risk Forward, Safeguard Generated Code, Refactor to Expand Surface, Encode Principles (AGENTS.md), Code Coverage Gates, End-to-End Automation
- Code Health threshold: target minimum 9.5 (ideally 10.0) for AI-ready code
- Key insight: "AI operates in a self-harm mode" - often writing code it cannot reliably maintain later

**GitClear**

- Analyzes code composition: new, updated, deleted, moved, copy/pasted categories
- Development Output metric: commits per week divided by contributing authors
- Tracks AI impact through changed-lines taxonomy
- Primary researcher on AI code quality decline (211M lines analyzed)

**SonarQube AI Code Assurance**

- Auto-detects GitHub Copilot usage via GitHub API
- Routes AI-generated code through separate quality gates
- Built-in "Sonar way for AI Code" gate with seven conditions
- Projects passing the gate receive a badge
- Higher standards for complexity, bugs, and injection vulnerabilities

**Jellyfish**

- Tracks AI adoption rates and productivity gains
- Measures PR volume per engineer, cycle time, code quality indicators
- Data shows 113% increase in PRs per engineer at full AI adoption
- Median cycle time dropped 24% (16.7 to 12.7 hours)
- Bug fix PRs increased from 7.5% to 9.5% at high-adoption companies

### 9. Pair Programming vs. Autonomous Agents: Different Git Footprints

The distinction matters significantly for git history analysis:

**Pair Programming Tools (Copilot, Cursor inline)**

- Code appears as part of normal developer commits
- No separate commit attribution - code is interleaved with human edits
- Harder to distinguish AI from human contributions
- Commit frequency similar to human patterns
- PR size may increase but follows human review cycles

**Autonomous Agents (Codex, Claude Code, GitHub Copilot Coding Agent, Aider)**

- Create their own commits and branches, often in git worktrees
- Every edit may be a separate commit (Aider: "every edit is a commit")
- PR workflow is automated: branch creation, commit messages, PR description
- Devin shows 67% merge rate on well-defined tasks, but 85% failure on complex/ambiguous tasks
- Worktree farm model: multiple agents work simultaneously in isolated environments
- Agent-first repos show +36.3% commits and +76.6% lines added (research data)

**Research Findings (Staggered Difference-in-Differences Study)**

| Metric | Agent-First Effect | IDE-First Effect |
|--------|-------------------|------------------|
| Commits | +36.25% | +3.06% |
| Lines Added | +76.59% | -6.34% |
| Static Analysis Warnings | +17.73% | +19.00% |
| Cognitive Complexity | +34.85% | +42.87% |

Key finding: Agents deliver meaningful velocity gains only in new-to-AI environments. Teams with existing IDE-based AI see diminishing returns. Both increase complexity.

### 10. Impact on Code Review Metrics

AI-generated code fundamentally changes code review dynamics:

- **PR volume surge**: Companies with full AI adoption see 113% more PRs per engineer.
- **Review time increase**: Code review time per PR increases 25-40% when reviewers didn't guide the AI output. Teams with high AI adoption saw review times balloon 91%.
- **PR size inflation**: Average PR size increased 154% with AI tools.
- **Trust deficit**: Only 3% of developers highly trust AI code; 71% refuse to merge without manual review.
- **AI reviewing AI**: By October 2025, 10-20% of PRs were reviewed by AI tools. Coding review agent adoption went from 14.8% to 51.4% in one year.
- **Readability issues**: 3x higher in AI contributions.
- **Security issues**: 2.74x higher in AI-generated code.

### 11. The "AI Slop" Problem

The term "AI slop" describes low-quality generated code that inflates metrics:

- AI code looks clean superficially - compiles, follows naming conventions, proper formatting - but masks deeper problems.
- "AI-generated code is almost always correct for the world it imagines. The problem is the world it imagines doesn't exist."
- Debugging AI code is harder than writing it manually because you're debugging assumptions that were never documented.
- AI velocity can mask downstream costs: projects saw 19% slower task completion despite expectations of 24% improvement, due to AI-error resolution overhead.
- 40% of developers spend 2-5 working days per month on debugging and maintenance caused by technical debt, much of it AI-originated.

### 12. Conventional Commit Conventions and AI

AI tools have mixed compatibility with project conventions:

- **AI commit message generators**: Tools like aicommits, LLMCommit, and IDE extensions generate conventional commit messages from diffs. Quality varies.
- **Windsurf**: Generates commit messages automatically by analyzing code changes.
- **Claude Code**: Adds a "Generated with Claude Code" line and Co-Authored-By trailer. Can be configured via `includeCoAuthoredBy` and `gitAttribution` settings.
- **Aider**: Adds (aider) to author name and includes the model as co-author.
- **Convention adherence**: Many organizations have strict commit message conventions that don't accommodate AI attribution lines. Extra text can conflict with tooling that parses commit messages.

**Proposed Attribution Categories**

| Level | AI Contribution | Trailer |
|-------|----------------|---------|
| Assisted-by | ~33% AI-generated | `Assisted-by: <tool>` |
| Co-authored-by | 34-66% AI participation | `Co-authored-by: <tool>` |
| Generated-by | 67-100% AI-created | `Generated-by: <tool>` |
| Commit-generated-by | AI summarized only | `Commit-generated-by: <tool>` |

No universal standard has been adopted yet.

### 13. Organizational Adaptations

Organizations are adapting git workflows for AI contributions:

**Governance Approaches**

- **AI Code Governance Tools**: Platforms like Sonar, CodeScene, and Kodus provide automated quality gates specifically for AI code.
- **Governance as Code**: Embedding AI policies into CI/CD pipelines as enforceable, version-controlled rules.
- **AI nutrition labels**: Self-disclosure standards for digital products to make AI usage transparent.
- **EU Code of Practice**: Consultations on marking and labeling AI-generated content opened.

**Team-Level Practices**

- Require developer explanation of AI-suggested code before merging
- Keep architectural decisions human-controlled
- Dedicate sprint capacity to reviewing and refactoring AI code
- Tag AI-generated PRs and require extra approvals
- Restrict AI deployment in security-sensitive code
- Document architectural intent in ARCHITECTURE.md or AGENTS.md
- Enforce evidence-gated merges for high-impact features

### 14. New Metrics and Adjustments for the AI Era

Traditional metrics need recalibration. Here are recommended adaptations:

**Adjusted Metrics**

| Traditional Metric | Problem with AI | Recommended Adjustment |
|-------------------|-----------------|----------------------|
| Lines of Code (LOC) | Inflated by AI duplication | Weight by AI-attribution; track "effective LOC" |
| Code Churn | Inflated by AI generate-then-fix cycles | Separate AI churn from human churn; track "churn source" |
| Ownership (git blame) | Human may not understand AI code | Weight by review depth and manual modification ratio |
| Bus Factor | Distorted when AI amplifies one developer | Track "understanding factor" alongside authorship |
| Coupling (co-change) | AI sessions create artificial co-change | Filter by commit source; distinguish session correlation from architectural coupling |
| Commit Frequency | Inflated by agent auto-commits | Normalize by commit source (human/agent/pair) |
| PR Merge Rate | Volume up but quality concerns | Track quality-adjusted merge rate (issues per PR) |
| Code Complexity | AI increases complexity ~35-43% | Monitor per-commit complexity delta by source |

**New Metrics to Consider**

| Metric | What It Measures | Why It Matters |
|--------|-----------------|----------------|
| AI Code Ratio | % of code from AI tools per file/module/repo | Identifies areas with ownership risk |
| Code Comprehension Score | Whether committer can explain their changes | Prevents phantom ownership |
| AI Churn Rate | Churn specifically in AI-generated code | Separates AI quality signal from normal iteration |
| Prompt-to-Production Ratio | How much AI code survives to production | Measures AI effectiveness |
| Agent Session Correlation | Files changed together in AI sessions | Distinguishes from architectural coupling |
| Code Health Trend | CodeScene-style quality over time by source | Catches AI-driven degradation |
| Review Depth Ratio | Human review time per AI-generated line | Ensures adequate oversight |
| Epistemic Debt | Knowledge gaps from AI code acceptance | Tracks understanding risk |

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Treating AI commits same as human commits | No detection/attribution in place | Implement Git AI, Agent Trace, or trailer-based tracking |
| Inflated productivity metrics | AI generates more LOC/commits | Normalize metrics by AI-ratio; track quality-adjusted output |
| False ownership signals | git blame shows human who accepted AI code | Weight ownership by manual modification and review depth |
| Misleading churn data | AI code gets revised quickly after generation | Separate AI churn from human iteration churn |
| Artificial coupling detection | AI modifies related files in same session | Filter co-change by commit source |
| Underestimating bus factor risk | One developer + AI looks like strong coverage | Track understanding-weighted ownership, not just authorship |
| Ignoring architectural erosion | AI generates code without system-wide awareness | Monitor Code Health metrics; check for coupling/duplication trends |
| Trusting LOC as productivity measure | AI inflates lines without proportional value | Use "effective lines" or quality-adjusted metrics |
| Missing AI code in security analysis | AI introduces 2.74x more security issues | Route AI code through stricter quality gates (Sonar) |
| Assuming commit messages reflect intent | AI generates plausible but shallow messages | Parse for AI tool signatures; require context metadata |

## Best Practices for Git Analysis Tools in the AI Era

1. **Detect AI contributions explicitly** - Integrate with Git AI, Agent Trace, or parse Co-Authored-By trailers. Heuristic detection as fallback. (Sources: Git AI, Agent Trace, Sonar)

2. **Separate metrics by source** - Track human-authored, AI-assisted, and AI-generated code as distinct categories. Each has different quality profiles. (Sources: GitClear, Jellyfish, AlterSquare)

3. **Apply quality-weighted ownership** - Weight code ownership by manual modification ratio and review depth, not just commit authorship. (Sources: Pullflow, ZenvanRiel)

4. **Recalibrate churn baselines** - The baseline for "normal" churn has shifted. AI-era churn above 5% within two weeks is the new normal, not necessarily a risk signal. (Source: GitClear)

5. **Distinguish session correlation from coupling** - When AI modifies multiple files in one prompt session, this is not necessarily architectural coupling. Filter by commit metadata. (Sources: AlterSquare, vFunction)

6. **Implement separate quality gates** - AI-generated code should face stricter static analysis, complexity, and security checks. (Sources: Sonar, CodeScene)

7. **Track Code Health trends** - Monitor objective code quality metrics (CodeScene Code Health, SonarQube Quality Gates) over time, segmented by AI contribution ratio. (Sources: CodeScene, Sonar)

8. **Normalize productivity metrics** - PRs per engineer, LOC, and commit frequency should be normalized by AI adoption level to enable meaningful comparisons. (Source: Jellyfish)

9. **Preserve provenance metadata** - Store AI tool/model, prompt context, and review status alongside git history for future analysis. (Sources: Git AI, Agent Trace, DEV Community)

10. **Monitor the understanding gap** - Track whether developers can explain code they committed. Bus factor calculations should consider comprehension, not just authorship. (Sources: ZenvanRiel, Fast Company)

## Tools and Standards Reference

| Tool/Standard | Type | Focus | URL |
|--------------|------|-------|-----|
| Git AI | Git extension | Explicit AI code tracking | usegitai.com |
| Agent Trace | Specification | Vendor-neutral attribution | agent-trace.dev |
| CodeScene | Analysis platform | Code Health + behavioral analysis | codescene.com |
| GitClear | Analytics platform | Code quality metrics + AI research | gitclear.com |
| SonarQube | Quality platform | AI Code Assurance quality gates | sonarsource.com |
| Jellyfish | Engineering intelligence | AI adoption and productivity metrics | jellyfish.co |
| isvibecoded | Detection tool | Repository-level AI detection | isvibecoded.com |
| VibeDetect | Detection tool | AI code scoring with heuristics | vibedetect.io |

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [GitClear 2025 AI Code Quality Research](https://www.gitclear.com/ai_assistant_code_quality_2025_research) | Research Report | Definitive study on AI impact on code quality metrics (211M lines) |
| [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/) | Industry Report | Platform-scale data on commit and developer activity trends |
| [AI IDEs or Autonomous Agents? (arXiv)](https://arxiv.org/html/2601.13597) | Academic Paper | Rigorous diff-in-diff study of agent vs IDE impact on development metrics |
| [Git AI Standard v3.0.0](https://github.com/git-ai-project/git-ai/blob/main/specs/git_ai_standard_v3.0.0.md) | Specification | Technical standard for tracking AI code authorship in git |
| [Agent Trace Specification](https://agent-trace.dev/) | Specification | Cursor's open standard for AI code attribution |
| [The New Git Blame (DEV Community)](https://dev.to/pullflow/the-new-git-blame-whos-responsible-when-ai-writes-the-code-285j) | Blog Post | Practical analysis of AI's impact on code ownership |
| [CodeScene: Agentic AI Coding Best Practices](https://codescene.com/blog/agentic-ai-coding-best-practice-patterns-for-speed-with-quality) | Blog Post | Six operational patterns for reliable AI coding with quality guardrails |
| [Jellyfish 2025 AI Metrics Review](https://jellyfish.co/blog/2025-ai-metrics-in-review/) | Data Report | 12 months of AI adoption and productivity data across companies |
| [AI-Generated Code Looks Clean. Here's Why It Isn't (AlterSquare)](https://altersquare.io/ai-generated-code-next-refactor-will-prove-its-not-clean/) | Analysis | Deep dive on hidden quality problems in AI code |
| [Did AI Erase Attribution? (DEV Community)](https://dev.to/anchildress1/did-ai-erase-attribution-your-git-history-is-missing-a-co-author-1m2l) | Blog Post | Proposed attribution categories for AI contributions |
| [Maintaining Code Ownership with AI (ZenvanRiel)](https://zenvanriel.com/ai-engineer-blog/maintaining-code-ownership-with-ai-assistance/) | Blog Post | Practical guide to preserving code ownership in AI era |
| [Vibe Coding Architecture (vFunction)](https://vfunction.com/blog/vibe-coding-architecture-ai-agents/) | Blog Post | How AI agents affect software architecture and boundaries |
| [Claude Code Git Attribution (DeployHQ)](https://www.deployhq.com/blog/how-to-use-git-with-claude-code-understanding-the-co-authored-by-attribution) | Tutorial | Detailed guide to Claude Code's git integration |
| [Sonar AI Code Assurance](https://www.sonarsource.com/blog/auto-detect-and-review-ai-generated-code-from-github-copilot/) | Product Blog | How Sonar detects and quality-gates AI code |
| [Agent Trace (InfoQ)](https://www.infoq.com/news/2026/02/agent-trace-cursor/) | News Analysis | Industry context for the Agent Trace specification |

---

*This guide was synthesized from 25 sources. See `resources/ai-agent-commits-git-analysis-impact-sources.json` for full source list with quality scores.*
