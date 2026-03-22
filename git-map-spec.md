# git-map: Static Git History Analysis Artifact

> Specification for a cached, incrementally-updatable JSON artifact that extracts temporal, social, and behavioral signals from git history. Companion to repo-map (AST-based static analysis).

**Status**: Design spec
**Date**: 2026-03-14
**Research**: Synthesized from 91 sources across 3 learning guides

## 1. Concept

git-map creates a static JSON artifact from git history analysis. It sits alongside repo-map in agent-core:

```
repo-map  ->  WHAT exists (symbols, exports, imports)     [static snapshot]
git-map   ->  WHAT HAPPENED (activity, ownership, trends) [temporal layer]
```

Combined, they give consumers a complete picture without re-running expensive analysis.

### Design principles

1. **Static, not live** - Pre-computed aggregates stored as JSON, not on-demand queries
2. **Incremental** - Git history is append-only; only process new commits since last scan
3. **No LLM** - Pure deterministic extraction and aggregation
4. **Query layer** - Consumers call functions over the cached map, not git commands
5. **AI-aware** - Detect and track AI-generated commits as a first-class concern

### Validation model

```
HEAD === map.git.analyzedUpTo  ->  valid, no work needed
HEAD !== map.git.analyzedUpTo  ->  run delta extraction, merge into map
analyzedUpTo not in history    ->  history rewritten (force push), full rebuild
```

## 2. Architecture

### Where it lives

```
agent-core/lib/
  git-map/
    index.js          # init, update, status, load (public API)
    extractor.js      # runs git commands, returns raw delta
    aggregator.js     # merges delta into existing map
    queries.js        # pure functions over the cached map
    cache.js          # load/save git-map.json (reuses repo-map cache pattern)
    ai-detect.js      # AI commit detection logic
    ai-signatures.json # updateable AI tool signature registry
  collectors/
    index.js          # add 'git' option to collector registry
    git.js            # thin wrapper: load map + run queries -> structured output
```

### Lifecycle

```
git-map init     full scan - processes all history
       |
       v
  git-map.json   stored in .claude/ (platform-aware state dir)
       |
       v
git-map status   HEAD === analyzedUpTo? -> valid / stale
       |
       v
git-map update   process analyzedUpTo..HEAD only, merge into map
```

### Relationship to existing infrastructure

| Component | Role |
|-----------|------|
| `agent-core/lib/git-map/` | Core library (extractor, aggregator, queries) |
| `agent-core/lib/collectors/git.js` | Thin consumer - loads map, runs queries, returns structured output |
| `agent-core/lib/collectors/index.js` | Registry - adds `'git'` as a collector option |
| `agent-core/lib/repo-map/` | Sibling - provides AST/symbol data that git-map complements |
| `agent-core/lib/collectors/github.js` | Sibling - provides GitHub API data (issues, PRs, milestones) |

Consumers use it via the collector pattern:

```js
const { collectors } = require('@agentsys/lib');
const data = collectors.collect({
  collectors: ['github', 'docs', 'code', 'git']  // just add 'git'
});
```

## 3. Data Model (Static Map Schema)

```json
{
  "version": "1.0",
  "generated": "2026-03-14T10:00:00Z",
  "updated": "2026-03-14T12:00:00Z",
  "partial": false,

  "git": {
    "analyzedUpTo": "abc123def456",
    "totalCommitsAnalyzed": 1847,
    "firstCommitDate": "2024-01-15",
    "lastCommitDate": "2026-03-13",
    "scope": null,
    "shallow": false
  },

  "contributors": {
    "humans": {
      "alice": {
        "commits": 1180,
        "firstSeen": "2024-01-15",
        "lastSeen": "2026-03-13",
        "aiAssistedCommits": 180
      },
      "bob": {
        "commits": 390,
        "firstSeen": "2024-03-20",
        "lastSeen": "2026-03-10",
        "aiAssistedCommits": 45
      }
    },
    "bots": {
      "dependabot[bot]": { "commits": 230, "firstSeen": "2024-02-01", "lastSeen": "2026-03-13" }
    }
  },

  "fileActivity": {
    "src/core/engine.rs": {
      "changes": 47,
      "authors": ["alice", "bob"],
      "created": "2024-01-15",
      "lastChanged": "2026-03-12",
      "additions": 1200,
      "deletions": 800,
      "aiChanges": 12,
      "aiAdditions": 480,
      "aiDeletions": 320,
      "bugFixChanges": 8
    }
  },

  "dirActivity": {
    "src/core/": {
      "changes": 120,
      "authors": ["alice", "bob", "carol"],
      "aiChanges": 28
    },
    "src/tools/": {
      "changes": 85,
      "authors": ["bob"],
      "aiChanges": 40
    }
  },

  "coupling": {
    "src/core/engine.rs": {
      "src/core/engine_test.rs": { "cochanges": 38, "humanCochanges": 30, "aiCochanges": 8 },
      "src/config.rs": { "cochanges": 12, "humanCochanges": 10, "aiCochanges": 2 }
    }
  },

  "commitShape": {
    "sizeDistribution": { "tiny": 340, "small": 280, "medium": 150, "large": 60, "huge": 17 },
    "filesPerCommit": { "median": 3, "p90": 8 },
    "mergeCommits": 120
  },

  "conventions": {
    "prefixes": { "feat": 340, "fix": 280, "chore": 150, "docs": 77 },
    "style": "conventional",
    "usesScopes": true,
    "samples": ["feat(tools): add file search", "fix: handle empty input"]
  },

  "aiAttribution": {
    "attributed": 180,
    "heuristic": 45,
    "none": 775,
    "tools": {
      "claude": 120,
      "aider": 35,
      "cursor": 25,
      "unknown": 45
    }
  },

  "releases": {
    "tags": [
      { "tag": "v0.5.0", "commit": "abc123", "date": "2026-03-01", "commitsSince": 12 }
    ],
    "cadence": "monthly"
  },

  "renames": [
    { "from": "src/util.rs", "to": "src/helpers.rs", "commit": "def456", "date": "2026-01-20" }
  ],

  "deletions": [
    { "path": "src/old_parser.rs", "commit": "789abc", "date": "2026-02-15" }
  ]
}
```

### What's stored vs derived

**Stored (incrementally updatable with simple arithmetic):**

| Field | Update operation |
|-------|-----------------|
| contributors | `count += delta`, update lastSeen |
| fileActivity per file | `changes += delta`, update lastChanged, union authors |
| dirActivity per dir | `changes += delta`, union authors |
| coupling pairs | `cochanges += delta` (prune pairs below threshold of 3) |
| commitShape.sizeDistribution | `bucket[size] += 1` per new commit |
| conventions.prefixes | `prefix[type] += delta` |
| aiAttribution | `attributed += delta`, update tool counts |
| releases.tags | append new tags |
| renames, deletions | append new entries |
| bugFixChanges per file | `count += delta` (commit message starts with `fix`) |

**Derived at query time (pure JS over cached data, zero git commands):**

| Query | Computation |
|-------|-------------|
| busFactor | Sort humans by commits, find N covering 80% |
| hotspots | Filter fileActivity by lastChanged > cutoff, sort by changes |
| coldspots | Filter fileActivity by lastChanged < cutoff |
| ownership per dir | Max contributor from dirActivity.authors + fileActivity |
| coupling strength | cochanges / totalChanges for file pair |
| recentContributors | Filter contributors by lastSeen > cutoff |
| isActive | lastCommitDate > 30 days ago |
| unreleasedWork | Commits since last tag |
| aiRatio per file | aiChanges / changes |
| aiRatio repo-wide | sum(aiChanges) / sum(changes) |
| commitFrequency | totalCommits / time span |
| adjustedBusFactor | Bus factor weighted by aiAssistedCommits ratio |

## 4. Extractor

The extractor runs git commands and returns a raw delta. On init, it processes full history. On update, only `analyzedUpTo..HEAD`.

### Git commands (the only 4 needed)

```js
function extractDelta(basePath, range) {
  return {
    // Commits with authors, dates, messages, and trailers
    commits: execGit([
      'log', range,
      '--format=%H|%aN|%aE|%aI|%s%n%b%n---TRAILER---%n%(trailers:key=Co-authored-by,valueonly)%n---END---',
      '--no-merges'
    ]),

    // Files changed per commit with additions/deletions
    files: execGit([
      'log', range,
      '--format=%H',
      '--numstat',
      '--no-merges'
    ]),

    // Renames
    renames: execGit([
      'log', range,
      '--diff-filter=R', '--format=%H|%aI', '-M',
      '--name-status'
    ]),

    // Deletions
    deletions: execGit([
      'log', range,
      '--diff-filter=D', '--format=%H|%aI',
      '--name-only'
    ])
  };
}
```

### AI detection (runs per-commit during extraction)

Detection is string matching on data already being parsed. No extra git commands.

```js
function detectAI(commit) {
  const signals = { detected: false, tool: null, method: null };

  // 1. Trailer check (highest confidence)
  for (const trailer of commit.trailers) {
    if (AI_SIGNATURES.trailerEmails.some(e => trailer.includes(e))) {
      signals.detected = true;
      signals.tool = identifyTool(trailer);
      signals.method = 'trailer';
      return signals;
    }
  }

  // 2. Author email check
  if (AI_SIGNATURES.authorEmails.some(e => commit.authorEmail.includes(e))) {
    signals.detected = true;
    signals.tool = identifyToolFromEmail(commit.authorEmail);
    signals.method = 'author';
    return signals;
  }

  // 3. Author name pattern check
  if (AI_SIGNATURES.authorPatterns.some(p => new RegExp(p).test(commit.authorName))) {
    signals.detected = true;
    signals.tool = identifyToolFromAuthor(commit.authorName);
    signals.method = 'author-pattern';
    return signals;
  }

  // 4. Branch prefix check
  if (commit.branch && AI_SIGNATURES.branchPrefixes.some(p => commit.branch.startsWith(p))) {
    signals.detected = true;
    signals.tool = identifyToolFromBranch(commit.branch);
    signals.method = 'branch';
    return signals;
  }

  // 5. Message body check
  if (AI_SIGNATURES.messagePatterns.some(p => new RegExp(p).test(commit.body))) {
    signals.detected = true;
    signals.tool = identifyToolFromMessage(commit.body);
    signals.method = 'message';
    return signals;
  }

  return signals;
}
```

### AI Signature Registry (ai-signatures.json)

Updateable data file - no code changes needed when tools change attribution:

```json
{
  "version": "1.0",
  "updated": "2026-03-14",
  "trailerEmails": {
    "noreply@anthropic.com": "claude",
    "noreply@aider.chat": "aider",
    "cursoragent@cursor.com": "cursor",
    "agent@cursor.com": "cursor",
    "bot@devin.ai": "devin"
  },
  "authorEmails": {
    "no-reply@replit.com": "replit"
  },
  "authorPatterns": {
    "\\(aider\\)$": "aider",
    "\\[bot\\]$": "bot"
  },
  "branchPrefixes": {
    "copilot/": "copilot",
    "cursor/": "cursor"
  },
  "messagePatterns": {
    "Generated with Claude Code": "claude",
    "^aider: ": "aider"
  },
  "trailerNames": {
    "Claude": "claude",
    "Cursor": "cursor",
    "Copilot": "copilot",
    "GitHub Copilot": "copilot",
    "Codex": "codex",
    "Jules": "jules",
    "Devin": "devin",
    "aider": "aider"
  },
  "botAuthors": {
    "dependabot[bot]": "dependabot",
    "renovate[bot]": "renovate",
    "github-actions[bot]": "github-actions",
    "devin-ai-integration[bot]": "devin"
  }
}
```

### Detection coverage (honest assessment)

| Tool | Detectable | Method | Default state |
|------|-----------|--------|---------------|
| Claude Code | Yes | Trailer + body text | On by default, can be disabled |
| Aider | Yes | Author suffix + trailer | On by default, can be disabled |
| Cursor Agent | Yes | Trailer + branch prefix | On, hard to disable |
| Copilot Coding Agent | Yes | Author + branch prefix | On, hardcoded |
| Devin | Yes | Bot author | Always present |
| Codex CLI | Yes | Trailer | Recently added, configurable |
| Replit Agent | Yes | Author email | Always present |
| Copilot inline | No | No trace | Fundamentally undetectable |
| Windsurf/Codeium | No | No trace | No attribution |
| JetBrains AI | No | No trace | No attribution |
| Tabnine | No | No trace | No attribution |
| Cline/Continue | No | No trace | No attribution |
| Amazon Q | No | No trace | No attribution |

`aiRatio` in the map is a **lower bound**. Industry data shows 41-46% of code is AI-generated, but only autonomous agents and some pair tools leave traces. Inline completions are invisible.

### Noise filtering

Excluded from coupling and hotspot analysis:

```js
const NOISE_FILES = [
  /package-lock\.json$/,
  /yarn\.lock$/,
  /Cargo\.lock$/,
  /go\.sum$/,
  /pnpm-lock\.yaml$/,
  /\.min\.(js|css)$/,
  /dist\//,
  /build\//,
  /vendor\//
];
```

Merge commits excluded via `--no-merges` flag to prevent double-counting.

## 5. Aggregator

Merges a delta into the existing map. Pure arithmetic - no git commands.

```js
function mergeInto(existingMap, delta) {
  const map = structuredClone(existingMap);

  for (const commit of delta.commits) {
    // Update contributor counts
    const isBot = AI_SIGNATURES.botAuthors[commit.authorName];
    const aiSignals = detectAI(commit);
    const bucket = isBot ? 'bots' : 'humans';

    if (!map.contributors[bucket][commit.authorName]) {
      map.contributors[bucket][commit.authorName] = {
        commits: 0, firstSeen: commit.date, lastSeen: commit.date
      };
      if (bucket === 'humans') {
        map.contributors.humans[commit.authorName].aiAssistedCommits = 0;
      }
    }
    map.contributors[bucket][commit.authorName].commits++;
    map.contributors[bucket][commit.authorName].lastSeen = commit.date;

    if (bucket === 'humans' && aiSignals.detected) {
      map.contributors.humans[commit.authorName].aiAssistedCommits++;
    }

    // Update AI attribution counts
    if (aiSignals.detected) {
      map.aiAttribution.attributed++;
      const tool = aiSignals.tool || 'unknown';
      map.aiAttribution.tools[tool] = (map.aiAttribution.tools[tool] || 0) + 1;
    } else {
      map.aiAttribution.none++;
    }

    // Update commit shape
    const size = classifyCommitSize(commit.additions + commit.deletions);
    map.commitShape.sizeDistribution[size]++;

    // Update conventions
    const prefix = extractConventionalPrefix(commit.subject);
    if (prefix) map.conventions.prefixes[prefix] = (map.conventions.prefixes[prefix] || 0) + 1;

    // Update per-file activity
    for (const file of commit.files) {
      if (isNoise(file.path)) continue;

      if (!map.fileActivity[file.path]) {
        map.fileActivity[file.path] = {
          changes: 0, authors: [], created: commit.date,
          lastChanged: commit.date, additions: 0, deletions: 0,
          aiChanges: 0, aiAdditions: 0, aiDeletions: 0, bugFixChanges: 0
        };
      }
      const f = map.fileActivity[file.path];
      f.changes++;
      f.lastChanged = commit.date;
      f.additions += file.additions;
      f.deletions += file.deletions;
      if (!f.authors.includes(commit.authorName)) f.authors.push(commit.authorName);

      if (aiSignals.detected) {
        f.aiChanges++;
        f.aiAdditions += file.additions;
        f.aiDeletions += file.deletions;
      }
      if (prefix === 'fix') f.bugFixChanges++;
    }

    // Update coupling (co-occurrence within same commit)
    const filePaths = commit.files.map(f => f.path).filter(p => !isNoise(p));
    for (let i = 0; i < filePaths.length; i++) {
      for (let j = i + 1; j < filePaths.length; j++) {
        const a = filePaths[i] < filePaths[j] ? filePaths[i] : filePaths[j];
        const b = filePaths[i] < filePaths[j] ? filePaths[j] : filePaths[i];
        if (!map.coupling[a]) map.coupling[a] = {};
        if (!map.coupling[a][b]) map.coupling[a][b] = { cochanges: 0, humanCochanges: 0, aiCochanges: 0 };
        map.coupling[a][b].cochanges++;
        if (aiSignals.detected) {
          map.coupling[a][b].aiCochanges++;
        } else {
          map.coupling[a][b].humanCochanges++;
        }
      }
    }

    // Aggregate dir activity
    for (const file of commit.files) {
      const dir = dirname(file.path) + '/';
      if (!map.dirActivity[dir]) {
        map.dirActivity[dir] = { changes: 0, authors: [], aiChanges: 0 };
      }
      map.dirActivity[dir].changes++;
      if (!map.dirActivity[dir].authors.includes(commit.authorName)) {
        map.dirActivity[dir].authors.push(commit.authorName);
      }
      if (aiSignals.detected) map.dirActivity[dir].aiChanges++;
    }
  }

  // Merge renames and deletions
  map.renames.push(...delta.renames);
  map.deletions.push(...delta.deletions);

  // Prune low-signal coupling (below threshold of 3 cochanges)
  for (const [fileA, pairs] of Object.entries(map.coupling)) {
    for (const [fileB, counts] of Object.entries(pairs)) {
      if (counts.cochanges < 3) delete pairs[fileB];
    }
    if (Object.keys(pairs).length === 0) delete map.coupling[fileA];
  }

  // Update git metadata
  map.git.analyzedUpTo = delta.head;
  map.git.totalCommitsAnalyzed += delta.commits.length;
  map.git.lastCommitDate = delta.commits[delta.commits.length - 1]?.date || map.git.lastCommitDate;

  map.updated = new Date().toISOString();
  return map;
}
```

## 6. Query API

Pure functions over the cached map. No git commands. No LLM.

```js
const gitMap = require('@agentsys/lib').gitMap;
const map = gitMap.load(basePath);

// --- Health ---
gitMap.isActive(map)
// -> bool (commits in last 30 days)

gitMap.getHealth(map)
// -> { active, busFactor, commitFrequency, age, aiRatio }

gitMap.getBusFactor(map, { adjustForAI: false })
// -> number (humans covering 80% of commits)
// adjustForAI: true weights by aiAssistedCommits ratio

// --- Contributors ---
gitMap.getContributors(map, { months: 6 })
// -> [{ name, commits, pct, aiAssistedPct }] filtered by lastSeen

gitMap.getOwnership(map, 'src/core/')
// -> { primary: 'alice', pct: 78, contributors: [...], aiRatio: 0.23 }

// --- Files ---
gitMap.getHotspots(map, { months: 3, limit: 10 })
// -> [{ path, changes, authors, aiRatio, bugFixes }]

gitMap.getColdspots(map, { months: 6 })
// -> [{ path, lastChanged, daysSince }]

gitMap.getFileHistory(map, 'src/engine.rs')
// -> { changes, authors, created, lastChanged, aiRatio, bugFixes, ... }

gitMap.getCoupling(map, 'src/engine.rs', { humanOnly: false })
// -> [{ path, strength, cochanges, humanCochanges }]
// humanOnly: true excludes AI session correlation

// --- AI ---
gitMap.getAIRatio(map)
// -> { ratio: 0.18, attributed: 180, total: 1000, tools: {...} }
// NOTE: this is a lower bound

gitMap.getAIRatio(map, 'src/tools/')
// -> per-directory AI ratio

gitMap.getAIHotspots(map, { limit: 10 })
// -> files with highest AI ratio AND high churn (risk signal)

// --- Conventions ---
gitMap.getConventions(map)
// -> { style, prefixes, samples, aiAttribution }

gitMap.getCommitShape(map)
// -> { typicalSize, filesPerCommit, mergeCommitRatio }

// --- Releases ---
gitMap.getReleaseInfo(map)
// -> { cadence, lastRelease, unreleased, tags: [...] }

// --- Changes ---
gitMap.getRecentDeletions(map, { months: 3 })
// -> [{ path, date, commit }]

gitMap.getRecentRenames(map, { months: 3 })
// -> [{ from, to, date, commit }]
```

## 7. Consumers

### Who uses what

| Consumer | Fields consumed | Benefit |
|----------|----------------|---------|
| `/onboard` | Everything | Build mental model of unfamiliar codebase |
| `/can-i-help` | contributors, conventions, ownership, hotspots, releases | Guide OSS contributions |
| drift-detect | hotspots, churn, deletions, renames, ownership | Detect plan vs reality gaps |
| sync-docs | hotspots, deletions, renames, coupling (docs <-> code) | Find stale documentation |
| enhance | conventions, aiAttribution | Validate project patterns |
| next-task | hotspots, ownership, aiRatio | Assign work to right areas |
| perf | hotspots, churn, coupling | Identify optimization targets |
| code review | coupling, ownership, aiRatio, bugFixes | PR context and risk signals |

### Example: drift-detect integration

```js
const { collectors } = require('@agentsys/lib');
const data = collectors.collect({
  collectors: ['github', 'docs', 'code', 'git']
});

// drift-detect now gets git temporal data alongside GitHub API data
// "Module X has lost its primary author and churn increased 40%"
// "src/legacy/ hasn't been touched in 275 days but docs still reference it"
```

### Example: /onboard output

```
Project: opencode (Go TUI)
Active: Yes (daily commits)
Bus factor: 2 (alice: 65%, bob: 22%)
AI ratio: 18% detected (lower bound)
Convention: conventional commits (87% conformance)
Release cadence: monthly (last: v0.5.0, 12 unreleased commits)

Hot areas (last 3 months):
  src/core/     120 changes, 3 authors, 23% AI
  src/tools/     85 changes, 1 author, 47% AI  [WARN: single owner + high AI]

Cold areas (untouched 6+ months):
  src/legacy/   last modified 2025-06-12

Key couplings:
  engine.rs <-> engine_test.rs  (38 co-changes, 79% human)
  engine.rs <-> config.rs       (12 co-changes, 83% human)
```

## 8. Edge Cases

| Edge case | Detection | Handling |
|-----------|-----------|----------|
| Shallow clone | `git rev-parse --is-shallow-repository` | Mark `partial: true`, analyze available history |
| Force push | `git cat-file -t analyzedUpTo` fails | Full rebuild |
| Monorepo | User passes `scope: 'packages/lib/'` | Filter commits to scope path |
| Merge commits | N/A | Excluded via `--no-merges` |
| Lockfiles/generated | Regex patterns | Excluded from coupling and hotspot analysis |
| Bot commits | Author name in botAuthors registry | Tracked separately, excluded from human metrics |
| AI attribution disabled | No trailer/metadata | Falls into `none` category; aiRatio is lower bound |

## 9. Performance

### Init (full scan)

For a repo with 5000 commits: 4 git commands, each producing text output. Parsing and aggregation is O(commits * files_per_commit). Expected: < 10 seconds for most repos.

### Update (incremental)

For 50 new commits since last scan: same 4 git commands scoped to `analyzedUpTo..HEAD`. Expected: < 1 second.

### Map size

Estimated JSON size for a 5000-commit repo with 500 tracked files:
- fileActivity: ~50KB (100 bytes per file)
- coupling: ~20KB (sparse, threshold at 3)
- contributors: ~5KB
- Other: ~5KB
- Total: ~80KB compressed

### Caching

Cached at `{stateDir}/git-map.json` using the same platform-aware state directory as repo-map:
- Claude Code: `.claude/git-map.json`
- OpenCode: `config/.opencode/git-map.json`
- Codex: `.codex/git-map.json`

## 10. Implementation Plan

### Phase 1: Core (MVP)

- [ ] `extractor.js` - 4 git commands, parse output
- [ ] `ai-detect.js` + `ai-signatures.json` - explicit signal detection
- [ ] `aggregator.js` - merge delta into map
- [ ] `cache.js` - load/save/status (reuse repo-map pattern)
- [ ] `index.js` - init/update/status/load API
- [ ] `collectors/git.js` - thin wrapper for collector registry
- [ ] Tests for all modules

### Phase 2: Query Layer

- [ ] `queries.js` - all query functions
- [ ] Integration with collector registry in `collectors/index.js`
- [ ] Tests for query edge cases (empty map, single contributor, no tags)

### Phase 3: Consumers

- [ ] `/onboard` command using git-map + repo-map
- [ ] `/can-i-help` command using git-map + GitHub collector
- [ ] drift-detect integration
- [ ] sync-docs integration

### Phase 4: Enhancement

- [ ] Heuristic AI detection (commit message patterns vs repo baseline)
- [ ] Agent Trace integration (if adopted by ecosystem)
- [ ] Deeper coupling analysis (cross-directory coupling highlighting)

## 11. Research Foundation

This spec is informed by 91 sources across 3 research guides:

| Guide | Sources | Key contribution |
|-------|---------|-----------------|
| `git-history-analysis-developer-tools.md` | 24 | Core analysis techniques: hotspots, coupling, ownership, churn, CodeScene/GitClear/Hercules patterns |
| `ai-agent-commits-git-analysis-impact.md` | 25 | AI impact on metrics: 41-46% AI code, churn inflation, ownership breakdown, metric adjustments |
| `ai-commit-detection-forensics.md` | 42 | Detection methods: per-tool signatures, composite scoring, 97.2% F1-score fingerprinting study |

Key academic references:
- "Fingerprinting AI Coding Agents on GitHub" (arXiv:2601.17406) - 97.2% F1, 41 behavioral features
- "Your Code as a Crime Scene" (Adam Tornhill) - behavioral code analysis paradigm
- GitClear 2025 research - 211M lines, AI churn impact data
- Agent Trace specification (agent-trace.dev) - emerging attribution standard
