---
name: apm-packaging
description: >
  How to structure APM packages correctly -- manifest schema (apm.yml), .apm/ directory
  layout, primitive types (instructions, skills, prompts, agents, hooks, plugins), SKILL.md
  format, and naming rules. Use whenever creating, structuring, or validating an APM package,
  writing apm.yml manifests, mapping skills or agents to APM primitives, or setting up a
  project for distribution via APM. Also use when the user mentions apm.yml, .apm/ directory,
  package primitives, SKILL.md, or agent package structure.
---

# APM Packaging

APM (Agent Package Manager) is Microsoft's open-source dependency manager for AI agent
configuration. It manages primitives across multiple AI tools: GitHub Copilot, Claude Code,
Cursor, OpenCode, Codex CLI, and Gemini.

This skill covers package structure, manifest schema, primitive types, and naming rules.

## Quick Reference

### Minimal Package

```
my-package/
├── apm.yml          # REQUIRED: package manifest
└── .apm/            # primitives directory
    ├── instructions/
    ├── prompts/
    ├── agents/
    ├── skills/
    └── hooks/
```

### Minimal apm.yml

```yaml
name: my-package
version: 1.0.0
```

Only `name` and `version` are required. Everything else is optional.

## Package Detection

APM auto-detects package type from files present:

| Has | Type | Behavior |
|-----|------|----------|
| `apm.yml` only | APM Package | Standard APM primitives |
| `SKILL.md` only | Claude Skill | Auto-generates apm.yml on install |
| `plugin.json` only | Plugin | Synthesizes apm.yml from plugin metadata |
| `hooks/*.json` only | Hook Package | Hook handlers only |
| `apm.yml` + `SKILL.md` | Hybrid | Both APM primitives and skill |
| `apm.yml` + `plugin.json` | Hybrid Plugin | APM deps + plugin export |

## apm.yml Manifest Schema

Version 0.1 (Working Draft, 2026-03-06). Format: YAML 1.2.

### Top-Level Fields

```yaml
name:          <string>                  # REQUIRED
version:       <string>                  # REQUIRED (semver: ^d+.d+.d+)
description:   <string>                  # optional
author:        <string>                  # optional
license:       <string>                  # optional (SPDX, e.g. MIT)
target:        <enum>                    # optional (auto-detected)
type:          <enum>                    # optional
scripts:       <map<string, string>>     # optional
dependencies:                            # optional
  apm:         <list<ApmDependency>>
  mcp:         <list<McpDependency>>
devDependencies:                         # optional
  apm:         <list<ApmDependency>>
  mcp:         <list<McpDependency>>
compilation:   <CompilationConfig>       # optional
```

### target Field

Controls which output targets are generated during compilation.

| Value | Effect |
|-------|--------|
| `vscode` | Emits AGENTS.md (alias: `agents`) |
| `claude` | Emits CLAUDE.md |
| `codex` | Emits AGENTS.md, deploys skills to .agents/skills/, agents to .codex/agents/ |
| `all` | Both vscode and claude targets |

Auto-detection: `vscode` if `.github/` exists, `claude` if `.claude/` exists, `all` if both,
`minimal` (AGENTS.md only) if neither.

### type Field

| Value | Behavior |
|-------|----------|
| `instructions` | Compiled into AGENTS.md only. No skill directory. |
| `skill` | Installed as native skill only. No AGENTS.md output. |
| `hybrid` | Both AGENTS.md compilation and skill installation. |
| `prompts` | Commands/prompts only. No instructions or skills. |

### scripts Field

Named commands executed via `apm run <name>`. Supports `--param key=value` substitution.

```yaml
scripts:
  review: "copilot -p 'code-review.prompt.md'"
  impl:   "copilot -p 'implement-feature.prompt.md'"
```

### dependencies.apm

Each element is either a **string** or an **object**.

**String forms:**

```yaml
dependencies:
  apm:
    # GitHub shorthand (default host)
    - microsoft/apm-sample-package              # latest
    - microsoft/apm-sample-package#v1.0.0       # tag
    - microsoft/apm-sample-package#main          # branch

    # Non-GitHub hosts (FQDN preserved)
    - gitlab.com/acme/coding-standards
    - bitbucket.org/team/repo#main

    # Full URLs
    - https://github.com/microsoft/apm-sample-package.git
    - git@github.com:microsoft/apm-sample-package.git

    # Virtual packages (subdirectory or single file)
    - ComposioHQ/awesome-claude-skills/brand-guidelines
    - contoso/prompts/review.prompt.md

    # Local path (development only)
    - ./packages/my-shared-skills
```

**Object form** (when shorthand is ambiguous):

```yaml
- git: https://gitlab.com/acme/repo.git
  path: instructions/security       # subdirectory within repo
  ref: v2.0                         # git reference
  alias: acme-sec                   # local alias
```

**Local path dependency:**

```yaml
- path: ./packages/my-shared-skills
```

**Canonical normalization:** GitHub URLs are stripped to `owner/repo`. Non-GitHub hosts keep
their FQDN. This prevents duplicate installs from different URL formats.

### dependencies.mcp

MCP server dependencies in three forms:

```yaml
dependencies:
  mcp:
    # Registry reference (string)
    - io.github.github/github-mcp-server

    # Registry with overlays (object)
    - name: io.github.github/github-mcp-server
      tools: ["repos", "issues"]
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    # Self-defined server (object, registry: false)
    - name: my-private-server
      registry: false
      transport: stdio                # REQUIRED when registry: false
      command: ./bin/my-server        # REQUIRED for stdio transport
      args: ["--port", "3000"]
      env:
        API_KEY: ${{ secrets.KEY }}
```

Self-defined server validation rules:
- `transport` MUST be present when `registry: false`
- `command` REQUIRED for `stdio` transport
- `url` REQUIRED for `http`, `sse`, `streamable-http` transports

### devDependencies

Same structure as `dependencies`. Installed locally but excluded from `apm pack --format plugin` bundles.

```yaml
devDependencies:
  apm:
    - owner/test-helpers
```

### compilation Config

```yaml
compilation:
  target: all                    # vscode | claude | codex | all
  strategy: distributed          # distributed | single-file
  output: AGENTS.md              # custom output path
  resolve_links: true            # resolve relative markdown links
  source_attribution: true       # include source-file origin comments
  exclude:                       # glob patterns to skip
    - "apm_modules/**"
    - "tmp/**"
  placement:
    min_instructions_per_file: 1
```

## Primitive Types

APM supports 7 primitive types. Each has a specific file format and storage location.

### 1. Instructions (.instructions.md)

Targeted guidance applied by file pattern.

```markdown
---
description: Python coding standards        # REQUIRED
applyTo: "**/*.py"                           # REQUIRED (glob pattern)
---

# Python Standards
- Follow PEP 8
- Use type hints for all function parameters
```

Location: `.apm/instructions/` or `.github/instructions/`

### 2. Prompts (.prompt.md)

Executable AI workflows with parameters.

```markdown
---
description: Run a security audit           # REQUIRED
input: [target_files, severity]              # parameter list
mode: security-expert                        # agent to use
mcp:
  - github-mcp-server                       # required MCP servers
---

# Security Audit
Review ${input:target_files} for vulnerabilities...
```

Location: `.apm/prompts/` or `.github/prompts/`

Parameters use `${input:parameter_name}` syntax.

### 3. Agents (.agent.md)

AI assistant personas with tool boundaries. Legacy format `.chatmode.md` is also supported.

```markdown
---
description: Senior backend developer       # REQUIRED
tools: ["terminal", "file-manager"]
expertise: ["security", "performance"]
---

You are a senior backend engineer...
```

Location: `.apm/agents/` or `.github/agents/`

### 4. Skills (SKILL.md)

Package meta-guides that help AI agents understand what a package does.

```markdown
---
name: Brand Guidelines                      # REQUIRED
description: Apply corporate brand standards # REQUIRED
---

# How to Use
When asked about branding, apply these standards...

## Guidelines
- Rule 1
- Rule 2

## Examples
...
```

Location: Root of package (package-level skill) or `.apm/skills/<name>/SKILL.md` (sub-skills).

**Skill folder naming rules** (agentskills.io spec):
- 1-64 characters
- Lowercase alphanumeric + hyphens only (a-z, 0-9, -)
- No consecutive hyphens (--)
- Cannot start or end with hyphen

**Bundled resources:**

```
my-skill/
├── SKILL.md           # main skill file
├── scripts/           # executable code
├── references/        # documentation
├── examples/          # sample files
└── assets/            # templates, images
```

### 5. Hooks (.json)

Lifecycle event handlers that run scripts during AI agent operations.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "tool_name": "write_file" },
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/lint-changed.sh $TOOL_INPUT_path"
          }
        ]
      }
    ]
  }
}
```

Supported events: `PreToolUse`, `PostToolUse`, `Stop`, `Notification`, `SubagentStop`

Location: `.apm/hooks/` or `hooks/`

### 6. Plugins (plugin.json)

Pre-packaged agent bundles. Only `name` is required.

```json
{
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "What this plugin does"
}
```

Plugin primitives are at the repository root:

```
plugin-repo/
├── plugin.json         # or .github/plugin/, .claude-plugin/, .cursor-plugin/
├── agents/
├── skills/
├── commands/
└── hooks.json
```

APM searches for plugin.json in priority order:
1. `plugin.json` (root)
2. `.github/plugin/plugin.json`
3. `.claude-plugin/plugin.json`
4. `.cursor-plugin/plugin.json`

### 7. MCP Servers

Declared in `dependencies.mcp` in apm.yml. See the dependencies.mcp section above.

## Package Structure Patterns

### Standalone Skill (simplest)

Just a `SKILL.md` at the repo root. APM auto-generates `apm.yml` on install.

```
my-skill/
└── SKILL.md
```

### APM Package with Skill

```
my-package/
├── apm.yml
├── SKILL.md              # package-level skill (not deployed as local skill)
└── .apm/
    ├── instructions/
    ├── prompts/
    └── agents/
```

The root `SKILL.md` describes the package itself -- it is NOT deployed as a local skill.

### Multi-Skill Package

Bundle multiple skills inside `.apm/skills/`. Each sub-skill is promoted to a
top-level entry in `.github/skills/` (or equivalent) on install.

```
my-package/
├── apm.yml
├── SKILL.md              # parent/orchestrator skill
└── .apm/
    ├── instructions/
    ├── prompts/
    └── skills/
        ├── skill-a/
        │   └── SKILL.md
        └── skill-b/
            └── SKILL.md
```

Result after install:
```
.github/skills/
├── my-package/           # parent skill
│   └── SKILL.md
├── skill-a/              # promoted sub-skill
│   └── SKILL.md
└── skill-b/              # promoted sub-skill
    └── SKILL.md
```

### Skills Collection (Monorepo)

Multiple standalone skills in a monorepo. Users install individually.

```
awesome-skills/
├── skill-1/
│   ├── SKILL.md
│   └── references/
├── skill-2/
│   └── SKILL.md
└── skill-3/
    ├── SKILL.md
    └── scripts/
```

Install with: `apm install org/awesome-skills/skill-1`

### Hybrid Plugin

Author with APM deps + security, export as standalone plugin.

```
my-plugin/
├── apm.yml
├── plugin.json
└── .apm/
    ├── agents/
    ├── skills/
    └── prompts/
```

## Target Deployment Mapping

Where primitives land for each AI tool:

| Primitive | Copilot (.github/) | Claude (.claude/) | Cursor (.cursor/) | OpenCode (.opencode/) | Codex |
|-----------|-------------------|-------------------|-------------------|-----------------------|-------|
| Instructions | instructions/*.instructions.md | rules/*.md | rules/*.mdc | Via AGENTS.md | Via AGENTS.md |
| Prompts | prompts/*.prompt.md | commands/*.md | -- | commands/*.md | -- |
| Agents | agents/*.agent.md | agents/*.md | agents/*.md | agents/*.md | .codex/agents/*.toml |
| Skills | skills/{name}/SKILL.md | skills/{name}/SKILL.md | skills/{name}/SKILL.md | skills/{name}/SKILL.md | .agents/skills/{name}/SKILL.md |
| Hooks | hooks/*.json | settings.json (hooks key) | hooks.json | -- | .codex/hooks.json |
| MCP | .vscode/mcp.json | .claude/settings.json | .cursor/mcp.json | opencode.json | ~/.codex/config.toml |

## Virtual Packages

A dependency can target a subdirectory, file, or collection within a repository.

Classification rules (evaluated in order):

| Kind | Detection Rule | Example |
|------|---------------|---------|
| File | Ends in `.prompt.md`, `.instructions.md`, `.agent.md`, `.chatmode.md` | `owner/repo/prompts/review.prompt.md` |
| Collection (dir) | Contains `/collections/` | `owner/repo/collections/security` |
| Subdirectory | Does not match file or collection rules | `owner/repo/skills/security` |

## Complete apm.yml Example

```yaml
name: my-project
version: 1.0.0
description: AI-native web application
author: Contoso
license: MIT
target: all
type: hybrid

scripts:
  review: "copilot -p 'code-review.prompt.md'"

dependencies:
  apm:
    - microsoft/apm-sample-package#v1.0.0
    - gitlab.com/acme/coding-standards#main
    - git: https://gitlab.com/acme/repo.git
      path: instructions/security
      ref: v2.0
  mcp:
    - io.github.github/github-mcp-server
    - name: my-private-server
      registry: false
      transport: stdio
      command: ./bin/my-server
      env:
        API_KEY: ${{ secrets.KEY }}

devDependencies:
  apm:
    - owner/test-helpers

compilation:
  target: all
  strategy: distributed
  exclude:
    - "apm_modules/**"
  placement:
    min_instructions_per_file: 1
```

## Gotchas and Common Mistakes

1. **Root SKILL.md is not a local skill.** The root `SKILL.md` describes the package
   itself. It is NOT deployed as a local skill. Sub-skills in `.apm/skills/` ARE deployed.

2. **GitHub shorthand strips `github.com`.** When writing to apm.yml, APM normalizes
   `https://github.com/owner/repo.git` to just `owner/repo`. Non-GitHub hosts keep their
   FQDN: `gitlab.com/owner/repo`.

3. **Nested GitLab groups + virtual paths are ambiguous in shorthand.** Use the object
   form to be explicit:
   ```yaml
   - git: gitlab.com/group/subgroup/repo
     path: file.prompt.md
   ```

4. **Skills folder naming is strict.** Must be lowercase, hyphens only, 1-64 chars,
   no consecutive hyphens, can't start/end with hyphen.

5. **Self-defined MCP servers from transitive deps are not auto-trusted.** They are
   skipped with a warning. Either re-declare in your own apm.yml or use
   `--trust-transitive-mcp`.

6. **Local path deps can't be packed.** `apm pack` rejects packages with local path
   dependencies. Replace with remote refs before distributing.

7. **type field is behavioral, not enforced at parse time.** Currently driven by
   package content (presence of SKILL.md, component directories). The field is
   reserved for future explicit overrides.

8. **Compilation is optional for most users.** `apm install` deploys all primitives
   natively for Copilot, Claude, and Cursor. Only run `apm compile` when targeting
   Codex, OpenCode (for instructions), or Gemini.
