---
name: apm-distribution
description: >
  How to distribute APM packages -- plugin format, marketplaces, CI/CD pipelines,
  bundle workflows, enterprise governance, multi-tool targeting, and cross-platform
  deployment. Use whenever packaging for distribution, exporting as plugins, setting up
  CI/CD with APM, configuring marketplaces, or deploying to multiple AI tools. Also use
  when the user mentions apm pack, plugin.json, marketplace, bundle, distribution,
  CI integration, enterprise deployment, or cross-platform targeting.
---

# APM Distribution

How to distribute APM packages across teams, CI pipelines, marketplaces, and
multiple AI coding tools.

## Distribution Methods

| Method | Use Case | Requires APM? |
|--------|----------|---------------|
| Git repo | Team sharing, `apm install owner/repo` | Yes (consumer) |
| Bundle (`apm pack`) | CI artifacts, air-gapped envs | No (consumer uses tar) |
| Plugin (`apm pack --format plugin`) | Standalone plugin hosts | No |
| Marketplace | Curated discovery | Yes (for `apm install`) |

## Bundle Workflow

The core pipeline: resolve once, distribute the artifact.

```
apm install  -->  apm pack  -->  upload  -->  download  -->  apm unpack (or tar xzf)
```

The left side needs APM. The right side needs nothing -- just extract at project root.

### Creating Bundles

```bash
# Default: directory output to ./build/<name>-<version>/
apm pack

# Archive for distribution
apm pack --archive

# Filter by target platform
apm pack --target copilot        # only .github/ files
apm pack --target claude         # only .claude/ files
apm pack --target all            # all platforms

# Custom output
apm pack -o dist/

# Preview
apm pack --dry-run
```

### Bundle Contents

The bundle mirrors what `apm install` produces. Extract at project root and
files land exactly where they belong.

```
build/my-project-1.0.0/
  .github/
    prompts/
    agents/
    instructions/
    skills/
  .claude/
    commands/
    agents/
    skills/
  apm.lock.yaml              # enriched copy (not written to consumer project)
```

### Cross-Target Path Mapping

When packing for a target different from install, skills/ and agents/ are
automatically remapped:

```
apm pack --target claude
# .github/skills/my-plugin/SKILL.md  -->  .claude/skills/my-plugin/SKILL.md
# .github/agents/helper.md           -->  .claude/agents/helper.md
```

Only skills/ and agents/ are remapped. Commands, instructions, and hooks are
target-specific and never mapped.

### Consuming Bundles

```bash
# With APM
apm unpack bundle.tar.gz

# Without APM (just tar)
tar xzf bundle.tar.gz -C .

# To a specific directory
apm unpack bundle.tar.gz -o /path/to/project

# Preview
apm unpack bundle.tar.gz --dry-run
```

`apm unpack` is additive-only (never deletes). Bundle files overwrite on conflict.
The bundle's lockfile is NOT copied to the consumer project.

### Target Mismatch Warning

```
$ apm unpack team-skills.tar.gz
[i] Bundle target: claude (1 dep(s), 3 file(s))
[!] Bundle target 'claude' differs from project target 'copilot'
```

Informational only -- files still extract. Choose target at pack time.

## Plugin Export

Transform an APM package into a standalone plugin consumable by Copilot CLI,
Claude Code, or other plugin hosts -- no APM-specific files in output.

```bash
apm pack --format plugin
```

### Output Mapping

| APM Source | Plugin Output |
|---|---|
| `.apm/agents/*.agent.md` | `agents/*.agent.md` |
| `.apm/skills/*/SKILL.md` | `skills/*/SKILL.md` |
| `.apm/prompts/*.prompt.md` | `commands/*.md` (renamed) |
| `.apm/instructions/*.instructions.md` | `instructions/*.instructions.md` |
| `.apm/hooks/*.json` | `hooks.json` (merged single file) |
| `.apm/commands/*.md` | `commands/*.md` |

### Plugin Output Structure

```
build/my-plugin-1.0.0/
  agents/
    architect.agent.md
  skills/
    security-scan/
      SKILL.md
  commands/
    review.md
  instructions/
    coding-standards.instructions.md
  hooks.json
  plugin.json
```

### plugin.json

Generated automatically from apm.yml metadata, or uses existing `plugin.json`
from the project (searched at root, `.github/plugin/`, `.claude-plugin/`,
`.cursor-plugin/`).

Minimal `plugin.json`:
```json
{
  "name": "Plugin Display Name"
}
```

Full plugin.json with custom component paths:
```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "What this plugin does",
  "agents": ["./agents/planner.md", "./agents/coder.md"],
  "skills": ["./skills/analysis", "./skills/review"],
  "commands": "my-commands/",
  "hooks": "hooks.json",
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"]
    }
  }
}
```

### devDependencies Exclusion

Packages in `devDependencies` are excluded from plugin bundles. Use for test
helpers, lint rules, and authoring tools that consumers don't need.

```bash
apm install --dev owner/test-helpers
apm pack --format plugin    # test-helpers excluded
```

### Hybrid Authoring Workflow

Author with full APM tooling, export as standalone plugin:

```bash
apm init my-plugin --plugin       # creates apm.yml + plugin.json
apm install --dev owner/helpers   # dev dependency
apm install owner/core-rules      # production dependency
apm pack --format plugin          # clean plugin output
```

## Marketplaces

Marketplaces are GitHub repositories containing a `marketplace.json` index of plugins.

### Setup

```bash
# Register a marketplace
apm marketplace add acme/plugin-marketplace --name acme-plugins

# Register with custom branch
apm marketplace add acme/plugin-marketplace --name acme-plugins --branch release

# GitHub Enterprise
apm marketplace add acme/plugins --host ghes.corp.example.com
```

### Browse and Search

```bash
apm marketplace list                     # list registered marketplaces
apm marketplace browse acme-plugins      # list all plugins
apm search "code review@acme-plugins"    # search by keyword
```

### Install from Marketplace

```bash
apm install code-review@acme-plugins
```

APM resolves marketplace entries to Git URLs, so marketplace-installed plugins
get full version locking, security scanning, and governance.

### Manage

```bash
apm marketplace update acme-plugins     # refresh cache
apm marketplace update                   # refresh all
apm marketplace remove acme-plugins     # unregister
```

## CI/CD Integration

### GitHub Actions -- Recommended Pattern

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: microsoft/apm-action@v1
      - run: apm install
      - run: apm audit --ci                    # security gate
      - run: apm pack --archive
      - uses: actions/upload-artifact@v4
        with:
          name: apm-bundle
          path: build/*.tar.gz

  test:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: apm-bundle, path: ./bundle }
      - run: tar xzf ./bundle/*.tar.gz -C .
      # No APM needed -- files are in place
```

### apm-action Modes

| Mode | Config | Use |
|------|--------|-----|
| Install (default) | `uses: microsoft/apm-action@v1` | Full install + compile |
| Pack | `with: { pack: true }` | Generate bundle artifact |
| Restore | `with: { bundle: ./path.tar.gz }` | Extract without APM |

### CI Audit Gate

```bash
# Baseline lockfile consistency
apm audit --ci

# With org policy enforcement
apm audit --ci --policy org

# With local policy file
apm audit --ci --policy ./apm-policy.yml

# Full diagnostic (no fail-fast)
apm audit --ci --policy org --no-fail-fast
```

Exit 0 = pass, exit 1 = fail.

### Release Audit Trail

```bash
apm pack --archive -o ./release-artifacts/
gh release upload v1.2.0 ./release-artifacts/*.tar.gz
```

### Dev Containers / Codespaces

```json
{
  "onCreateCommand": "tar xzf .devcontainer/apm-bundle.tar.gz -C ."
}
```

## Multi-Tool Targeting

APM deploys primitives to all detected AI tools simultaneously.

### What `apm install` Deploys (Native)

| AI Tool | Directory | Primitives |
|---------|-----------|------------|
| GitHub Copilot | `.github/` | instructions, prompts, agents, hooks, skills, MCP |
| Claude Code | `.claude/` | rules, commands, agents, skills, hooks, MCP |
| Cursor | `.cursor/` | rules, agents, skills, hooks, MCP |
| OpenCode | `.opencode/` | agents, commands, skills, MCP |
| Codex CLI | `.codex/`, `.agents/` | agents, skills, hooks |

### What `apm compile` Adds (Instructions)

| Target | Output | Consumers |
|--------|--------|-----------|
| `copilot` | AGENTS.md | Copilot, Cursor, OpenCode, Gemini |
| `claude` | CLAUDE.md | Claude Code, Claude Desktop |
| `codex` | AGENTS.md | Codex CLI |
| `all` | Both | Universal |

### When to Compile

- **Skip compile:** Copilot, Claude, Cursor users (native primitives via install)
- **Run compile:** Codex (instructions), OpenCode (instructions), Gemini users

### Force Target

```bash
apm install --target claude          # deploy only to .claude/
apm compile --target all             # compile for everything
apm pack --target copilot            # bundle only .github/ files
```

Or set in apm.yml:
```yaml
target: all    # or vscode, claude, codex
```

## Security

### Hidden Unicode Scanning

Every `apm install` scans source files for hidden Unicode characters before deployment:

- **Critical** (blocks deployment): Tag characters, bidi overrides, variation selectors 17-256
- **Warning**: Zero-width spaces/joiners, bidi marks, invisible operators
- **Info**: Non-breaking spaces, emoji presentation selectors

Use `--force` to override critical blocks.

### MCP Trust Model

| Dep Type | Registry Servers | Self-Defined Servers |
|----------|-----------------|---------------------|
| Direct (depth 1) | Auto-trusted | Auto-trusted |
| Transitive (depth > 1) | Auto-trusted | Skipped (warning) |

To trust transitive self-defined servers: re-declare in your apm.yml (preferred)
or use `--trust-transitive-mcp`.

### Lockfile as Audit Trail

```bash
git log --oneline apm.lock.yaml          # history of dep changes
git diff HEAD~1 -- apm.lock.yaml         # what changed last
git show v4.2.1:apm.lock.yaml            # deps at a specific release
```

## Enterprise Patterns

### Org-Wide Distribution

A platform team maintains the canonical prompt library:

1. Maintain a central APM package repo
2. Monthly: `apm install && apm pack --archive`
3. Publish bundle to internal artifact registry
4. Downstream repos pull during CI or onboarding

### Org Policy Enforcement

```bash
# CI gate with org-level policies
apm audit --ci --policy org

# Local policy file
apm audit --ci --policy ./apm-policy.yml

# Force fresh policy fetch (skip cache)
apm audit --ci --policy org --no-cache
```

### Private Packages

For private repositories:
- **GitHub**: Fine-grained token with Contents:Read + Metadata:Read
- **Other hosts**: SSH keys or git credential helpers
- **Azure DevOps**: `dev.azure.com/org/project/_git/repo`

## Gotchas

1. **Pack requires apm.lock.yaml.** Always run `apm install` before `apm pack`.

2. **Local path deps can't be packed.** Replace `./path` references with remote
   refs before running `apm pack`.

3. **Bundle target is fixed at pack time.** `apm unpack` does NOT remap paths.
   If you need a different target, re-pack from source.

4. **Plugin prompt rename.** `review.prompt.md` becomes `review.md` in
   `commands/` during plugin export.

5. **Self-defined MCP from transitive deps are skipped by default.** Re-declare
   them or use `--trust-transitive-mcp`.

6. **Lockfile is sole writer territory.** Never manually edit `apm.lock.yaml`.
   APM is the only writer.

7. **Global scope restrictions.** No local path deps, no MCP at user scope.
   Use remote refs for `apm install -g`.

8. **Cross-target mapping is selective.** Only `skills/` and `agents/` paths
   are remapped across targets. Instructions, commands, and hooks are
   target-specific.
