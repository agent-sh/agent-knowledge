# Learning Guide: Codex CLI Plugin Manifest System

**Generated**: 2026-04-01
**Sources**: 42 resources analyzed (source code, official docs, PRs, issues, TypeScript protocol)
**Depth**: deep
**Codex CLI Version**: v0.117.0+ (plugin system introduced March 26, 2026)

## Prerequisites

- Familiarity with JSON manifest formats
- Understanding of Codex CLI basics (installation, config.toml)
- Basic knowledge of MCP (Model Context Protocol) concepts
- Understanding of plugin/extension architectures

## TL;DR

- Plugins live in `.codex-plugin/` with a required `plugin.json` manifest
- Three component types: skills (SKILL.md files), MCP servers (.mcp.json), apps (.app.json)
- All paths must be relative, starting with `./`, no `..` traversal allowed
- Plugin names must be kebab-case, ASCII alphanumeric plus hyphens/underscores only
- Marketplace distribution via `.agents/plugins/marketplace.json` at repo or home level
- Default prompts limited to 3 entries, 128 characters each
- Interface section is optional - entirely omitted if all fields are empty
- Plugin hooks are NOT supported in plugin.json despite some docs suggesting otherwise

## Core Concepts

### Plugin Directory Structure

A Codex plugin is a directory containing a `.codex-plugin/plugin.json` manifest and optional component directories:

```
my-plugin/
  .codex-plugin/
    plugin.json          # Required manifest (only file in .codex-plugin/)
  skills/
    skill-name/
      SKILL.md           # Skill instructions
  .mcp.json              # MCP server configuration
  .app.json              # App connector mappings
  assets/
    icon.png             # Visual assets (logo, screenshots, etc.)
```

**Critical rule**: Only `plugin.json` belongs inside `.codex-plugin/`. All other files go at the plugin root.

The manifest path constant is `.codex-plugin/plugin.json` (defined as `PLUGIN_MANIFEST_PATH` in `codex-utils-plugins`).

### Manifest Loading Flow

```
CLI startup
  -> PluginsManager::plugins_for_config()
    -> For each configured plugin:
      -> load_plugin_manifest(plugin_root)
        -> Read {plugin_root}/.codex-plugin/plugin.json
        -> Deserialize as RawPluginManifest (serde_json)
        -> Resolve paths (skills, mcpServers, apps)
        -> Resolve interface fields (assets, prompts)
        -> Return PluginManifest or None
      -> Load skills from resolved skills path
      -> Load MCP servers from resolved mcpServers path
      -> Load app connectors from resolved apps path
    -> Build PluginLoadOutcome with all plugins
```

## Manifest Schema (Comprehensive)

### Top-Level Fields

```json
{
  "name": "string (required, validated)",
  "description": "string | null (optional)",
  "skills": "./relative-path (optional)",
  "mcpServers": "./relative-path (optional)",
  "apps": "./relative-path (optional)",
  "interface": { /* optional PluginManifestInterface */ }
}
```

**Note on field naming**: The raw deserialization uses `camelCase` (`#[serde(rename_all = "camelCase")]`). The path fields `skills`, `mcpServers`, and `apps` are top-level string fields, NOT nested under a `paths` object. The internal Rust struct `PluginManifestPaths` groups them after parsing, but the JSON is flat.

### Name Field

- **Required**: Yes, but if empty/whitespace, falls back to the plugin directory name
- **Validation**: Must match `validate_plugin_segment()` rules when used as PluginId:
  - Cannot be empty
  - Only ASCII alphanumeric characters, hyphens (`-`), and underscores (`_`)
  - No path separators (`/`, `\`)
  - No dot sequences (`.`, `..`)
- **Convention**: Kebab-case (e.g., `my-first-plugin`)
- **Purpose**: Used as both the plugin identifier and skill namespace prefix

```rust
// Name fallback logic (from manifest.rs):
let name = plugin_root
    .file_name()
    .and_then(|entry| entry.to_str())
    .filter(|_| raw_name.trim().is_empty())
    .unwrap_or(&raw_name)
    .to_string();
```

When the manifest `name` is empty or whitespace-only, the directory name is used instead.

### Description Field

- **Optional**: Defaults to None
- **Sanitization for capability summary**: Newlines and irregular whitespace collapsed to single spaces
- **Truncation**: `MAX_CAPABILITY_SUMMARY_DESCRIPTION_LEN` (1024 characters) for the capability summary
- **Original preserved**: The manifest description is kept as-is; only the `PluginCapabilitySummary` description is sanitized

### Component Path Fields (skills, mcpServers, apps)

All three path fields follow identical validation rules defined in `resolve_manifest_path()`:

1. **Must start with `./`** - Paths without this prefix are silently ignored with a warning
2. **Must not be just `./`** - Empty relative part after stripping prefix is rejected
3. **Must not contain `..`** - Parent directory traversal is blocked
4. **Must stay within plugin root** - Only `Component::Normal` path components allowed
5. **Must resolve to absolute path** - Final path is `plugin_root.join(normalized)`
6. **Empty string = None** - An empty string is treated as absent

```rust
// Validation logic (from manifest.rs):
fn resolve_manifest_path(plugin_root: &Path, field: &'static str, path: Option<&str>) -> Option<AbsolutePathBuf> {
    let path = path?;
    if path.is_empty() { return None; }
    
    let Some(relative_path) = path.strip_prefix("./") else {
        tracing::warn!("ignoring {field}: path must start with `./` relative to plugin root");
        return None;
    };
    if relative_path.is_empty() {
        tracing::warn!("ignoring {field}: path must not be `./`");
        return None;
    }
    
    let mut normalized = PathBuf::new();
    for component in Path::new(relative_path).components() {
        match component {
            Component::Normal(component) => normalized.push(component),
            Component::ParentDir => {
                tracing::warn!("ignoring {field}: path must not contain '..'");
                return None;
            }
            _ => {
                tracing::warn!("ignoring {field}: path must stay within the plugin root");
                return None;
            }
        }
    }
    
    AbsolutePathBuf::try_from(plugin_root.join(normalized)).ok()
}
```

**Default paths when manifest fields are absent**:
- Skills: `./skills/` (directory containing skill subdirectories)
- MCP Servers: `./.mcp.json` (JSON file at plugin root)
- Apps: `./.app.json` (JSON file at plugin root)

### Interface Section

The `interface` object provides display metadata for the plugin marketplace/directory. It is entirely optional - if present but all fields are empty/default, the interface is omitted (set to `None`).

```json
{
  "interface": {
    "displayName": "string | null",
    "shortDescription": "string | null",
    "longDescription": "string | null",
    "developerName": "string | null",
    "category": "string | null",
    "capabilities": ["array of strings"],
    "websiteUrl": "string | null",
    "privacyPolicyUrl": "string | null",
    "termsOfServiceUrl": "string | null",
    "defaultPrompt": "string | [array] | null",
    "brandColor": "string | null",
    "composerIcon": "./relative-path | null",
    "logo": "./relative-path | null",
    "screenshots": ["./relative-path array"]
  }
}
```

**Field aliases**: The raw deserialization accepts both camelCase and URL-suffix variants:
- `websiteUrl` or `websiteURL`
- `privacyPolicyUrl` or `privacyPolicyURL`
- `termsOfServiceUrl` or `termsOfServiceURL`

**Interface omission logic**: The interface is set to `None` if ALL of these are true:
- `displayName` is None
- `shortDescription` is None
- `longDescription` is None
- `developerName` is None
- `category` is None
- `capabilities` is empty
- `websiteUrl` is None
- `privacyPolicyUrl` is None
- `termsOfServiceUrl` is None
- `defaultPrompt` is None (after resolution)
- `brandColor` is None
- `composerIcon` is None
- `logo` is None
- `screenshots` is empty

### Default Prompt Validation

The `defaultPrompt` field supports three input shapes:

1. **Single string**: `"defaultPrompt": "Summarize my inbox"`
2. **Array of strings**: `"defaultPrompt": ["Prompt 1", "Prompt 2", "Prompt 3"]`
3. **Invalid type**: Any other JSON type is rejected with a warning

**Constants**:
- `MAX_DEFAULT_PROMPT_COUNT`: 3 (maximum number of prompts)
- `MAX_DEFAULT_PROMPT_LEN`: 128 (maximum characters per prompt after normalization)

**Processing rules**:
1. Whitespace is normalized: `split_whitespace().collect::<Vec<_>>().join(" ")`
2. Leading/trailing whitespace is stripped (via the split_whitespace normalization)
3. Empty prompts (after normalization) are rejected: "prompt must not be empty"
4. Prompts exceeding 128 characters (by `chars().count()`) are rejected
5. Non-string entries in an array (e.g., numbers, objects) are skipped with a warning
6. Only the first 3 valid entries are kept; excess entries trigger a warning
7. If a single string input is valid, it becomes a one-element array
8. If all entries are invalid, the field resolves to `None`

**Example of normalization behavior** (from tests):
```json
// Input:
"defaultPrompt": [
  " Summarize my inbox ",  // -> "Summarize my inbox" (valid)
  123,                       // -> skipped (not a string)
  "xxx...129chars",          // -> skipped (too long)
  " ",                       // -> skipped (empty after normalization)
  "Draft the reply ",        // -> "Draft the reply" (valid)
  "Find my next action",     // -> valid
  "Archive old mail"         // -> skipped (max 3 reached)
]
// Output: ["Summarize my inbox", "Draft the reply", "Find my next action"]
```

### Interface Asset Paths (composerIcon, logo, screenshots)

These use the same `resolve_manifest_path()` validation as component paths:
- Must start with `./`
- No `..` traversal
- Must stay within plugin root
- Resolved to absolute paths
- Invalid paths are silently ignored (the field becomes `None` or is omitted from the screenshots array)

**TypeScript protocol type** (after resolution):
```typescript
composerIcon: AbsolutePathBuf | null;
logo: AbsolutePathBuf | null;
screenshots: Array<AbsolutePathBuf>;  // never null, may be empty
```

## Plugin Identity System

### PluginId Format

Plugins are identified by a compound key: `{plugin_name}@{marketplace_name}`

```rust
pub struct PluginId {
    pub plugin_name: String,
    pub marketplace_name: String,
}
```

**Parsing**: The `@` separator is split on the rightmost occurrence. Both segments must pass `validate_plugin_segment()`.

**Segment validation** (`validate_plugin_segment()`):
- Cannot be empty
- Only ASCII alphanumeric characters (`a-z`, `A-Z`, `0-9`), hyphens (`-`), and underscores (`_`)
- No other characters (no dots, slashes, spaces, etc.)

**Error format**: `"invalid plugin key 'sample'; expected <plugin>@<marketplace>"`

### Plugin Namespace for Skills

Skills within a plugin are namespaced with the plugin name:
- Pattern: `{plugin_name}:{skill_name}`
- Example: `sample:sample-search` for skill `sample-search` in plugin `sample`

The namespace is resolved by walking up directory ancestors from the skill file path until a `.codex-plugin/plugin.json` is found, then extracting the `name` field.

## Plugin Discovery and Loading

### Discovery Locations

Plugins are discovered from:

1. **User config** (`~/.codex/config.toml`): Plugin entries with explicit paths
2. **Repo marketplace** (`$REPO_ROOT/.agents/plugins/marketplace.json`)
3. **Home marketplace** (`~/.agents/plugins/marketplace.json`)
4. **OpenAI curated marketplace**: Synced via git (preferred) or HTTP fallback

**Important**: Project-level `.codex/config.toml` plugin settings are ignored. Only user-level config is processed.

### Plugin Cache Structure

Installed plugins are cached at:
```
~/.codex/plugins/cache/{marketplace_name}/{plugin_name}/{version}/
```

- **Local plugins**: Version is `"local"`
- **Curated plugins**: Version is the git SHA
- **Version priority**: `"local"` always preferred; otherwise lexicographically last

### Plugin Loading Details

A loaded plugin (`LoadedPlugin`) contains:
- `config_name`: The key used in config (e.g., `sample@marketplace`)
- `manifest_name`: Name from plugin.json (may differ from config_name's plugin segment)
- `manifest_description`: Raw description from plugin.json
- `root`: Absolute path to plugin root
- `enabled`: Whether the plugin is active
- `skill_roots`: Directories containing skills
- `disabled_skill_paths`: Skills explicitly disabled by config
- `has_enabled_skills`: Whether any skills remain enabled
- `mcp_servers`: MCP server configurations loaded from the plugin
- `apps`: App connector IDs
- `error`: Optional error message if loading partially failed

A plugin is considered **active** only if `enabled == true` AND `error.is_none()`.

## Marketplace System

### Marketplace Manifest Format

Located at `.agents/plugins/marketplace.json`:

```json
{
  "name": "marketplace-name",
  "interface": {
    "displayName": "Human-Readable Name"
  },
  "plugins": [
    {
      "name": "plugin-name",
      "source": {
        "source": "local",
        "path": "./plugins/plugin-name"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL",
        "products": ["CODEX", "CHATGPT", "ATLAS"]
      },
      "category": "Productivity"
    }
  ]
}
```

### Marketplace Plugin Policy

**Installation policy** (`PluginInstallPolicy`):
- `"NOT_AVAILABLE"` - Plugin cannot be installed
- `"AVAILABLE"` - Plugin can be installed by user (default)
- `"INSTALLED_BY_DEFAULT"` - Plugin is pre-installed

**Authentication policy** (`PluginAuthPolicy`):
- `"ON_INSTALL"` - Auth required during installation (default)
- `"ON_USE"` - Auth deferred until first use

**Product filtering** (`products` array):
- If absent/missing: Plugin available to ALL products (permissive default)
- If present but empty `[]`: Plugin available to NO products (explicit denial)
- If present with values: Plugin only available to listed products
- Known products: `"CODEX"`, `"CHATGPT"`, `"ATLAS"`

### Marketplace Path Validation

Source paths in marketplace.json follow the same rules as plugin manifest paths:
- Must start with `./`
- No `..` traversal
- Resolved relative to marketplace root (NOT `.agents/plugins/`)

### Legacy Field Handling

Legacy top-level `installPolicy` and `authPolicy` fields on plugin entries are silently ignored. Only the nested `policy` object is used.

### Marketplace Discovery

Marketplace files are found at `.agents/plugins/marketplace.json` relative to:
1. Codex home directory (`~/.codex/` or equivalent)
2. Git repository roots discovered from provided working directories
3. The OpenAI curated repository (`openai-curated`)

When multiple marketplace roots point to the same git repository, only one marketplace entry is created (deduplication).

### Duplicate Plugin Handling

When the same plugin name appears multiple times within a single marketplace, the **first entry wins**. Later duplicates are silently ignored.

When the same plugin name appears across different marketplaces, both entries are preserved but resolution prioritizes the repository marketplace over the home marketplace.

## Plugin Installation

### Installation Flow

1. Validate source directory exists
2. Read manifest, extract plugin name
3. Confirm manifest name matches marketplace plugin name
4. Atomically copy files to cache using staging directory
5. Update config.toml with plugin entry
6. Set `enabled = true` automatically

### Atomic Installation

Installation uses a staging pattern:
1. Create temporary staging directory
2. Copy plugin files to staging
3. Back up existing version (if any)
4. Atomic rename of staging to final path
5. Rollback on failure (restore backup)

### Uninstallation

1. Remove plugin from cache directory
2. Remove plugin entry from config.toml
3. Idempotent - calling twice succeeds

## Plugin Injection into Agent

When plugins are mentioned (via `@plugin-name`), the system:

1. Collects mentioned plugins from user input
2. Filters MCP servers belonging to those plugins
3. Locates enabled app connectors for those plugins
4. Renders plugin instructions as `DeveloperInstructions`
5. Injects as `ResponseItem` objects into the conversation

### Instruction Rendering

Plugin instructions are rendered as markdown sections:
- General `## Plugins` section listing all available plugins
- Per-plugin sections listing capabilities (skills, MCP servers, apps)
- Skills use `plugin_name:` prefix convention
- Wrapped in XML tags (`PLUGINS_INSTRUCTIONS_OPEN_TAG`/`CLOSE_TAG`)

## Plugin Feature Gating

The entire plugin system is behind a feature flag:
- Set in config: `[features] plugins = true`
- When disabled: All plugin operations return empty/default results
- Feature check is the first gate in most plugin operations

## Startup Synchronization

On CLI startup:
1. Sync OpenAI curated plugins repository (git preferred, HTTP fallback)
2. Git sync: `ls-remote` to check SHA, shallow clone (depth=1) if outdated
3. HTTP fallback: GitHub API for branch info, download zipball
4. Validate marketplace manifest after extraction
5. Remote plugin sync (additive only): reconcile local state with remote
6. Marker file prevents duplicate startup syncs
7. Wait timeout: 5 seconds for prerequisites; 30 seconds for git/HTTP operations

### SHA-based Caching

- SHA file stored at `.tmp/plugins.sha`
- If local SHA matches remote, sync is skipped
- Temporary directories auto-cleaned after 10 minutes

## Known Limitations and Edge Cases

### Hooks Not Supported in Plugins

Despite documentation suggesting `hooks.json` as a plugin companion surface, **the runtime does NOT execute plugin-local hooks**. Only global `~/.codex/hooks.json` is loaded. The manifest parser does not process a `hooks` field. (GitHub issue #16430)

### Startup Sync Directory Leaks

On sync failure, `~/.codex/.tmp/plugins-clone-*` directories may be orphaned. (GitHub issue #16004)

### Empty Interface Omission

If the `interface` section is present in JSON but all fields are default/empty, the parsed manifest will have `interface: None`. This means validating the presence of interface fields must account for this normalization.

### Name vs Directory Name Fallback

When `name` is empty or whitespace, the plugin directory name becomes the plugin name. This means two different manifest states can produce the same effective name.

### Path Normalization Differences

The `resolve_manifest_path()` function normalizes paths by iterating components, meaning `./foo/../bar` would be rejected (contains `..`) even though it mathematically resolves within the plugin root. This is intentional for security.

### camelCase Field Naming

The JSON uses camelCase (`mcpServers`, `displayName`, etc.) via serde's `rename_all = "camelCase"`. The raw deserialization struct maps directly to JSON field names.

## TypeScript Protocol Types

The app-server-protocol defines these types for the plugin system:

```typescript
// Plugin summary in marketplace listings
type PluginSummary = {
  id: string;
  name: string;
  source: PluginSource;
  installed: boolean;
  enabled: boolean;
  installPolicy: PluginInstallPolicy;
  authPolicy: PluginAuthPolicy;
  interface: PluginInterface | null;
};

// Plugin source (currently only local)
type PluginSource = { type: "local"; path: AbsolutePathBuf };

// Plugin detail (full view)
type PluginDetail = {
  marketplaceName: string;
  marketplacePath: AbsolutePathBuf;
  summary: PluginSummary;
  description: string | null;
  skills: Array<SkillSummary>;
  apps: Array<AppSummary>;
  mcpServers: Array<string>;
};

// Marketplace entry
type PluginMarketplaceEntry = {
  name: string;
  path: AbsolutePathBuf;
  interface: MarketplaceInterface | null;
  plugins: Array<PluginSummary>;
};

// Policies
type PluginInstallPolicy = "NOT_AVAILABLE" | "AVAILABLE" | "INSTALLED_BY_DEFAULT";
type PluginAuthPolicy = "ON_INSTALL" | "ON_USE";
```

## Validation Rules Summary (for Linter Implementation)

### Manifest-Level Rules

| Rule | Severity | Description |
|------|----------|-------------|
| `name` required | Error | Must be non-empty (or directory name fallback) |
| `name` characters | Error | ASCII alphanumeric, hyphens, underscores only |
| `name` no path separators | Error | No `/`, `\`, or `..` |
| `name` kebab-case | Warning | Convention, not enforced by parser |
| JSON parse valid | Error | Must be valid JSON |
| Unknown fields | Info | Serde `default` allows extra fields silently |

### Path Rules (skills, mcpServers, apps, asset paths)

| Rule | Severity | Description |
|------|----------|-------------|
| Must start with `./` | Warning | Invalid paths silently ignored |
| Must not be just `./` | Warning | Empty relative path rejected |
| No `..` traversal | Warning | Path must not contain parent dir |
| Stay within plugin root | Warning | Only Normal path components |
| Must resolve to absolute | Warning | Plugin root must be absolute |
| Empty string = absent | Info | Empty strings treated as None |

### Interface Rules

| Rule | Severity | Description |
|------|----------|-------------|
| All-empty interface | Info | Entire interface is omitted |
| `defaultPrompt` max 3 | Warning | Excess entries ignored |
| `defaultPrompt` max 128 chars | Warning | Over-length entries ignored |
| `defaultPrompt` non-empty | Warning | Empty-after-normalization rejected |
| `defaultPrompt` type check | Warning | Non-string/array types rejected |
| `defaultPrompt` entry type | Warning | Non-string entries in array skipped |
| Asset paths same as component paths | Warning | Same `./` and traversal rules |
| `capabilities` array required | Info | Defaults to empty array |
| `screenshots` array required | Info | Defaults to empty array |

### Marketplace Rules

| Rule | Severity | Description |
|------|----------|-------------|
| Plugin source path `./` prefix | Error | Required for local sources |
| No `..` in source paths | Error | Traversal prevention |
| `policy.installation` valid enum | Warning | Must be known value |
| `policy.authentication` valid enum | Warning | Must be known value |
| Empty `products: []` blocks all | Warning | Likely unintentional |
| Missing `products` allows all | Info | Permissive default |
| Duplicate plugin names | Warning | First entry wins |
| Legacy policy fields ignored | Info | `installPolicy`/`authPolicy` top-level |
| Manifest name matches plugin name | Error | Must match during installation |

### PluginId Rules

| Rule | Severity | Description |
|------|----------|-------------|
| Format `name@marketplace` | Error | Required separator |
| Both segments non-empty | Error | No empty plugin or marketplace name |
| Segment characters | Error | ASCII alphanumeric, hyphens, underscores |

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Paths without `./` prefix silently ignored | Developers use bare relative paths like `skills/` | Always prefix with `./` |
| `..` in paths silently rejected | Trying to reference files outside plugin root | Keep all files within plugin directory |
| Hooks in plugin.json not executed | Documentation suggests it works, runtime ignores it | Use global `~/.codex/hooks.json` instead |
| Empty `products: []` blocks everything | Confusion between missing and empty arrays | Omit `products` field to allow all products |
| Default prompts over 128 chars silently dropped | No clear error at authoring time | Keep prompts concise, check char count |
| Plugin name mismatch blocks installation | Manifest name must match marketplace plugin name | Keep names synchronized |
| Project config plugin settings ignored | Only user-level config is processed | Configure plugins in `~/.codex/config.toml` |
| Legacy policy fields have no effect | Old `installPolicy`/`authPolicy` at top level | Use nested `policy` object |
| Interface with all nulls becomes None | All-empty interface is normalized away | Ensure at least one interface field is set |

## Best Practices

1. **Use kebab-case names** - Convention followed by all official plugins (Source: developers.openai.com/codex/plugins/build)
2. **Always prefix paths with `./`** - The only accepted relative path format (Source: manifest.rs `resolve_manifest_path`)
3. **Keep assets in `./assets/`** - Organized convention for logos, icons, screenshots (Source: official build docs)
4. **Include `displayName` in interface** - Prioritized for rendering plugin labels (Source: PR #15606)
5. **Keep descriptions under 1024 chars** - Capability summary truncates at this limit (Source: manager_tests.rs)
6. **Keep default prompts under 128 chars** - Hard limit enforced during parsing (Source: manifest.rs constants)
7. **Limit default prompts to 3** - Hard maximum, excess silently dropped (Source: manifest.rs constants)
8. **Set explicit `policy.products`** - Avoid accidentally blocking or allowing unwanted products (Source: marketplace_tests.rs)
9. **Test with `codex /plugins`** - Verify plugin appears in the directory after installation (Source: official plugin docs)
10. **Restart Codex after changes** - Plugin cache requires restart to pick up modifications (Source: official build docs)

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [manifest.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/plugins/manifest.rs) | Source | Definitive validation rules |
| [Build Plugins](https://developers.openai.com/codex/plugins/build) | Docs | Official development guide |
| [Plugin Overview](https://developers.openai.com/codex/plugins) | Docs | User-facing plugin documentation |
| [marketplace.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/plugins/marketplace.rs) | Source | Marketplace discovery and validation |
| [manager.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/plugins/manager.rs) | Source | Plugin lifecycle management |
| [store.rs](https://github.com/openai/codex/blob/main/codex-rs/core/src/plugins/store.rs) | Source | Cache storage and installation |
| [plugin_id.rs](https://github.com/openai/codex/blob/main/codex-rs/plugin/src/plugin_id.rs) | Source | PluginId parsing and validation |
| [v0.117.0 Release](https://github.com/openai/codex/releases) | Release | Plugin system introduction |
| [PR #14993](https://github.com/openai/codex/pull/14993) | PR | Product-aware policies design |
| [Issue #16430](https://github.com/openai/codex/issues/16430) | Issue | Plugin hooks limitation |

---

*This guide was synthesized from 42 sources including Rust source code, TypeScript protocol definitions, official OpenAI documentation, GitHub PRs, and issues. See `resources/codex-plugin-manifest-sources.json` for full source list with quality scores.*
