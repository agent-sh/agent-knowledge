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

## Trigger Phrases

Use this knowledge when user asks about:
- "How to use claude -p" -> ai-cli-non-interactive-programmatic-usage.md
- "Non-interactive AI CLI" -> ai-cli-non-interactive-programmatic-usage.md
- "Claude Code programmatic" -> ai-cli-non-interactive-programmatic-usage.md
- "Gemini CLI headless" -> ai-cli-non-interactive-programmatic-usage.md
- "Codex CLI quiet mode" -> ai-cli-non-interactive-programmatic-usage.md
- "OpenCode non-interactive" -> ai-cli-non-interactive-programmatic-usage.md
- "AI CLI session management" -> ai-cli-non-interactive-programmatic-usage.md
- "Claude Agent SDK" -> ai-cli-non-interactive-programmatic-usage.md
- "Consultant pattern AI" -> ai-cli-non-interactive-programmatic-usage.md
- "Piping to AI CLI" -> ai-cli-non-interactive-programmatic-usage.md
- "Structured output from AI CLI" -> ai-cli-non-interactive-programmatic-usage.md
- "AI in CI/CD" -> ai-cli-non-interactive-programmatic-usage.md
- "claude mcp serve" -> ai-cli-advanced-integration-patterns.md
- "MCP server mode" -> ai-cli-advanced-integration-patterns.md
- "AI CLI MCP integration" -> ai-cli-advanced-integration-patterns.md
- "Gemini CLI extensions" -> ai-cli-advanced-integration-patterns.md
- "Codex MCP" -> ai-cli-advanced-integration-patterns.md
- "Crush OpenCode" -> ai-cli-advanced-integration-patterns.md
- "Agent SDK hosting" -> ai-cli-advanced-integration-patterns.md
- "A2A protocol" -> ai-cli-advanced-integration-patterns.md
- "Cross-tool MCP" -> ai-cli-advanced-integration-patterns.md
- "AI CLI server mode" -> ai-cli-advanced-integration-patterns.md
- "Claude Code as service" -> ai-cli-advanced-integration-patterns.md
- "Plugin distribution patterns" -> skill-plugin-distribution-patterns.md
- "npm vs git submodules vs vendoring" -> skill-plugin-distribution-patterns.md
- "How to distribute CLI plugins" -> skill-plugin-distribution-patterns.md
- "Plugin registry architecture" -> skill-plugin-distribution-patterns.md
- "Extension marketplace design" -> skill-plugin-distribution-patterns.md
- "asdf mise plugin model" -> skill-plugin-distribution-patterns.md
- "Terraform provider registry" -> skill-plugin-distribution-patterns.md
- "Homebrew taps" -> skill-plugin-distribution-patterns.md
- "oh-my-zsh plugin system" -> skill-plugin-distribution-patterns.md
- "Plugin contract design" -> skill-plugin-distribution-patterns.md
- "Small ecosystem plugin strategy" -> skill-plugin-distribution-patterns.md
- "Convention-based plugin discovery" -> skill-plugin-distribution-patterns.md
- "monorepo packages" -> all-in-one-plus-modular-packages.md
- "meta-package pattern" -> all-in-one-plus-modular-packages.md
- "batteries included but removable" -> all-in-one-plus-modular-packages.md
- "npm workspaces" -> all-in-one-plus-modular-packages.md
- "pnpm workspaces" -> all-in-one-plus-modular-packages.md
- "Lerna monorepo" -> all-in-one-plus-modular-packages.md
- "Changesets versioning" -> all-in-one-plus-modular-packages.md
- "modular package architecture" -> all-in-one-plus-modular-packages.md
- "scoped npm packages" -> all-in-one-plus-modular-packages.md
- "plugin architecture" -> all-in-one-plus-modular-packages.md
- "CLI installer pattern" -> all-in-one-plus-modular-packages.md
- "degit tiged scaffolding" -> all-in-one-plus-modular-packages.md
- "dual CJS ESM" -> all-in-one-plus-modular-packages.md
- "package exports map" -> all-in-one-plus-modular-packages.md
- "independent versioning" -> all-in-one-plus-modular-packages.md
- "GitHub releases distribution" -> all-in-one-plus-modular-packages.md
- "AWS SDK v3 modular" -> all-in-one-plus-modular-packages.md
- "lodash per-method packages" -> all-in-one-plus-modular-packages.md
- "Babel plugin architecture" -> all-in-one-plus-modular-packages.md
- "Turborepo Nx Rush" -> all-in-one-plus-modular-packages.md
- "GitHub Projects v2" -> github-org-project-management.md
- "org-level project management" -> github-org-project-management.md
- "multi-repo GitHub board" -> github-org-project-management.md
- "GitHub Issue Types" -> github-org-project-management.md
- "GitHub sub-issues" -> github-org-project-management.md
- "GitHub roadmap project" -> github-org-project-management.md
- "addProjectV2ItemById" -> github-org-project-management.md
- "actions/add-to-project" -> github-org-project-management.md
- "Kubernetes SIG model" -> github-org-project-management.md
- "RFC process GitHub" -> github-org-project-management.md
- "cross-repo issue tracking" -> github-org-project-management.md
- "GitHub Projects automation" -> github-org-project-management.md
- "GitHub org structure" -> github-org-structure-patterns.md
- "GitHub .github repo" -> github-org-structure-patterns.md
- "org community health files" -> github-org-structure-patterns.md
- "reusable GitHub Actions workflows" -> github-org-structure-patterns.md
- "CODEOWNERS working groups" -> github-org-structure-patterns.md
- "GitHub org profile README" -> github-org-structure-patterns.md
- "developer tool org layout" -> github-org-structure-patterns.md
- "open source org patterns" -> github-org-structure-patterns.md
- "Slack login automation" -> browser-agent-auth-flows.md
- "Notion auth flow" -> browser-agent-auth-flows.md
- "AWS Console login" -> browser-agent-auth-flows.md
- "magic link detection" -> browser-agent-auth-flows.md
- "browser agent authentication" -> browser-agent-auth-flows.md
- "SSO SAML browser automation" -> browser-agent-auth-flows.md
- "CAPTCHA detection selectors" -> generic-auth-patterns-captcha-systems.md
- "MFA TOTP browser agent" -> browser-agent-auth-flows.md
- "AWS IAM Identity Center login" -> browser-agent-auth-flows.md
- "Slack workspace signin" -> browser-agent-auth-flows.md
- "Notion token_v2 cookie" -> browser-agent-auth-flows.md
- "AWS console session detection" -> browser-agent-auth-flows.md
- "X Twitter login automation" -> social-platform-auth-flows.md
- "Twitter auth flow" -> social-platform-auth-flows.md
- "X multi-step login" -> social-platform-auth-flows.md
- "Arkose Labs FunCAPTCHA" -> generic-auth-patterns-captcha-systems.md
- "Reddit login automation" -> social-platform-auth-flows.md
- "old reddit vs new reddit login" -> social-platform-auth-flows.md
- "Discord login automation" -> social-platform-auth-flows.md
- "Discord phone verification" -> social-platform-auth-flows.md
- "Discord hCaptcha" -> generic-auth-patterns-captcha-systems.md
- "LinkedIn login automation" -> social-platform-auth-flows.md
- "LinkedIn email verification loop" -> social-platform-auth-flows.md
- "li_at cookie" -> social-platform-auth-flows.md
- "auth_token ct0 cookie" -> social-platform-auth-flows.md
- "social media CAPTCHA detection" -> generic-auth-patterns-captcha-systems.md
- "social platform anti-bot" -> social-platform-auth-flows.md
- "GDPR cookie consent banner login" -> generic-auth-patterns-captcha-systems.md
- "Google login automation" -> google-microsoft-auth-flows.md
- "Google auth flow selectors" -> google-microsoft-auth-flows.md
- "Google reCAPTCHA login" -> generic-auth-patterns-captcha-systems.md
- "Google 2FA TOTP" -> google-microsoft-auth-flows.md
- "Google Workspace SSO" -> google-microsoft-auth-flows.md
- "SAPISID cookie" -> google-microsoft-auth-flows.md
- "Google account chooser" -> generic-auth-patterns-captcha-systems.md
- "Microsoft login automation" -> google-microsoft-auth-flows.md
- "Microsoft Entra ID login" -> google-microsoft-auth-flows.md
- "Azure AD login flow" -> google-microsoft-auth-flows.md
- "ESTSAUTH cookie" -> google-microsoft-auth-flows.md
- "Microsoft passwordless" -> generic-auth-patterns-captcha-systems.md
- "Microsoft conditional access" -> google-microsoft-auth-flows.md
- "Microsoft account picker" -> generic-auth-patterns-captcha-systems.md
- "Azure AD B2C" -> google-microsoft-auth-flows.md
- "Microsoft MFA Authenticator" -> google-microsoft-auth-flows.md
- "login.microsoftonline.com" -> google-microsoft-auth-flows.md
- "accounts.google.com selectors" -> google-microsoft-auth-flows.md
- "Google verify it's you" -> google-microsoft-auth-flows.md
- "Microsoft KMSI stay signed in" -> google-microsoft-auth-flows.md
- "federated auth redirect" -> google-microsoft-auth-flows.md
- "home realm discovery" -> google-microsoft-auth-flows.md
- "GitHub login automation" -> auth-flows-github-gitlab-atlassian.md
- "GitLab login CAPTCHA" -> auth-flows-github-gitlab-atlassian.md
- "Atlassian auth flow" -> auth-flows-github-gitlab-atlassian.md
- "Arkose Labs GitHub" -> auth-flows-github-gitlab-atlassian.md
- "GitHub 2FA selectors" -> auth-flows-github-gitlab-atlassian.md
- "GitLab _gitlab_session cookie" -> auth-flows-github-gitlab-atlassian.md
- "Atlassian multi-step login" -> auth-flows-github-gitlab-atlassian.md
- "GitHub Enterprise auth" -> auth-flows-github-gitlab-atlassian.md
- "Bitbucket login" -> auth-flows-github-gitlab-atlassian.md
- "auth success detection providers" -> generic-auth-patterns-captcha-systems.md
- "SSO redirect detection" -> auth-flows-github-gitlab-atlassian.md
- "WebAuthn passkey automation" -> generic-auth-patterns-captcha-systems.md
- "providers.json auth config" -> auth-flows-github-gitlab-atlassian.md
- "reCAPTCHA selectors" -> generic-auth-patterns-captcha-systems.md
- "hCaptcha selectors" -> generic-auth-patterns-captcha-systems.md
- "Cloudflare Turnstile" -> generic-auth-patterns-captcha-systems.md
- "PerimeterX HUMAN" -> generic-auth-patterns-captcha-systems.md
- "AWS WAF CAPTCHA" -> generic-auth-patterns-captcha-systems.md
- "cookie consent dismiss" -> generic-auth-patterns-captcha-systems.md
- "OneTrust CookieBot TrustArc" -> generic-auth-patterns-captcha-systems.md
- "browser fingerprinting stealth" -> generic-auth-patterns-captcha-systems.md
- "navigator.webdriver detection" -> generic-auth-patterns-captcha-systems.md
- "playwright stealth" -> generic-auth-patterns-captcha-systems.md
- "bot detection automation" -> generic-auth-patterns-captcha-systems.md
- "email-first login flow" -> generic-auth-patterns-captcha-systems.md
- "magic link flow detection" -> generic-auth-patterns-captcha-systems.md
- "social login buttons detection" -> generic-auth-patterns-captcha-systems.md
- "account picker detection" -> generic-auth-patterns-captcha-systems.md
- "trust this device prompt" -> generic-auth-patterns-captcha-systems.md
- "auth error detection" -> generic-auth-patterns-captcha-systems.md
- "wrong password detection" -> generic-auth-patterns-captcha-systems.md
- "account locked detection" -> generic-auth-patterns-captcha-systems.md
- "Playwright storageState" -> generic-auth-patterns-captcha-systems.md
- "waitForURL vs waitForNavigation" -> generic-auth-patterns-captcha-systems.md
- "multi-domain redirect auth" -> generic-auth-patterns-captcha-systems.md
- "SPA auth success detection" -> generic-auth-patterns-captcha-systems.md
- "Cursor rules for AI" -> cursor-ide-memory-context.md
- ".cursorrules file" -> cursor-ide-memory-context.md
- ".cursor/rules directory" -> cursor-ide-memory-context.md
- "Cursor project memory" -> cursor-ide-memory-context.md
- "Cursor MDC frontmatter" -> cursor-ide-memory-context.md
- "AGENTS.md Cursor" -> cursor-ide-memory-context.md
- "Cursor notepads" -> cursor-ide-memory-context.md
- "Cursor context window" -> cursor-ide-memory-context.md
- "Cursor codebase indexing" -> cursor-ide-memory-context.md
- "Cursor SKILL.md" -> cursor-ide-memory-context.md
- "Cursor session memory" -> cursor-ide-memory-context.md
- "Cursor hooks.json" -> cursor-ide-memory-context.md
- "cursorignore" -> cursor-ide-memory-context.md
- "Cursor Max Mode" -> cursor-ide-memory-context.md
- "Cursor alwaysApply" -> cursor-ide-memory-context.md
- "multi-product docs" -> multi-product-org-docs.md
- "documentation architecture" -> multi-product-org-docs.md
- "docs site structure" -> multi-product-org-docs.md
- "Docusaurus multi-instance" -> multi-product-org-docs.md
- "Starlight Astro docs" -> multi-product-org-docs.md
- "Mintlify docs" -> multi-product-org-docs.md
- "VitePress multi-sidebar" -> multi-product-org-docs.md
- "developer portal" -> multi-product-org-docs.md
- "plugin catalog page" -> multi-product-org-docs.md
- "marketplace page design" -> multi-product-org-docs.md
- "docs SEO multi-product" -> multi-product-org-docs.md
- "hub and spoke docs" -> multi-product-org-docs.md
- "unified docs vs separate sites" -> multi-product-org-docs.md
- "llms.txt AI discoverability" -> multi-product-org-docs.md
- "GitHub Pages org site docs" -> multi-product-org-docs.md
- "HashiCorp developer portal" -> multi-product-org-docs.md
- "Supabase docs structure" -> multi-product-org-docs.md
- "agent-sh website" -> multi-product-org-docs.md
- "agnix docs separate site" -> multi-product-org-docs.md
- "plugin catalog generation" -> multi-product-org-docs.md
- "cross-product navigation" -> multi-product-org-docs.md
- "product switcher docs" -> multi-product-org-docs.md
- "Diataxis framework" -> multi-product-org-docs.md
- "Pagefind search" -> multi-product-org-docs.md
- "docs-as-code" -> multi-product-org-docs.md
- "ACP protocol" -> acp-with-codex-gemini-copilot-claude.md
- "Agent Communication Protocol" -> acp-with-codex-gemini-copilot-claude.md
- "A2A protocol" -> acp-with-codex-gemini-copilot-claude.md
- "agent to agent communication" -> acp-with-codex-gemini-copilot-claude.md
- "cross AI tool communication" -> acp-with-codex-gemini-copilot-claude.md
- "MCP vs ACP vs A2A" -> acp-with-codex-gemini-copilot-claude.md
- "Codex CLI agent protocol" -> acp-with-codex-gemini-copilot-claude.md
- "Gemini CLI agent protocol" -> acp-with-codex-gemini-copilot-claude.md
- "Copilot agent communication" -> acp-with-codex-gemini-copilot-claude.md
- "Claude Code agent teams" -> acp-with-codex-gemini-copilot-claude.md
- "Claude Agent SDK multi-agent" -> acp-with-codex-gemini-copilot-claude.md
- "sequential thinking MCP" -> acp-with-codex-gemini-copilot-claude.md
- "MCP bridge server" -> acp-with-codex-gemini-copilot-claude.md
- "BeeAI Agent Stack" -> acp-with-codex-gemini-copilot-claude.md
- "Kiro ACP" -> acp-with-codex-gemini-copilot-claude.md
- "Cross-tool MCP bridges" -> acp-with-codex-gemini-copilot-claude.md
- "ACP SDK" -> acp-with-codex-gemini-copilot-claude.md
- "A2A SDK" -> acp-with-codex-gemini-copilot-claude.md
- "agent interoperability" -> acp-with-codex-gemini-copilot-claude.md
- "Agent Cards A2A" -> acp-with-codex-gemini-copilot-claude.md
- "claude mcp serve cross-tool" -> acp-with-codex-gemini-copilot-claude.md
- "Kiro supervised autopilot" -> kiro-supervised-autopilot.md
- "Kiro specs hooks" -> kiro-supervised-autopilot.md

## Quick Lookup

| Keyword | Guide |
|---------|-------|
| claude -p, headless, non-interactive | ai-cli-non-interactive-programmatic-usage.md |
| gemini cli, gemini -p | ai-cli-non-interactive-programmatic-usage.md |
| codex cli, codex -q | ai-cli-non-interactive-programmatic-usage.md |
| opencode, opencode -p | ai-cli-non-interactive-programmatic-usage.md |
| agent sdk, claude-agent-sdk | ai-cli-non-interactive-programmatic-usage.md |
| session, resume, continue | ai-cli-non-interactive-programmatic-usage.md |
| json output, structured output | ai-cli-non-interactive-programmatic-usage.md |
| mcp, model context protocol | ai-cli-non-interactive-programmatic-usage.md |
| subprocess, spawn, piping | ai-cli-non-interactive-programmatic-usage.md |
| consultant pattern | ai-cli-non-interactive-programmatic-usage.md |
| claude mcp serve, mcp server | ai-cli-advanced-integration-patterns.md |
| gemini extensions, a2a protocol | ai-cli-advanced-integration-patterns.md |
| codex shell-tool-mcp | ai-cli-advanced-integration-patterns.md |
| crush, opencode successor | ai-cli-advanced-integration-patterns.md |
| agent sdk hosting, containers | ai-cli-advanced-integration-patterns.md |
| cross-tool mcp, tool integration | ai-cli-advanced-integration-patterns.md |
| remote agents, long-running | ai-cli-advanced-integration-patterns.md |
| plugin distribution, extension marketplace | skill-plugin-distribution-patterns.md |
| npm packages, vendoring, git submodules | skill-plugin-distribution-patterns.md |
| terraform registry, homebrew taps | skill-plugin-distribution-patterns.md |
| asdf plugins, mise backends | skill-plugin-distribution-patterns.md |
| plugin contract, plugin protocol | skill-plugin-distribution-patterns.md |
| oh-my-zsh, zsh plugins, vim plugins | skill-plugin-distribution-patterns.md |
| vscode extensions, grafana plugins | skill-plugin-distribution-patterns.md |
| small ecosystem, plugin discovery | skill-plugin-distribution-patterns.md |
| monorepo, workspaces, multi-package | all-in-one-plus-modular-packages.md |
| lerna, changesets, turborepo, nx, rush | all-in-one-plus-modular-packages.md |
| meta-package, re-export, barrel | all-in-one-plus-modular-packages.md |
| lodash, aws-sdk, babel, eslint, angular | all-in-one-plus-modular-packages.md |
| plugin architecture, core + plugins | all-in-one-plus-modular-packages.md |
| degit, tiged, create-react-app, scaffolding | all-in-one-plus-modular-packages.md |
| npm publish, semantic-release, np | all-in-one-plus-modular-packages.md |
| CJS, ESM, dual module, exports map | all-in-one-plus-modular-packages.md |
| cargo workspaces, go modules, pip extras | all-in-one-plus-modular-packages.md |
| github releases, binary distribution | all-in-one-plus-modular-packages.md |
| github projects v2, projects board | github-org-project-management.md |
| issue types, sub-issues, github hierarchy | github-org-project-management.md |
| cross-repo tracking, org project | github-org-project-management.md |
| graphql addProjectV2ItemById | github-org-project-management.md |
| actions/add-to-project, auto-add | github-org-project-management.md |
| kubernetes SIG, KEP, enhancement proposals | github-org-project-management.md |
| rust RFC, working groups | github-org-project-management.md |
| astro roadmap, withastro | github-org-project-management.md |
| github public roadmap, shipped label | github-org-project-management.md |
| iteration fields, sprint planning github | github-org-project-management.md |
| label sync, org labels | github-org-project-management.md |
| .github repo, org defaults, community health | github-org-structure-patterns.md |
| CODEOWNERS, working groups, wg-tauri | github-org-structure-patterns.md |
| reusable workflows, workflow_call | github-org-structure-patterns.md |
| org profile README, pinned repos | github-org-structure-patterns.md |
| bun org, deno org, astro org | github-org-structure-patterns.md |
| AI contribution policy, AI disclosure | github-org-structure-patterns.md |
| just commands, justfile | github-org-structure-patterns.md |
| starter workflows, workflow-templates | github-org-structure-patterns.md |
| github pages org site | github-org-structure-patterns.md |
| verified domain github | github-org-structure-patterns.md |
| slack login, slack magic link, slack auth | browser-agent-auth-flows.md |
| notion login, notion magic link, token_v2 | browser-agent-auth-flows.md |
| aws console login, aws signin, IAM login | browser-agent-auth-flows.md |
| aws sso, iam identity center, awsapps | browser-agent-auth-flows.md |
| mfa totp, 2fa browser, hardware mfa | browser-agent-auth-flows.md |
| sso saml redirect, idp detection | browser-agent-auth-flows.md |
| enterprise grid, slack enterprise sso | browser-agent-auth-flows.md |
| aws govcloud, cross-account, switch role | browser-agent-auth-flows.md |
| x twitter login, twitter auth, x.com login | social-platform-auth-flows.md |
| reddit login, reddit auth, old reddit | social-platform-auth-flows.md |
| discord login, discord auth, discord token | social-platform-auth-flows.md |
| discord phone verification | social-platform-auth-flows.md |
| linkedin login, linkedin auth, li_at | social-platform-auth-flows.md |
| linkedin email verification, linkedin checkpoint | social-platform-auth-flows.md |
| auth_token, ct0, csrf token social | social-platform-auth-flows.md |
| social media bot detection, anti-bot | social-platform-auth-flows.md |
| reddit_session, token_v2 reddit | social-platform-auth-flows.md |
| discord localstorage token | social-platform-auth-flows.md |
| jsessionid linkedin csrf | social-platform-auth-flows.md |
| google login, accounts.google.com, google auth | google-microsoft-auth-flows.md |
| google 2fa, google totp, google prompt | google-microsoft-auth-flows.md |
| google workspace sso, google saml | google-microsoft-auth-flows.md |
| SAPISID, SID, HSID, google cookies | google-microsoft-auth-flows.md |
| microsoft login, login.microsoftonline.com | google-microsoft-auth-flows.md |
| entra id, azure ad, microsoft entra | google-microsoft-auth-flows.md |
| ESTSAUTH, microsoft cookies | google-microsoft-auth-flows.md |
| conditional access, AADSTS error | google-microsoft-auth-flows.md |
| azure ad b2c, b2clogin.com | google-microsoft-auth-flows.md |
| microsoft mfa, microsoft 2fa | google-microsoft-auth-flows.md |
| kmsi, stay signed in microsoft | google-microsoft-auth-flows.md |
| federated auth, home realm discovery, HRD | google-microsoft-auth-flows.md |
| google verify, unusual activity google | google-microsoft-auth-flows.md |
| microsoft proof up, mfa registration | google-microsoft-auth-flows.md |
| github login, logged_in cookie | auth-flows-github-gitlab-atlassian.md |
| gitlab login, _gitlab_session | auth-flows-github-gitlab-atlassian.md |
| atlassian login, id.atlassian.com | auth-flows-github-gitlab-atlassian.md |
| bitbucket login, bb_session | auth-flows-github-gitlab-atlassian.md |
| recaptcha gitlab | auth-flows-github-gitlab-atlassian.md |
| 2fa selectors, totp automation | auth-flows-github-gitlab-atlassian.md |
| saml sso redirect, oauth callback | auth-flows-github-gitlab-atlassian.md |
| github enterprise server auth, GHES | auth-flows-github-gitlab-atlassian.md |
| gitlab self-hosted auth, ldap | auth-flows-github-gitlab-atlassian.md |
| atlassian data center, jira login | auth-flows-github-gitlab-atlassian.md |
| providers.json, auth-providers | auth-flows-github-gitlab-atlassian.md |
| cloud.session.token, atlassian cookie | auth-flows-github-gitlab-atlassian.md |
| recaptcha v2, recaptcha v3, recaptcha enterprise | generic-auth-patterns-captcha-systems.md |
| hcaptcha selectors, h-captcha div | generic-auth-patterns-captcha-systems.md |
| cloudflare turnstile, cf-turnstile | generic-auth-patterns-captcha-systems.md |
| arkose labs, funcaptcha, fc-iframe | generic-auth-patterns-captcha-systems.md |
| aws waf captcha, awswaf | generic-auth-patterns-captcha-systems.md |
| perimeterx, human security, px-captcha | generic-auth-patterns-captcha-systems.md |
| captcha detection function | generic-auth-patterns-captcha-systems.md |
| cookie consent, gdpr banner, onetrust | generic-auth-patterns-captcha-systems.md |
| cookiebot, trustarc, quantcast | generic-auth-patterns-captcha-systems.md |
| navigator.webdriver, stealth, fingerprint | generic-auth-patterns-captcha-systems.md |
| playwright stealth, bot detection | generic-auth-patterns-captcha-systems.md |
| email-first flow, magic link, passkey | generic-auth-patterns-captcha-systems.md |
| social login buttons, sign in with google | generic-auth-patterns-captcha-systems.md |
| account picker, account chooser | generic-auth-patterns-captcha-systems.md |
| trust device, remember browser, tos gate | generic-auth-patterns-captcha-systems.md |
| auth success detection, multi-signal | generic-auth-patterns-captcha-systems.md |
| spa auth, dom mutation observer | generic-auth-patterns-captcha-systems.md |
| wrong password, account locked, auth error | generic-auth-patterns-captcha-systems.md |
| storageState, context.cookies | generic-auth-patterns-captcha-systems.md |
| waitForURL, waitForLoadState, networkidle | generic-auth-patterns-captcha-systems.md |
| multi-domain redirect, auth redirect chain | generic-auth-patterns-captcha-systems.md |
| auth timeout strategy, human interaction | generic-auth-patterns-captcha-systems.md |
| .cursorrules, cursor rules, cursor project rules | cursor-ide-memory-context.md |
| AGENTS.md cursor, cursor agents.md | cursor-ide-memory-context.md |
| .cursor/rules, MDC frontmatter, alwaysApply | cursor-ide-memory-context.md |
| cursor memory, cursor context, cursor project context | cursor-ide-memory-context.md |
| cursor notepads, cursor session memory | cursor-ide-memory-context.md |
| .cursorignore, cursorindexingignore | cursor-ide-memory-context.md |
| cursor codebase indexing, cursor semantic search | cursor-ide-memory-context.md |
| SKILL.md cursor, cursor skills | cursor-ide-memory-context.md |
| cursor hooks, sessionStart hook | cursor-ide-memory-context.md |
| cursor max mode, cursor context window | cursor-ide-memory-context.md |
| cursor team rules, cursor user rules | cursor-ide-memory-context.md |
| multi-product docs, documentation architecture | multi-product-org-docs.md |
| Docusaurus, Starlight, Mintlify, VitePress | multi-product-org-docs.md |
| developer portal, docs site structure | multi-product-org-docs.md |
| plugin catalog, marketplace page | multi-product-org-docs.md |
| docs SEO, llms.txt, AI discoverability | multi-product-org-docs.md |
| hub-and-spoke, unified portal | multi-product-org-docs.md |
| HashiCorp, Supabase, Vercel docs | multi-product-org-docs.md |
| product switcher, cross-product navigation | multi-product-org-docs.md |
| Diataxis, docs-as-code | multi-product-org-docs.md |
| Pagefind, Algolia DocSearch | multi-product-org-docs.md |
| agent-sh website, agnix docs | multi-product-org-docs.md |
| GitBook, ReadMe, Nextra | multi-product-org-docs.md |
| GitHub Pages docs hosting | multi-product-org-docs.md |
| Cloudflare Pages, Vercel hosting | multi-product-org-docs.md |
| ACP, Agent Communication Protocol | acp-with-codex-gemini-copilot-claude.md |
| A2A, Agent2Agent, agent-to-agent | acp-with-codex-gemini-copilot-claude.md |
| ACP vs MCP vs A2A | acp-with-codex-gemini-copilot-claude.md |
| BeeAI, Agent Stack, agent interop | acp-with-codex-gemini-copilot-claude.md |
| agent teams, subagents, multi-agent | acp-with-codex-gemini-copilot-claude.md |
| MCP bridge, Gemini bridge, OpenAI bridge | acp-with-codex-gemini-copilot-claude.md |
| sequential thinking, cross-AI reasoning | acp-with-codex-gemini-copilot-claude.md |
| Agent Cards, agent discovery | acp-with-codex-gemini-copilot-claude.md |
| claude mcp serve, claude as MCP server | acp-with-codex-gemini-copilot-claude.md |
| acp-sdk, a2a-sdk | acp-with-codex-gemini-copilot-claude.md |
| Kiro, supervised autopilot, specs | kiro-supervised-autopilot.md |

## How to Use

1. Check if user question matches a topic
2. Read the relevant guide file
3. Answer based on synthesized knowledge
4. Cite the guide if user asks for sources
